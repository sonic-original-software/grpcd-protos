# grpcd - Architectural Overview

## Purpose

`grpcd` provides **method-to-address mapping** for the Cumulus microservices
mesh. Services register their gRPC methods on startup; clients query `grpcd` to
find which addresses serve specific methods.

**What `grpcd` Does:**

- Accept method registrations from services
- Store method → address mappings with TTL
- Return addresses for method lookup queries

**What `grpcd` Does NOT Do:**

- Store proto descriptors (services expose via gRPC reflection)
- Track connections or maintain state
- Emit events or manage subscriptions
- Route traffic (gateway concern)

## Core Architecture

### Stateless by Design

`grpcd` instances are **completely stateless**:

- No connection tracking
- No in-memory state
- No goroutines per connection
- No peer discovery
- No replication logic

All state lives in the storage backend implmentation. This enables:

- Infinite horizontal scaling
- Instant crash recovery
- Any instance serves any request
- Simple deployment

### Layered Architecture

```
┌────────────────────────────────────────┐
│           `grpcd` Service              │
│                                        │
│   ┌──────────────────────────────┐     │
│   │     Service Layer            │     │
│   │  • Register                  │     │
│   │  • Discover                  │     │
│   │  • Deregister                │     │
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

Storage backend handles persistence, TTL expiry, and HA/replication. `grpcd`
only implements business logic.

### Regional Deployment

Each geographic region runs an independent `grpcd` cluster:

```
┌───────────────────────┐         ┌───────────────────────┐
│    us-east-1          │         │    eu-west-1          │
│                       │         │                       │
│  Load Balancer        │         │  Load Balancer        │
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
- DNS routes to regional cluster
- Services register locally, clients discover locally
- Follows Kubernetes regional cluster pattern

## Core Flows

### Service Registration

When a service starts:

1. Service spawns background registration goroutine
2. Goroutine makes Register RPC with list of methods it implements
3. `grpcd` extracts real address from gRPC peer context (TCP connection)
4. `grpcd` validates method names (must be fully qualified)
5. `grpcd` stores each method → address mapping with configurable TTL
6. `grpcd` stores reverse mapping (address → methods) for bulk cleanup
7. RPC completes, connection closes
8. Goroutine sleeps 5 minutes, re-registers to refresh TTL
9. On graceful shutdown, goroutine calls Deregister before exit

**Crash Handling:** If service crashes without deregistering, TTL expires and
storage backend auto-deletes entries.

### Client `grpcd`

When a client needs to call a method:

1. Client checks local cache for method → address mapping
2. On cache miss, client makes Discover RPC with method name
3. `grpcd` validates method name
4. `grpcd` queries storage for method → address
5. `grpcd` returns address or NotFound error
6. RPC completes, connection closes
7. Client creates gRPC connection to discovered address
8. Client queries gRPC reflection endpoint for proto descriptor
9. Client caches (address + connection + descriptor)
10. Client makes actual business call
11. Client maintains health checks; on failure removes from cache and re-queries

**Eventual Consistency:** Brief window where clients may discover dead addresses
(crashed service, TTL not yet expired). Clients detect via health checks and
re-query.

### Service Deregistration

When a service shuts down gracefully:

1. Service context cancelled (SIGTERM)
2. Registration goroutine calls Deregister RPC
3. `grpcd` extracts address from gRPC peer context
4. `grpcd` queries reverse mapping for all methods at that address
5. `grpcd` deletes all method mappings atomically
6. `grpcd` deletes reverse mapping
7. RPC completes, connection closes
8. Service exits

**Idempotent:** Can be called multiple times safely.

## Key Design Decisions

### Peer Context Extraction

`grpcd` extracts service addresses from **gRPC peer context** (TCP connection
metadata), not from request fields.

**Why:**

- Address is cryptographically guaranteed by TCP handshake
- Cannot be spoofed by malicious services
- Services cannot register methods for other addresses
- Services don't need to know their own address
- Prevents method hijacking attacks

**Security Boundary:** Trust is at TCP connection establishment, not application
layer.

### TTL-Based Cleanup

Storage backend automatically expires entries after configurable TTL. Services
re-register periodically to refresh TTL.

**Alternatives Considered:**

- **Connection tracking:** `grpcd` monitors gRPC connections, cleans up on
  disconnect

  - Requires state (connection map per instance)
  - Requires goroutines (per-connection lifecycle management)
  - Makes `grpcd` stateful and complex

- **Heartbeat protocol:** Services send periodic heartbeats, `grpcd` marks as
  dead on timeout
  - Requires state (heartbeat timestamps per instance)
  - Requires goroutines (per-service timeout monitoring)
  - More network overhead

**TTL Wins:**

- Zero state in `grpcd` (storage backend handles expiry)
- Services self-manage lifecycle (re-registration in background)
- Proven pattern (Consul, etcd, DNS)
- Simple failure model

**Tradeoff:** Dead services discoverable for up to TTL duration. Acceptable
given simplicity gains and client health check mitigation.

### Storage Backend Abstraction

`grpcd` delegates all persistence to pluggable storage backend.

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

**Tradeoff:** External dependency (Redis must be available). Acceptable for
operational simplicity.

### gRPC Reflection for Descriptors

Services expose proto descriptors via standard gRPC reflection API. Clients
fetch descriptors directly from services, not from `grpcd`.

**Why `grpcd` Doesn't Store Descriptors:**

- Descriptors are large (100KB+ per service)
- Descriptors are static (compiled into binaries)
- Descriptors only change on service restart
- `grpcd` stays lightweight (small footprint: 78 bytes per method)
- No blob storage complexity

### No Events or Subscriptions

`grpcd` provides pull-based lookup only, not push-based notifications.

**Why No Pub/Sub:**

- Would require `grpcd` to track subscribers (stateful)
- Would require connection lifecycle management
- Would require event fanout logic
- Adds distributed system complexity
- Clients already health-check connections

**Client-Managed Lifecycle:**

- Client caches `grpcd` results with connections
- Client health-checks cached connections
- On health check failure: remove from cache, re-query `grpcd`
- Simpler than distributed event system

### Regional Isolation

Each geographic region runs an independent `grpcd` cluster with its own storage
backend. No cross-region replication or state sync.

**Why Regional, Not Global:**

- Follows Kubernetes pattern (regional clusters, not global)
- etcd manages single cluster, not global state
- Latency: services discover local instances
- Blast radius: regional failure doesn't affect other regions
- Simplicity: no distributed state across continents

**Multi-Region Pattern:**

- DNS/load balancer routes to regional cluster
- Services register in their region
- Clients discover in their region
- Infrastructure handles geographic routing

### Last-Writer-Wins

When multiple services register the same method, last registration overwrites
previous.

**Why:**

- Enables failover (new instance takes over methods)
- Simple conflict resolution (no coordination)
- Storage backend handles atomically (single SET operation)

**Tradeoff:** No load balancing across multiple instances serving same method.
`grpcd` returns single address, not list. Clients can retry with re-query for
basic HA.

## Data Model

`grpcd` stores two mappings in storage backend:

**Forward Mapping (method → address):**

- Key: method name (fully qualified)
- Value: network address
- TTL: configurable (default 10 minutes)
- Purpose: Lookup during Discover

**Reverse Mapping (address → methods):**

- Key: network address
- Value: set of method names
- TTL: same as forward mapping
- Purpose: Bulk delete during Deregister

**Storage Footprint:**

- ~78 bytes per method mapping
- 10,000 services × 5 methods = 50,000 mappings ≈ 4 MB
- Negligible storage, fits in memory

## Validation

Method names must be fully qualified (contain dots) to prevent ambiguity.

**Valid Examples:**

- service.Method
- package.service.Method
- deeply.nested.package.service.Method

**Invalid Examples:**

- Method (not qualified - which service?)
- service..Method (consecutive dots)
- .service.Method (leading dot)
- service.Method. (trailing dot)
- service. Method (whitespace)

**Rationale:**

- Prevents namespace collision (multiple "GetUser" methods)
- Matches gRPC naming conventions
- Enables future namespace-based routing/policies

## Failure Modes

**`grpcd` Instance Crash:**

- No state lost (stateless)
- Other instances continue serving
- Load balancer routes around failed instance
- Recovery: restart instance, immediately operational

**Storage Backend Failure:**

- All `grpcd` instances fail lookups (no state)
- Services continue operating with cached connections
- On cache miss, clients receive errors
- Recovery: restore storage backend, services re-register

**Service Crash (No Deregister):**

- Method mappings remain until TTL expires
- Clients discover stale address
- Client connection fails or health check fails
- Client re-queries `grpcd`
- Window: up to TTL duration

**Network Partition:**

- Regional isolation prevents cross-region impact
- Within region: `grpcd` instances share storage backend
- If `grpcd` can't reach storage: fails open (returns errors)
- Services and clients use cached connections

## Observability

`grpcd` exposes business metrics via OpenTelemetry:

- Total registrations (counter)
- Total deregistrations (counter)
- Total discoveries (counter)

Standard observability endpoints:

- Metadata: service name and version
- Diagnostics: storage connectivity, instance health

Export metrics to Prometheus/Grafana for monitoring.

## Configuration

`grpcd` configured entirely via environment variables (12-factor):

- Storage backend type and address
- TTL for method mappings (hot-read, no restart)
- Server port and version

Services connecting to `grpcd`:

- `grpcd` cluster address (optional, disconnected mode if unset)
- Re-registration interval (must be < TTL/2 for safety margin)
