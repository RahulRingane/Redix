# Redix

A simple Redis-compatible, asynchronous, in-memory key-value store written in Go. Redix speaks the RESP protocol, so it works with existing Redis clients and tools like `redis-cli` and `nc`.

## Getting Started

### Prerequisites

- Go 1.23+

### Run the server

```bash
$ go run main.go
```

By default Redix listens on `0.0.0.0:7379`. Both host and port are configurable via flags:

```bash
$ go run main.go --host 0.0.0.0 --port 7379
```

## Supported Commands

| Command        | Description                                              |
| -------------- | ---------------------------------------------------------- |
| `PING [msg]`   | Health check; replies with `PONG` or echoes `msg`        |
| `SET k v [EX seconds]` | Sets a key to a value, with optional expiry in seconds |
| `GET k`        | Returns the value of a key, or nil if missing/expired     |
| `DEL k [k ...]` | Deletes one or more keys, returns count deleted          |
| `EXPIRE k seconds` | Sets a TTL on an existing key                         |
| `TTL k`        | Returns remaining TTL in seconds (`-1` no TTL, `-2` no key) |
| `INCR k`       | Atomically increments an integer-valued key               |
| `INFO`         | Returns basic keyspace stats                               |
| `BGREWRITEAOF` | Triggers a rewrite of the append-only file                |
| `CLIENT`       | No-op, returns `OK` (for client compatibility)             |
| `LATENCY`      | No-op, returns an empty array (for client compatibility)   |
| `LRU`          | Forces an immediate LRU eviction pass                      |

## Pipelining Example

Commands can be pipelined — sent back-to-back in a single request without waiting for individual replies.

```
PING:       *1\r\n$4\r\nPING\r\n
SET k v:    *3\r\n$3\r\nSET\r\n$1\r\nk\r\n$1\r\nv\r\n
GET k:      *2\r\n$3\r\nGET\r\n$1\r\nk\r\n
```

Send raw commands over a TCP connection:

```bash
$ (printf 'CMD1CMD2CMD3';) | nc localhost 7379
```

Or send fully RESP-encoded pipelined commands:

```bash
$ (printf '*1\r\n$4\r\nPING\r\n*3\r\n$3\r\nSET\r\n$1\r\nk\r\n$1\r\nv\r\n*2\r\n$3\r\nGET\r\n$1\r\nk\r\n';) | nc localhost 7379
```

## Persistence (AOF)

Redix supports basic append-only file persistence. Run `BGREWRITEAOF` to dump all keys to the AOF file (default: `./dice-master.aof`). Expiration and non-string data types are not yet persisted.

## Eviction

When the number of keys exceeds `KeysLimit` (default: 100), Redix evicts a portion of keys (`EvictionRatio`, default: 40%) according to the configured strategy:

- `allkeys-lru` (default) — approximated LRU using a sampled eviction pool
- `allkeys-random` — random eviction
- `simple-first` — evicts the first key encountered

These are configured in `config/main.go`.

## Storm

Storm is a set of utilities for firing bulk requests at Redix, useful for load testing. `storm/set` spins up 5 concurrent workers that continuously issue random `SET` commands against a local server.

```bash
$ go run storm/set/main.go
```

## Monitoring with Prometheus

Redix can be monitored using `redis_exporter` and Prometheus, since it speaks the Redis protocol.

```bash
$ ./redis_exporter -redis.addr redis://localhost:7379
$ ./prometheus --web.enable-admin-api
$ curl -X POST -g 'http://localhost:9090/api/v1/admin/tsdb/delete_series?match[]={instance="localhost:9121"}'
```
