# grpcd protocol

The contract between a grpcd server and the services and clients that talk to
it. Three RPCs, defined in `protos/grpcd/`, and the rules an implementation of
either side honors.

## Installation

```bash
go get github.com/grpcd/protos
```

The generated Go lives at the module root, package `grpcd`.

## Method Names

Every method name on the wire is gRPC wire format: a leading slash, the fully
qualified service name, a slash, and the method name.

Valid:

- `/Service/Method`
- `/package.Service/Method`
- `/deeply.nested.package.Service/Method`

Invalid:

- `package.Service.Method` (no slashes)
- `/package.Service/Method/` (trailing slash)
- `package.Service/Method` (no leading slash)
- `//package.Service/Method` (consecutive slashes)
- `/package.Service/Get Method` (whitespace)

This is the form that appears as the HTTP/2 `:path`, so a client discovering a
method holds the string it will send. The server rejects anything else with
`INVALID_ARGUMENT`.

## Addresses

An address is `host:port`. The server composes it: the IP comes from the peer
of the `Register` connection, the port from the request. Neither party holds
both halves. A containerized service does not know its reachable IP, and the
server sees only the ephemeral source port of the connection. Reading the IP
off the connection also means a service can register only the address it
actually connected from.

A method maps to a set of addresses. Replicas of one service serve the same
methods and each registers its own address, so removing one leaves the others.

## Register

```proto
rpc Register(RegisterRequest) returns (stream RegisterResponse);
```

The request carries the service's name, its method list, and its listening
port. The server writes one row per method, sends one empty `RegisterResponse`,
and holds the stream. The stream then carries nothing.

The registration is the stream. When it ends, by clean shutdown, crash, or
transport failure, the server removes every row the request named. There is no
refresh, no expiry, and no deregistration RPC; a service that crashes and one
that exits cleanly take the same path.

`server_name` names the service, not the instance. Replicas share it; the
address identifies the instance.

A `Register` naming no methods, or a port outside 1–65535, is rejected with
`INVALID_ARGUMENT`.

## Discover

```proto
rpc Discover(stream DiscoverRequest) returns (stream DiscoverResponse);
```

The client's first message is `method_name`. The server answers with one
candidate address, drawn at random from the method's set, so clients
discovering the same method spread across its replicas.

The client reaches the candidate. If it can, it closes the stream. If it
cannot, it sends `dead_address` naming the candidate; the server removes that
address from the method and answers with the next.

When the set is empty the server holds the stream and answers with the next
address registered for the method as it arrives. The client blocks on its
receive rather than asking again.

A client holds the address it took until the transport to it drops. It then
opens a new `Discover`, reports the address dead, and takes the next candidate.

## Watch

```proto
rpc Watch(WatchRequest) returns (stream WatchResponse);
```

The request names a method and the address the client holds for it. The server
holds the stream.

When a new address registers for the method, the server sends it to a share of
the clients holding other addresses: each holder is chosen with probability
`1/N`, `N` being the method's address count after the addition. The rest hear
nothing. The expected share of clients moving is the new replica's fair share,
so a new replica takes its part of existing connections without every client
reconnecting.

A client sent an address probes it. Reachable: it moves, opens a new `Watch`
naming the new address, and closes the old one. Unreachable: it stays and
reports nothing.

A client opens `Watch` before closing the `Discover` that gave it the address,
and opens the new `Watch` before closing the old one, so no registration falls
between the two.

## Notifications

The server pushes to a client only on a stream that client opened and holds: a
`Discover` waiting for a registration, or a `Watch` naming an address it holds.
When the `Watch` stream breaks, the client opens a new one naming the same
address.

## Removal Is Reversible

A client reporting an address dead may be wrong; its own network can be at
fault while the service is healthy. The server treats the open `Register`
stream as proof the service is up and writes the row back. A client with a
persistent local fault drives a remove-and-restore cycle rather than losing the
row for everyone.
