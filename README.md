# @lam/vulture/std-plugin

`@lam/vulture/std-plugin` is the standard plugin collection for
[`@lam/vulture`](https://github.com/thallium-solutions/lam-vulture). It keeps
common server functionality reusable without expanding the transport package's
core API. The package is written for the
[Lammergeier language](https://github.com/thallium-solutions/lammergeier-lang)
and is the standalone continuation of Lammergeier's original
`lamserver_plugins` module.

## Requirements

- `lamc` 1.17.0 or a compatible later 1.x release (`^1.17`).
- Go, used by `lamc` for compression, cryptography, networking, concurrency,
  timekeeping, and runtime callback bridges.
- `@lam/vulture` v1.0.0. A Git dependency pinned to the v1.0.0 tag is declared
  in `lamlib.toml` and is installed transitively.

## Installation

### Registry

Install the v1.0.0 registry release in a Lammergeier project:

```bash
lamc install @lam/vulture/std-plugin@1.0.0
```

Or declare it in the consumer's `lamlib.toml` and install the manifest:

```toml
[dependencies]
"@lam/vulture/std-plugin" = "1.0.0"
```

```bash
lamc install
```

### Git

Install the tagged Git checkout directly:

```bash
lamc install https://github.com/thallium-solutions/lam-vulture-std-plugin.git@v1.0.0
```

A manifest may also pin the Git source explicitly:

```toml
[dependencies]
"@lam/vulture/std-plugin" = { git = "https://github.com/thallium-solutions/lam-vulture-std-plugin.git", ref = "v1.0.0" }
```

The package itself resolves `@lam/vulture` from
`https://github.com/thallium-solutions/lam-vulture.git` at `v1.0.0`.

### Local checkout

For development, install this repository by path:

```bash
git clone https://github.com/thallium-solutions/lam-vulture.git
git clone https://github.com/thallium-solutions/lam-vulture-std-plugin.git
cd your-lammergeier-project
lamc install ../lam-vulture-std-plugin
```

To make both sibling checkouts authoritative while editing them, use project
replacements:

```toml
[dependencies]
"@lam/vulture" = "1.0.0"
"@lam/vulture/std-plugin" = "1.0.0"

[replace]
"@lam/vulture" = { path = "../lam-vulture" }
"@lam/vulture/std-plugin" = { path = "../lam-vulture-std-plugin" }
```

Then run `lamc install`. Whether installed from the registry, Git, or a local
path, application imports stay the same:

```lammergeier
from @lam/vulture import Server, Request, Response
from @lam/vulture/std-plugin import helmet, requestId
```

## Exports

Every public symbol is re-exported by `__init__.lam`.

### Request and response policy

- `requestLog(srv, label="[req]")` writes request completion lines to stderr.
- `requestId(srv, header="X-Request-Id")` preserves or creates a request ID.
- `rateLimit(srv, max=100, windowMs=60000, message="rate limit exceeded")`
  adds an in-memory sliding-window client limiter.
- `compress(srv, minBytes=1024)` applies gzip to eligible buffered and stream
  responses while excluding SSE and hijacked connections.
- `helmet(srv)` adds conservative browser security headers.
- `etag(srv)` creates weak ETags and handles matching `If-None-Match` requests.
- `cacheControl(srv, directive="no-store", ifMissing=true)` selects a default
  cache policy.
- `serverTiming(srv)` emits an RFC 8290 `Server-Timing` total duration.

### Operations and resilience

- `healthcheck(srv, livePath="/healthz", readyPath="/readyz")` registers
  liveness and readiness routes.
- `markReady(srv)` and `markNotReady(srv)` atomically change readiness.
- `metrics(srv, path="/metrics")` exports Prometheus counters, latency buckets,
  and process uptime.
- `tracing(srv, echoHeader=true)` propagates W3C Trace Context.
- `idempotency(srv, header="Idempotency-Key", ttlSec=300)` replays cached
  successful mutation responses.
- `CircuitBreaker(threshold=5, cooldownMs=1000)` guards zero-argument
  downstream calls and reports failures through `Result`.

### Authentication and browser state

- `basicAuth(srv, users, realm="Restricted")` validates HTTP Basic credentials.
- `bearerAuth(srv, verify, header="Authorization")` delegates bearer-token
  verification to an application callback.
- `ipFilter(srv, allow=[], deny=[])` filters exact addresses and CIDR ranges.
- `session(srv, secret, cookieName="sid", maxAge=86400, secure=false,
  sameSite="Lax")` supplies signed, in-memory sessions.
- `csrf(srv, secret, cookieName="csrf", headerName="X-CSRF-Token",
  safeMethods=["GET", "HEAD", "OPTIONS"])` supplies signed double-submit-cookie
  protection.

The rate limiter, metrics, idempotency cache, session store, and readiness gate
are process-local. Use shared infrastructure when state must span processes or
survive restarts.

## Basic server example

```lammergeier
from @lam/vulture import Server, Request, Response
from @lam/vulture/std-plugin import cacheControl, helmet, healthcheck, requestId

func hello(req: Request, res: Response) {
    res.json({"message": "hello", "requestId": req.id()})
}

func main() {
    srv: Server = Server()

    requestId(srv)
    helmet(srv)
    cacheControl(srv, "private, max-age=30")
    healthcheck(srv)

    srv.get("/hello", hello)
    srv.listen(8080)
}
```

Plugins append lifecycle hooks, so registration order matters when two plugins
write the same header or consume each other's context. Register request context
producers such as `requestId` and `tracing` before custom hooks that read them.
Most response defaults are additive and preserve an explicit handler value.

## Authentication example

```lammergeier
from @lam/vulture import Server, Request, Response
from @lam/vulture/std-plugin import bearerAuth, rateLimit

func verifyToken(token: str) -> any {
    if token == "development-token" {
        return {"name": "alice"}
    }
    return None
}

func profile(req: Request, res: Response) {
    res.json(req.ctx["user"])
}

func main() {
    srv: Server = Server()
    rateLimit(srv, max=60, windowMs=60000)
    bearerAuth(srv, verifyToken)
    srv.get("/profile", profile)
    srv.listen(8080)
}
```

Use Basic authentication only over TLS. The bundled session store is intended
for examples and single-process services; production deployments generally
need an external, expiring session backend.

## Circuit breaker example

```lammergeier
from @lam/vulture/std-plugin import CircuitBreaker
from lamerrors import Result

func refreshDownstream() {
    # Call a database, HTTP service, or other dependency here.
}

func main() {
    breaker: CircuitBreaker = CircuitBreaker(threshold=3, cooldownMs=2000)
    result: Result = breaker.callFn(refreshDownstream)
    if not result.ok() {
        print("downstream unavailable: " + str(result.error))
    }
}
```

The breaker starts closed. Consecutive callback failures open it; calls then
fail immediately with `CircuitOpen`. After the cooldown, a successful half-open
probe closes it. Callback panics become `CircuitFailed` Results.

## Testing

Keep this repository beside an `lam-vulture` checkout and run:

```bash
python3 tests/run_tests.py --verbose
# or through the manifest
lamc lib run test
```

The runner requires `lamc >= 1.17.0`, creates an isolated temporary `extlibs`
tree, stages the sibling `lam-vulture` checkout as `@lam/vulture`, nests this
checkout as `@lam/vulture/std-plugin`, builds every migrated consumer-style test
with `--no-cache`, runs it, and compares stdout with its `# expect:` directives.
It does not write generated dependencies or build artifacts into either source
checkout.

## Development boundary

This package is intentionally a collector of common server functionality, not a
second server runtime. Request routing, transport types, response primitives,
and server lifecycle machinery belong to
[`@lam/vulture`](https://github.com/thallium-solutions/lam-vulture). Generic
language and compiler work belongs to
[`lammergeier-lang`](https://github.com/thallium-solutions/lammergeier-lang).
Straightforward policy, collection handling, and Result construction in this
package remain in Lam; raw Go is reserved for compression, cryptography,
network parsing, locks and atomics, time internals, and callback/type-assertion
boundaries.

## License

Licensed under the Apache License, Version 2.0. See [LICENSE](LICENSE).
