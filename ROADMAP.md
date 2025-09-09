# Jetstream — Project Roadmap (v0.x → v1.0)

**Positioning:** a minimal, stdlib-lean, **unified sync+async HTTP client** that feels like `requests` and performs like a modern async library.  
**Guiding principles:** security by default, zero hard dependencies, predictable ergonomics, measured performance.

---

## 1) Scope, Non-Goals & Success Criteria

**In-scope**
- Unified sync/async API with a single mental model.
- HTTP/1.1 first-class; **optional** HTTP/2 with graceful fallback.
- Robust timeouts, retries (with jittered backoff), redirects, cookies.
- Connection pooling, DNS caching, proxy support.
- Streaming requests/responses, multipart/form-data, JSON helpers.
- Fine-grained control (hooks/middleware), observability points (logging, tracing).
- Minimal API surface; clear migration path from `requests`.

**Non-goals (pre-1.0)**
- HTTP/3/QUIC (considered **post-1.0 experimental**).
- A built-in WebSocket client.
- High-level OAuth flows beyond simple token helpers.
- Exotic auth (NTLM, Kerberos), SOCKS proxies (may come via optional extras).
- Any hard dependency (HTTP/2, Brotli, SOCKS behind extras only).

**v1.0 Success**
- API stability: no breaking changes required for common `get/post` + session flows for 12 months.
- Performance: within ±5% of `httpx` for HTTP/1.1 throughput; within ±10% on large streaming bodies.
- Security: TLS verification on by default, secure redirects, header sanitation; zero critical advisories.
- Adoption: ≥5 public dependents, ≥1 framework integration example (FastAPI/Django) in docs.

---

## 2) Technical Strategy & Architecture

**Core model**
- **Engine:** async-first implementation; sync API is a thin wrapper that drives the engine with a private runner.
- **Session (`Client`)**: manages pools, cookies, proxies, default headers, timeouts, retries, and feature flags (e.g., `http2=True`).
- **Transport layer:** pluggable transports (`HTTP11Transport`, `HTTP2Transport?`) behind a common interface.
- **Pool:** connection pool keyed by (scheme, host, port, proxy); LIFO reuse, TTL, max per host & total; keep-alive.
- **Timeouts:** structured `Timeout(connect, read, write, pool)` with global defaults and per-request overrides.
- **Retries:** idempotent-by-default (GET/HEAD/OPTIONS/TRACE) + user override; exponential backoff with jitter; retry budget.
- **Redirects:** RFC-compliant, with policy knobs (max redirects, allow downgrade, preserve method on 308/307).
- **Cookies:** RFC6265 cookie jar with persistence hooks.
- **Proxies:** HTTP/HTTPS + env var discovery; SOCKS via optional extra.
- **Compression:** `gzip`/`deflate` by default; **Brotli via optional extra**.
- **HTTP/2:** **optional extra** (`jetstream[http2]`), using a pure-Python H2 implementation; ALPN negotiation; graceful fallback to 1.1.
- **Observability:** structured events (request start/finish, DNS resolve, connect, TLS, send/recv chunk, redirect, retry) surfaced via hooks; OpenTelemetry shims (**optional**).
- **Typing:** precise type hints; `mypy` + pyright green; stable public types (`Response`, `Request`, `Client`, `Timeout`, `Retry`).

**Sync wrapper**
- Private loop runner avoiding conflicts in existing event loops (detect & spawn a dedicated loop/thread when needed).
- `Client` supports both `with Client():` and `async with Client():`.

**Security defaults**
- `ssl.create_default_context()` with OS trust store; hostname verification on; minimum TLS version 1.2; OCSP stapling if present; robust redirect policy; header normalization to avoid request smuggling patterns.

---

## 3) Release Train & Timeline (targeting EU/Paris time)

> Dates are **targets** to sequence work; ship when quality gates pass.

### Phase A — Alpha Foundations (Sep–Nov 2025)
**v0.1.0 (Alpha-1)** — *Core HTTP/1.1 GET/POST*  
- Minimal `get/post` one-shots + `Client` session  
- Request/Response types (`.status`, `.headers`, `.content`, `.text()`, `.json()`)  
- Basic timeouts (connect/read) + SSL verification (on)  
- CI (Linux/macOS/Windows), Python 3.10–3.13, PyPy3; ruff, mypy, coverage ≥90%  
**Exit:** Download a page over HTTPS with cert verification and decode JSON reliably.

**v0.2.0 (Alpha-2)** — *Pooling, Redirects, Cookies, Proxies*  
- Connection pooling + keep-alive (per-host limits, total cap, TTL)  
- Redirect policy with counters; cookie jar (session + per-request)  
- HTTP/HTTPS proxy support; env var discovery (`HTTP_PROXY`, `NO_PROXY`)  
- Structured `Timeout(connect, read, write, pool)`  
**Exit:** Stable soak tests with long-running pool reuse; no fd leaks.

**v0.3.0 (Alpha-3)** — *Retries & Streaming*  
- Retries with backoff + jitter; idempotency detection; hooks to override  
- Streaming request/response (`iter_bytes()`, `iter_lines()`, file-like bodies)  
- Multipart/form-data; chunked uploads; JSON helpers (encoder override)  
**Exit:** Upload & download large bodies without excessive memory; retry telemetry visible.

### Phase B — Beta Capabilities (Dec 2025–Feb 2026)
**v0.4.0 (Beta-1)** — *HTTP/2 Optional Extra*  
- `jetstream[http2]` extra; ALPN negotiation; max concurrent streams; back-pressure  
- Multiplexing with per-stream flow control; fallback to 1.1 when needed  
**Exit:** Multiplexed downloads benchmark; fallback seamless where ALPN lacks H2.

**v0.5.0 (Beta-2)** — *Hooks/Middleware & Observability*  
- Request/response lifecycle hooks (pre-send, post-receive, retry decision)  
- Metrics counters/timers (pluggable), OpenTelemetry shim package (**optional**)  
- Structured debug logging with redaction of sensitive headers  
**Exit:** Tracing example in docs (e.g., OTLP exporter) with measurable spans.

**v0.6.0 (Beta-3)** — *Auth, Encoding & Error Model*  
- Helpers: Basic, Bearer token, Digest (best-effort)  
- Brotli support via `jetstream[brotli]` extra  
- Unified exception hierarchy with retryable/non-retryable flags; rich error messages  
**Exit:** Clear, documented error taxonomy; helpful reprs & `__str__`.

### Phase C — RC & Hardening (Mar–Apr 2026)
**v0.7.0 (RC-1)** — *API Freeze Candidate*  
- Finalize public API names & parameters; deprecation shims if needed  
- Documentation overhaul: “Requests → Jetstream in 15 minutes”, cookbook, recipes  
- Migration lints (optional script to flag common `requests` patterns)  
**Exit:** API review sign-off; no known breaking changes planned.

**v0.8.0 (RC-2)** — *Performance & Soak*  
- Bench harness (local server + `pytest-benchmark`/`locust` dev-only)  
- Perf targets validated across OS/Python versions; memory profiling  
- Fuzz tests (URLs, headers, chunk boundaries, malformed responses)  
**Exit:** Perf targets hit; fuzz suite green ≥24h soak.

### Phase D — v1.0 GA (May 2026)
**v1.0.0** — *Stable Release*  
- Final docs site & API reference; versioned docs  
- Long-term support matrix announcement; deprecation policy  
- Blog/announcement & examples repo

---

## 4) Quality Gates (apply to each milestone)

- **Tests:** unit (≥90% cov), integration (TLS, redirects, proxies, streaming), property tests for parsers, platform tests.
- **Security:** TLS hardening checks; header canonicalization; redirect/scheme downgrade guards; secrets redaction in logs.
- **Performance:** regression thresholds defined per benchmark (RPS, p50/p95 latency, memory/conn churn).
- **API ergonomics:** doctests in all public entry points; copy-paste-able examples.
- **Docs:** “How to:” for each new capability; release notes with upgrade steps.

---

## 5) Public API (stabilization candidates)

~~~python
from jetstream import Client, get, post, request
from jetstream.types import Timeout, Retry
from jetstream.errors import HTTPError, ConnectError, TimeoutError, TLSError

# One-shot
r = get("https://api.example.com/users", params={"q": "ana"}, retry=Retry(total=3))
data = r.json()

# Session (sync)
with Client(http2=True, timeout=Timeout(connect=2, read=5), retries=Retry(total=2)) as c:
    r = c.post("/login", json={"u": "x", "p": "y"})
    r = c.get("/me", headers={"X-Req": "1"})

# Session (async)
async with Client(http2=True) as c:
    r = await c.get("https://example.com/stream")
    async for chunk in r.iter_bytes():
        ...

# Hooks (beta-2)
def on_request(ctx): ...
def on_response(ctx): ...
with Client(hooks={"request": [on_request], "response": [on_response]}) as c:
    ...
~~~

**Stability policy:** Names above are candidates for pre-1.0 stabilization. Anything outside them remains experimental until RC.

---

## 6) Compatibility, Dependencies & Extras

- **Python:** 3.10–3.13, PyPy3 (supported); 3.9 best-effort (no CI), may be dropped before 1.0.
- **OS:** Linux, macOS, Windows; musllinux wheels where possible.
- **Zero hard deps:** base install relies only on stdlib.
- **Extras:**  
  - `jetstream[http2]` → enables HTTP/2 (pure-Python H2).  
  - `jetstream[brotli]` → Brotli compression.  
  - `jetstream[socks]` → SOCKS proxy support.  
  - `jetstream[otel]` → tracing exporters/shims.

---

## 7) Security & Privacy

- TLS verification on; system CA trust; opt-in for custom CA bundles.
- Minimum TLS 1.2; ciphers per OpenSSL defaults; SNI required.
- Safe redirect policy (no POST → GET without explicit allow when status code requires it; never auto-redirect from HTTPS→HTTP).
- Sensitive headers (Authorization, Cookie) not forwarded across host changes by default.
- Request smuggling protections via header normalization; strict parser for status line and headers.
- Security audit checklist before GA; coordinate CVE process; `SECURITY.md` and responsible disclosure.

---

## 8) Observability & Tooling

- **Logging:** structured events with correlation IDs; redaction of secrets.
- **Tracing:** optional OpenTelemetry spans (DNS resolution, connect, TLS, request send, response receive, retry).
- **Metrics:** counters for requests, retries, redirects; histograms for connect/read/write latencies; pool stats.

---

## 9) Documentation Plan

- “Start here” page (5-minute quickstart).
- “Requests → Jetstream” migration guide with side-by-side examples.
- Cookbook: retries correctly; streaming uploads; large downloads; proxies; custom SSL; tracing.
- Design docs: timeouts model, retries policy, transport abstraction.
- Versioned API reference; typed signatures included.
- Examples repo with runnable scripts; perf harness instructions.

---

## 10) Governance, Releases & Community

- **SemVer:** breaking only at MAJOR; pre-1.0 can break but minimized after RC.
- **Release process:** `release-please`/GitHub Actions; signed wheels/sdist; changelog & upgrade notes.
- **Issue triage:** labels (`good first issue`, `help wanted`, `perf`, `security`, `breaking-change`) and weekly triage rotation.
- **ADRs:** record architecture decisions (`/docs/adrs`).
- **Code of conduct** and contribution guide.
- **Roadmap transparency:** milestone boards per phase.

---

## 11) Risk Register & Mitigations

- **HTTP/2 complexity** → ship as optional extra; exhaustive tests; graceful fallback.
- **Event loop edge cases** (sync wrapper inside async apps) → detect running loop and delegate to background runner; document caveats.
- **Performance regressions** → CI perf checks on PRs (smoke), weekly full runs.
- **TLS/platform quirks** → matrix CI; explicit integration tests per platform; use OS trust by default.

---

## 12) Concrete Backlog (first 8–12 weeks)

**Alpha-1**
- [ ] `Request`/`Response` types (+ streaming API shape).
- [ ] `Client` skeleton; one-shot helpers; SSL on by default.
- [ ] Basic `Timeout`; JSON/text helpers; content decoding (gzip/deflate).
- [ ] CI + ruff + mypy + coverage; docs quickstart.

**Alpha-2**
- [ ] Pool manager (per-host/total limits, TTL, keep-alive).
- [ ] Redirect policy & tests; cookie jar; proxies & env discovery.
- [ ] Full `Timeout` structure and per-request overrides.

**Alpha-3**
- [ ] `Retry` policy (budget, jittered backoff, idempotency rules).
- [ ] Streaming request/response; multipart; file uploads.
- [ ] Error hierarchy with retryable classification.

---

## 13) Exit Checklist for v1.0

- [ ] Public API review complete; deprecated aliases documented.
- [ ] Security audit pass; dependency extras vetted; license compliance.
- [ ] Perf targets validated on all supported Pythons/OSes.
- [ ] Docs complete: migration, cookbook, API reference, versioned site.
- [ ] Backward-compat tests: upgrade from last RC without changes.
