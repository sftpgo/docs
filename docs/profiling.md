---
description: "Profile SFTPGo performance with the built-in Go pprof endpoint. Collect CPU profiles, heap snapshots, goroutine dumps, and execution traces."
---

# Profiling

SFTPGo exposes the standard Go runtime profiler — the same [`net/http/pprof`](https://pkg.go.dev/net/http/pprof){:target="_blank"} endpoints available in any Go program — with no SFTPGo-specific customization. Use it to collect CPU profiles, execution traces, allocation samples, and heap snapshots for bottleneck and memory-leak analysis.

The profiler is served by the telemetry server over HTTP/HTTPS under `/debug/pprof/`, in the format expected by the [pprof](https://github.com/google/pprof/blob/main/doc/README.md){:target="_blank"} visualization tool.

## Enabling the profiler

Set `enable_profiler` to `true` in the `telemetry` section of the configuration file (or via the equivalent environment variable) and configure `bind_port` so the telemetry server listens. See [Telemetry](config-file.md#telemetry) for the full list of options, including TLS and basic-auth settings.

The pprof endpoints expose internal runtime state and can reveal traffic patterns, so they **must not** be reachable from untrusted networks. Keep the telemetry server bound to a private address (`bind_address`) and, in shared environments, require HTTP basic authentication by pointing `auth_user_file` to an htpasswd-compatible file.

## Available profiles

The profiler index is at `/debug/pprof/`. Each profile is fetched with a plain HTTP GET:

- `allocs` — sampling of all past memory allocations
- `block` — stack traces that led to blocking on synchronization primitives
- `goroutine` — stack traces of all current goroutines
- `heap` — sampling of memory allocations of live objects. Add the `gc` query parameter to run GC before sampling
- `mutex` — stack traces of holders of contended mutexes
- `profile` — CPU profile. Add `seconds=N` to control the sampling window; analyze with `go tool pprof`
- `threadcreate` — stack traces that led to the creation of new OS threads
- `trace` — execution trace of the running program. Add `seconds=N`; analyze with `go tool trace`

## Examples

Collect a 30-second CPU profile and open it in pprof's interactive terminal:

```bash
go tool pprof http://127.0.0.1:10000/debug/pprof/profile?seconds=30
```

Snapshot live-heap allocations after a forced GC:

```bash
curl -o heap.pprof http://127.0.0.1:10000/debug/pprof/heap?gc=1
go tool pprof heap.pprof
```

Capture a 10-second execution trace for goroutine-level analysis:

```bash
curl -o trace.out http://127.0.0.1:10000/debug/pprof/trace?seconds=10
go tool trace trace.out
```

If the telemetry server requires basic authentication, pass `--user name:password` to `curl` or let `go tool pprof` prompt you.
