# grpcd - Architectural Overview

## Purpose

`grpcd` provides **method-to-address mapping** for a microservice mesh. Services
hold a registration stream naming the methods they implement; clients query
`grpcd` to find which addresses serve specific methods.

**What `grpcd` Does:**

- Accept registrations held open on a stream
- Store method → addresses, removing an address when its stream ends
- Answer lookups one candidate at a time
- Remove an address a client reports it cannot reach

**What `grpcd` Does NOT Do:**

- Store proto descriptors (services expose via gRPC reflection)
- Health-check services
- Push notifications to clients
- Route traffic (gateway concern)

## Core Architecture

### State lives in the storage backend

Every fact `grpcd` serves is in the storage backend, so any instance answers any
lookup. An instance additionally holds the registration streams it accepted,
which is what ties a row's lifetime to its service's.

This enables:

- Horizontal scaling — instances are interchangeable for lookups
- Crash recovery — a replacement instance serves immediately
- Simple deployment

### Layered Architecture

```
┌────────────────────────────────────────┐
│           `grpcd` Service              │
│                                        │
│   ┌──────────────────────────────┐     │
│   │     Service Layer            │     │
│   │  • Register (held stream)    │     │
│   │  • Discover (bidirectional)  │     │
│   │  • Validation                │     │
│   └──────────┬───────────────────┘     │
│              │                         │
│   ┌──────────▼───────────────────┐     │
│   │   Storage Interface          │     │
│   │  (abstract backend)          │     │
│   └──────────┬───────────────────┘     │
│              │                         │
└──────────────┼─────────────────────────┘
               │
     ┌─────────┴─────────┐
     │                   │
 ┌───▼────┐      ┌───────▼────┐
 │ Redis  │      │    Mock    │
 │Backend │      │  (Testing) │
 └────────┘      └────────────┘
```

Storage backend handles persistence and HA/replication. `grpcd` only implements
business logic.

### Regional Deployment

Each geographic region runs an independent `grpcd` cluster:

```
┌───────────────────────┐         ┌───────────────────────┐
│    us-east-1          │         │    eu-west-1          │
│                       │         │                       │
│  regional DNS name    │         │  regional DNS name    │
│        │              │         │        │              │
│  ┌─────┴──────┬────┐  │         │  ┌─────┴──────┬────┐  │
│  ▼            ▼    ▼  │         │  ▼            ▼    ▼  │
│ D-1          D-2  ... │         │ D-1          D-2  ... │
│  │            │    │  │         │  │            │    │  │
│  └─────┬──────┘    │  │         │  └─────┬──────┘    │  │
│        ▼           │  │         │        ▼           │  │
│   ┌──────────┐     │  │         │   ┌──────────┐     │  │
│   │  Redis   │     │  │         │   │  Redis   │     │  │
│   └──────────┘     │  │         │   └──────────┘     │  │
└───────────────────────┘         └───────────────────────┘
```

- No cross-region state synchronization
- `GRPCD_ADDRESS` names the regional cluster; which instance a service or client
  reaches is whatever that name resolves to
- Services register locally, clients discover locally

## Core Flows

### Service Registration

When a service starts:

1. Service binds its listener and reads the port from it
2. Service opens the `Register` stream, sending its name, its methods, and that
   port
3. `grpcd` extracts the IP from the gRPC peer context (TCP connection) and
   composes the address
4. `grpcd` validates method names (must be in gRPC wire format)
5. `grpcd` adds the address to each method's set and stores the anchor: the id
   of the instance holding this stream
6. `grpcd` sends `RegisterResponse` and holds the stream open
7. When the stream ends — clean shutdown, crash, or transport failure — `grpcd`
   removes that address from every method the request named. The handler holds
   that list for the life of the stream, so no reverse mapping is stored

A registration lives exactly as long as its stream. There is no interval to
refresh and no deregistration call.

### Client Discovery

When a client needs to call a method:

1. Client opens a `Discover` stream and sends the method name
2. `grpcd` validates the method name
3. `grpcd` answers with one candidate address drawn at random from that
   method's set, so clients discovering the same method spread across its
   replicas rather than all taking the same one
4. Client reaches that address and closes the stream
5. If the client cannot reach the candidate it sends `dead_address`; `grpcd`
   removes that address from the method being discovered and answers with the
   next candidate. Other methods that address serves are removed the same way,
   by a client failing on them
6. With no candidates left, `grpcd` holds the stream open and offers the next
   address registered for that method as it arrives. The client blocks on its
   receive rather than asking again, and is woken by the registration

The client watches the connection it took. When that connection breaks it opens
a new `Discover` stream, reports the address dead, and takes the next candidate.

### Reverting a Wrong Removal

A client reporting an address dead may be wrong — its own network can be at
fault while the service is healthy.

The instance anchoring an address is notified when that row is removed, and
writes it back. Its open `Register` stream is live proof the service is up, so
it performs no check of its own.

A client with a persistent local fault drives a remove-and-restore cycle rather
than losing the row. `grpcd` counts reverted removals so that surfaces in
monitoring.

### Losing the Storage Backend

An instance that cannot reach the storage backend can neither record a
registration nor answer a lookup, so it drops the registration streams it holds.
Those services reconnect to a healthy instance, which records their rows under
its own anchor.

## Key Design Decisions

### Peer Context Extraction

`grpcd` extracts the service's IP from **gRPC peer context** (TCP connection
metadata), not from request fields.

**Why:**

- Address is guaranteed by the TCP handshake
- Cannot be spoofed by malicious services
- Services cannot register methods for other addresses
- Prevents method hijacking attacks

The port comes from the service, read from its listener so a bind to `:0`
reports what it actually received. Neither party holds both halves: a
containerized service does not know its reachable IP, and `grpcd` sees only an
ephemeral source port.

**Security Boundary:** Trust is at TCP connection establishment, not application
layer.

### Connection-Anchored Rows

A row exists while its registration stream is held, and the stream ending
removes it.

**Why:**

- A dead service stops being discoverable when it dies, rather than after an
  expiry window
- No clock in the data, no refresh loop in every service, no deregistration RPC
- The evidence is the connection, which `grpcd` already has

**Tradeoff:** An instance holds one stream per registered service replica, so
its memory grows with the mesh and instances are added to spread that. An
instance crashing drops the streams it held, and those services reconnect
elsewhere.

The one case the rule does not cover is a service and its anchoring instance
dying together, leaving a row nothing will remove. That row is removed by the
first client that fails against it.

### Many Addresses per Method

A method maps to a set of addresses. Replicas of one service all serve the same
methods and each registers its own address, so removal takes one address out of
the set and leaves the others.

### Storage Backend Abstraction

`grpcd` delegates all persistence to a pluggable storage backend.

**Alternatives Considered:**

- **Embedded storage (bbolt):**
  - Requires Raft consensus for HA
  - Requires complex distributed state management
  - Reinvents what Redis already does

- **In-memory with peer sync:**
  - Requires gossip protocol between instances
  - Requires distributed state reconciliation
  - Race conditions, split-brain scenarios

- **Direct Redis dependency:**
  - Tight coupling to Redis specifics
  - Hard to test (no mock)
  - No flexibility for other backends

**Backend Abstraction Wins:**

- `grpcd` stays simple (just business logic)
- Default backend is battle-tested (Redis/Valkey)
- Clear separation of concerns (business vs persistence)
- Testable (mock backend for unit tests)
- Infrastructure team owns storage HA, not `grpcd` developers

**Tradeoff:** External dependency. Its availability is the backend's concern.

### gRPC Reflection for Descriptors

Services expose proto descriptors via standard gRPC reflection API. Clients
fetch descriptors directly from services, not from `grpcd`.

**Why `grpcd` Doesn't Store Descriptors:**

- Descriptors are large relative to an address
- Descriptors are static (compiled into binaries)
- Descriptors only change on service restart
- `grpcd` stays lightweight
- No blob storage complexity

### No Notifications to Clients

`grpcd` answers lookups and pushes nothing to clients. A Discover that is
waiting for a registration is still a lookup the client opened and is holding
for an answer — nothing reaches a client that did not ask.

**Why:**

- Would require `grpcd` to track subscribers
- Would require event fanout logic
- A client holding a connection can watch that connection itself

**Client-Managed Lifecycle:** the client can hold the connection it discovered
and watch it. When it breaks, the client discovers again and reports the address
dead. What else it keeps alongside that connection is its own business.

Instances do notify each other about removals, over the storage backend's
publish/subscribe, addressed to a single anchor.

### Regional Isolation

Each geographic region runs an independent `grpcd` cluster with its own storage
backend. No cross-region replication or state sync.

**Why Regional, Not Global:**

- Latency: services discover local instances
- Blast radius: regional failure doesn't affect other regions
- Simplicity: no distributed state across continents

**Multi-Region Pattern:**

- A regional DNS name resolves to that region's instances
- Services register in their region
- Clients discover in their region
- Infrastructure handles geographic routing

## Data Model

`grpcd` stores two mappings in the storage backend:

**Forward Mapping (method → addresses):**

- Key: method name (fully qualified)
- Value: set of network addresses
- Purpose: lookup during Discover

**Anchor (address → instance id):**

- Key: network address
- Value: id of the instance holding that address's `Register` stream
- Purpose: addressing the notification when a row is removed

No entry carries an expiry.

**Anchor ids:** an instance generates a UUIDv4 at startup and uses it as its
anchor id and as its notification channel. The id names a channel that lives and
dies with the process and is never referenced afterward, so it needs no
coordination and no durability.

**Storage Footprint:** a method name, an address, and an instance id per
registered method.

## Validation

Method names are gRPC wire format: a leading slash, the fully qualified service
name, a slash, and the method name.

**Valid Examples:**

- `/Service/Method`
- `/package.Service/Method`
- `/deeply.nested.package.Service/Method`

**Invalid Examples:**

- `package.Service.Method` (no slashes)
- `/package.Service/Method/` (trailing slash)
- `package.Service/Method` (no leading slash)
- `//package.Service/Method` (consecutive slashes)
- `/package.Service/Get Method` (whitespace)

**Rationale:**

- This is the form that appears on the wire as the HTTP/2 `:path`, so a client
  discovering a method has the string it will actually send
- Prevents namespace collision (multiple "GetUser" methods)
- Enables future namespace-based routing/policies

## Failure Modes

**Service Crash:**

- Its `Register` stream ends and `grpcd` removes its address immediately
- Clients holding a connection to it see that connection break and rediscover

**`grpcd` Instance Crash:**

- Its rows remain in the storage backend, because removal happens when an
  instance observes a stream ending and this one is gone. Live services stay
  discoverable throughout
- Its registration streams break. Each of those services reconnects and
  re-registers, adding its address to the same method sets — the same rows,
  written again — and overwriting the anchor with the new instance's id
- Until that re-registration lands, those rows carry the id of a channel nobody
  reads. A removal in that window is not reverted, so a wrong `dead_address`
  report can drop a live service until it re-registers
- Other instances keep answering lookups, since every fact they serve is in the
  storage backend

**Service and Its Anchoring Instance Crash Together:**

- Nothing runs the removal, so the row remains
- The first client to discover that address fails against it and reports it,
  which removes it

**Storage Backend Failure:**

- Instances drop their registration streams and stop serving
- Services and clients continue on the connections they already hold
- Recovery: restore storage backend, services reconnect and re-register

**Network Partition:**

- Regional isolation prevents cross-region impact
- Within region: an instance cut off from storage behaves as above

## Observability

`grpcd` exposes business metrics via OpenTelemetry:

- Total registrations (counter)
- Total removals (counter)
- Total discoveries (counter)
- Reverted removals (counter)

Standard observability endpoints:

- Metadata: service name and version
- Diagnostics: storage connectivity, instance health

Export metrics to Prometheus/Grafana for monitoring.

## Configuration

`grpcd` configured entirely via environment variables (12-factor):

- Storage backend type and address
- Server port and version

Services connecting to `grpcd`:

- `grpcd` cluster address (optional, disconnected mode if unset)
