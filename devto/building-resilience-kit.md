---
title: "Building resilience-kit: A Python Resilience Kernel Forged in Production"
published: true
description: "Designing a framework-agnostic, async-first resilience kernel for Python — five design axioms, a dogfooding release gate (ship iff both boilerplates score ≥8/10), and the bugs that earned the rules."
tags: python, architecture, fastapi, django
canonical_url: https://dev.to/prajwalmahajan101/building-resilience-kit

# dev.to metadata
# article_id: 3872139
# published_at: 2026-06-11T08:18:08Z
# url: https://dev.to/prajwalmahajan101/building-resilience-kit-a-python-resilience-kernel-forged-in-production-5973
---

## TL;DR

**`resilience-kit` is a framework-agnostic, async-first resilience kernel for Python.** Circuit breakers, retries with jitter, token-bucket throttles, SSRF guards with DNS pinning, fire-and-forget audit logs, field-level encryption. Pluggable backends (memory, Redis/Valkey, `pybreaker`). Thin adapters for FastAPI and Django that wire the primitives into each framework's lifecycle and contain zero business logic. One core, 11 ADRs, two adapters.

The interesting part isn't the primitives — most of them are well-trodden ground. The interesting part is **the release rule**: v0.1.0 only shipped because I had to upgrade two unrelated starter repos to it and score **≥ 8/10** on each migration. The reports got filed. FastAPI scored 8/10. Django scored 9/10. The gate held. Anything below 8 would have blocked the cut.

That rule — *"the kit ships iff the kit survives its own migration"* — is what shaped the public surface. Every helper in v0.1.0 (`bind_to`, `from_exception`, `legacy_env_alias`, `verify_envelope_contract`) exists because the first dogfooding round flagged its absence as a blocker. Every doc gap got patched because the migration reports cited the line number it should have been on. This post is how the kit got built, the five axioms underneath it, the gate that gated it, and the bugs it found in itself along the way.

95 source modules, 52 test modules across unit/contract/integration suites, 11 ADRs.

---

## Five Axioms

I didn't draft these up front. I extracted them after the fact by reading the ADRs and asking *what did I refuse to compromise on?* They're the load-bearing constraints; everything else follows.

### 1. One package, many extras. Not a forest of micro-packages.

`pip install resilience-kit` gives you the core. `pip install resilience-kit[redis,pybreaker,fastapi]` adds the backends and adapter. **No micro-libraries until the surface justifies it.** One repo, one CHANGELOG, one tag. Modularity is enforced by import discipline (`import-linter` with layered contracts) and by entry-point provider discovery, not by package boundaries.

![Layered package: your app → adapters → core; extras implement the core Protocols via entry-points](https://mermaid.ink/img/Zmxvd2NoYXJ0IFRCCiAgICBzdWJncmFwaCBDWyJZb3VyIGFwcGxpY2F0aW9uIl0KICAgICAgICBBUFBbRmFzdEFQSSAvIERqYW5nbyBzZXJ2aWNlXQogICAgZW5kCiAgICBzdWJncmFwaCBBWyJyZXNpbGllbmNlX2tpdC5hZGFwdGVycy4qIl0KICAgICAgICBGQVthZGFwdGVycy5mYXN0YXBpXQogICAgICAgIERKW2FkYXB0ZXJzLmRqYW5nb10KICAgIGVuZAogICAgc3ViZ3JhcGggRVsiT3B0aW9uYWwgZXh0cmFzPGJyLz4oZGlzY292ZXJlZCB2aWEgZW50cnktcG9pbnRzKSJdCiAgICAgICAgUkVEW3JlZGlzIGJhY2tlbmRdCiAgICAgICAgUEJbcHlicmVha2VyIGJhY2tlbmRdCiAgICAgICAgUEdbcG9zdGdyZXMgYXVkaXRdCiAgICBlbmQKICAgIHN1YmdyYXBoIEtbInJlc2lsaWVuY2Vfa2l0IGNvcmUiXQogICAgICAgIFBST1RPW1Byb3RvY29sczxici8+Y2FjaGUgwrcgYnJlYWtlciDCtyB0aHJvdHRsZSDCtyBhdWRpdF0KICAgICAgICBQUklNW1ByaW1pdGl2ZXM8YnIvPnJldHJ5IMK3IHJlZ2lzdHJ5IMK3IHJlY292ZXJ5IMK3IGh0dHBfY2xpZW50XQogICAgICAgIFBSSU0gLS0+IFBST1RPCiAgICBlbmQKICAgIEFQUCAtLT4gRkEKICAgIEFQUCAtLT4gREoKICAgIEZBIC0tPiBQUklNCiAgICBESiAtLT4gUFJJTQogICAgUkVEIC0uLT58aW1wbGVtZW50c3wgUFJPVE8KICAgIFBCIC0uLT58aW1wbGVtZW50c3wgUFJPVE8KICAgIFBHIC0uLT58aW1wbGVtZW50c3wgUFJPVE8KICAgIHN0eWxlIEsgZmlsbDojZGJlYWZlLHN0cm9rZTojMWQ0ZWQ4CiAgICBzdHlsZSBBIGZpbGw6I2ZlZjNjNyxzdHJva2U6I2Q5NzcwNgogICAgc3R5bGUgRSBmaWxsOiNmM2U4ZmYsc3Ryb2tlOiM3YzNhZWQKICAgIHN0eWxlIEMgZmlsbDojZGNmY2U3LHN0cm9rZTojMTZhMzRhCg==?type=png)

Arrows are hard imports (`from resilience_kit.X import Y`). Dotted lines are entry-point conformance — no import required, just a Protocol-shaped class registered in the extra's `pyproject.toml` under `[project.entry-points."resilience_kit.cache"]` (and siblings). The kit's import graph is acyclic by import-linter contract; consumer apps depend only on the adapter layer.

The bet: a single distributable up to ~10k LOC is simpler to release, version, and depend on than a federation. If the surface ever grows past that, splitting into namespace packages later is mechanical and doesn't break the public API.

### 2. Protocols, not ABCs.

Every swappable interface — cache, breaker, throttle, audit backend, metrics sink — is a `typing.Protocol`, not an `abc.ABC`. Third-party backends import *zero* kit packages at definition time. They expose a class whose shape matches; `mypy --strict` verifies the structural conformance.

**The rule:** your code should conform to a shape, not inherit from a library. This is true dependency inversion — the kit doesn't even appear in the import graph of the thing implementing its protocol. Entry points provide the precedence chain at runtime: explicit > env > entry-point > default.

### 3. Outer breaker, inner retry. Composition order is correctness.

If you compose a circuit breaker and a retry, the breaker must wrap the retry. If you accidentally invert the stack — retry around breaker — the retry loop will hammer an `OPEN` breaker, defeating the point of the breaker. This isn't a style preference; it's a correctness property.

I learned this the hard way early on. A caller passed `retry_on=(Exception,)` to the retry decorator. The decorator happily caught `ServiceUnavailableError` — the exception the breaker raises to say "stop calling me." The fix (commit `f6cebdd`) made the dangerous composition fail safe by stripping `ServiceUnavailableError` from the retry's catch list, no matter what the caller passes:

```python
def _filter_retry_on(exceptions):
    """Drop ServiceUnavailableError from a retry_on tuple."""
    return tuple(
        e for e in exceptions
        if not (isinstance(e, type) and issubclass(e, ServiceUnavailableError))
    )
```

**The rule:** the dangerous composition must be the one that fails safe. An inverted stack now fails instantly, because the retry loop refuses to catch the one error a breaker is designed to throw.

### 4. Async-first with explicit sync bridges. Never trust auto-bridging.

Every primitive's async API is primary; sync wrappers exist only where the primitive is idiomatically sync (Django middleware, DRF throttles, management commands). Crucially, the kit **does not use `asgiref.sync.async_to_sync`**. That function caches a thread-local event loop to "avoid overhead" — which means it's a hidden global namespace, and any other library that also caches a thread-local loop will collide with it.

Instead, the kit owns both crossings explicitly:
- **Long-lived background work** (recovery monitor, audit dispatcher): the Django adapter spawns a single daemon thread with its own private event loop. The loop drives the monitor and owns the audit queue. `atexit` drains the queue on the *same* loop before closing it.
- **Short-lived sync→async calls** (DRF throttles checking Redis): a per-call `asyncio.run()`. Sub-millisecond overhead. No shared state. No collision risk.

**The rule:** a bridge between two worlds must own its own crossing. Don't borrow infrastructure from either side.

### 5. Adapters are dumb wiring. If your adapter has business logic, your primitive is wrong.

`resilience_kit.adapters.fastapi` and `resilience_kit.adapters.django` exist solely to wire kit primitives into each framework's lifecycle: ASGI middleware, FastAPI dependencies, Django `AppConfig.ready()`, DRF throttle classes, management commands. They translate framework-shaped concepts (a Django `MIDDLEWARE` tuple entry, a FastAPI `Depends()`) into kit calls. Nothing else.

If an adapter file ever grows past ~300 LOC, the primitive itself is wrong, not the adapter. This is enforced socially (PR review) and architecturally (import-linter prevents adapters from importing each other or from owning state).

The closing line of this post is the punchier statement of this axiom.

---

## The Dogfooding Gate

The most consequential decision in this project wasn't a code decision. It was a process decision:

> **"v0.1.0 is a clean cut iff both boilerplate reports score ≥ 8/10."**
> — `docs/RELEASE-PLAN.md` §4 verification

I write FastAPI and Django services. Before the kit existed, both starter repos owned the same resilience layer twice — circuit breakers, retries, throttles, SSRF guards, audit logs — copy-pasted and lightly diverged. The kit is the deduplication. But deduplication is easy to talk about and hard to validate; "the kit can replace the boilerplate code" is a claim, not a proof.

So I made it a release gate. Cut a release candidate. Upgrade both boilerplates against it. **File a structured report on each migration** — blockers, helpers used, missing surface, pain points, doc gaps, ROADMAP suggestions — and a 1–10 outcome score. If either score is < 8, the kit isn't done. Iterate. Re-test.

![Dogfooding cycle: cut RC → migrate both boilerplates → file reports → both ≥8/10 ships, otherwise iterate](https://mermaid.ink/img/Zmxvd2NoYXJ0IFRECiAgICBBW0N1dCByZWxlYXNlIGNhbmRpZGF0ZV0gLS0+IEJbVXBncmFkZSBmYXN0YXBpX2JvaWxlcnBsYXRlXQogICAgQSAtLT4gRFtVcGdyYWRlIGRqYW5nb19ib2lsZXJwbGF0ZV0KICAgIEIgLS0+IENbRmlsZSBGYXN0QVBJIHJlcG9ydDxici8+c2NvcmUgMeKAkzEwXQogICAgRCAtLT4gRVtGaWxlIERqYW5nbyByZXBvcnQ8YnIvPnNjb3JlIDHigJMxMF0KICAgIEMgLS0+IEZ7Qm90aCDiiaUgOC8xMD99CiAgICBFIC0tPiBGCiAgICBGIC0tIHllcyAtLT4gR1tDdXQgcmVsZWFzZTxici8+dGFnICsgUHlQSV0KICAgIEYgLS0gbm8gLS0+IEhbSXRlcmF0ZSBvbiBraXQ8YnIvPmZpeCBibG9ja2Vyc10KICAgIEggLS0+IEEKICAgIEcgLS0+IElbRmluZGluZ3MgZmVlZDxici8+bmV4dCBwYXRjaCArIG1pbm9yXQogICAgc3R5bGUgRiBmaWxsOiNmZWYzYzcsc3Ryb2tlOiNkOTc3MDYKICAgIHN0eWxlIEcgZmlsbDojZGNmY2U3LHN0cm9rZTojMTZhMzRhCiAgICBzdHlsZSBIIGZpbGw6I2ZlZTJlMixzdHJva2U6I2RjMjYyNgo=?type=png)

That cycle ran from `0.1.0rc1` → `0.1.0`. Both reports passed:

| Repo | Score | Date |
|---|---|---|
| [`fastapi_boilerplate`](https://github.com/prajwalmahajan101/fastapi_boilerplate) | **8 / 10** | 2026-06-10 |
| [`django_boilerplate`](https://github.com/prajwalmahajan101/django_boilerplate) | **9 / 10** | 2026-06-11 |

**Both reports applied all four primary helpers** that shipped in v0.1.0: `bind_to(target)` for ContextVar mirroring, `from_exception(exc)` for the kit-shape → adopter-envelope projection, `legacy_env_alias()` for pre-kit env-var translation, and `verify_envelope_contract()` for an adopter-side test that the bridge maps every kit exception cleanly. The fact that *both* reports used the same four — first try, no fallbacks — was the convergent signal that the helper surface was right.

What's interesting isn't that the gate passed. It's *what the gate produced*.

---

## What the Migration Reports Found

The reports were structured by design — same template, same sections — so the findings could be aligned and compared. Four findings showed up in both:

| # | Cross-cutting finding | Target |
|---|---|---|
| **X1** | The public exception bridge lives at `resilience_kit.adapters._envelope.from_exception` — the leading underscore reads as private. Re-export it without the underscore, or rename the module. | v0.1.1 |
| **X2** | `verify_envelope_contract` raises a flat `AssertionError`. Pytest only surfaces the first failure; programmatic CI dashboards can't introspect a structured result. Return an `EnvelopeContractResult` (list of `(exc_class, ok, reason)`). | v0.1.1 |
| **X3** | `legacy_env_alias(aliases=...)` *replaces* the default table. Adopters who want to extend with project-specific aliases must copy the default dict + add. Add `extra_aliases=` that merges, or document the copy-pattern explicitly. | v0.1.1 |
| **X4** | The rc1 → v0.1.0 migration guide had four documented gaps — different recipes needed Django-specific snippets, ordering was load-bearing in one place, the projection shape was wrong for a required-`code` envelope, and a `request_id=None` patch needed an explicit "top up from your own ContextVar" note. | v0.1.0 doc patch |

Two of those (X1, X2) are surface-quality nits — the kit works, but the ergonomics chafed. Two (X3, X4) are real correctness traps — silent override of the default alias table, and migration recipes that worked for FastAPI but not for DRF. Both reports independently flagged all four. That kind of convergence is exactly what dogfooding is for: a single user can rationalize around a sharp edge; two users hitting the same edge means the edge is the bug.

The single-repo findings were sharper.

**FastAPI-specific:** `from_exception(envelope_cls=...)` projects the kit's exception details into `[{field, message}]` per error-list-item. FastAPI's `ErrorDetail` schema also requires `code` — so the projection raised a `ValidationError` at handler time. Cost to diagnose: ~10 minutes. Workaround: drop `envelope_cls` from the call, translate manually in the handler (six lines). Tracked as a v0.1.1 patch (add `code=exc.error_code` to the projected items, or add a per-entry callback hook).

**Django-specific:** the migration guide's `verify_envelope_contract` example called `handler(...).body` — that's the FastAPI shape. DRF stores the body at `.data`, not `.body`. Doc-only fix; it landed in the same PR as the report.

**Smoke proof:** the Django report didn't just say the bridge worked. It quoted the response:

```
GET /api/accounts/me/  (anonymous, 401)

X-Request-Id: 1e43827d5faa4fcfa3551acfcf9caa27
{"request_id":"1e43827d5faa4fcfa3551acfcf9caa27", …}
```

The same UUID in the response header *and* in the JSON envelope's `request_id` field. rc1 returned `"request_id":null`. v0.1.0 returned the bound value. The bridge — `bind_to(request_id_ctx)` in a thin `BindRequestIdMiddleware` slotted immediately after the kit's `RequestIdMiddleware` — flowed kit → bridge → DRF handler → response. End to end. That's the kind of proof migration reports should ship.

**Convergent v0.2 wishlist.** Both reports independently asked for the same five things: `DjangoSettingsSource` (read `settings.RESILIENCE` instead of forcing env vars), `resilience_kit.utils.*` (five small modules every boilerplate re-implements: `log_sanitization`, `network`, `timing`, `function_logger`, `data`), a `GlobalThrottle` (process-wide cap; nginx covers it in prod but laptop dev wants the in-process belt), a free-function `metrics.record_*` shim over `MetricsSink` to keep adopters' bounded-cardinality guards in place, and multi-alias Redis topology so cache, throttle, and breaker can point at separate instances. That convergence is the v0.2 design spec, half-written for me.

---

## Bugs the Kit Found in Itself

The dogfooding gate is the systemic story. The bugs are the craft story.

### `f6cebdd` — outer breaker, inner retry

The bug behind axiom 3. A caller passed `retry_on=(Exception,)` to the retry decorator. The decorator caught `ServiceUnavailableError` — exactly the wrong exception. The fix is the `_filter_retry_on` snippet above. The rule it earned is in axiom 3.

![Inverted vs correct stack: retry-wraps-breaker hammers the OPEN state; breaker-wraps-retry contains transient failures and trips cleanly](https://mermaid.ink/img/Zmxvd2NoYXJ0IFRCCiAgICBzdWJncmFwaCBCQURbIuKdjCBJbnZlcnRlZCBzdGFjayDigJQgcmV0cnkgd3JhcHMgYnJlYWtlciJdCiAgICAgICAgZGlyZWN0aW9uIFRCCiAgICAgICAgQlIxW0NhbGxlcl0gLS0+IEJSMltyZXRyeSBkZWNvcmF0b3JdCiAgICAgICAgQlIyIC0tPiBCUjNbY2lyY3VpdCBicmVha2VyXQogICAgICAgIEJSMyAtLT4gQlI0W291dGJvdW5kIGNhbGxdCiAgICAgICAgQlI0IC0uLT58ZmFpbHN8IEJSMwogICAgICAgIEJSMyAtLi0+fE9QRU48YnIvPnJhaXNlczxici8+U2VydmljZVVuYXZhaWxhYmxlfCBCUjIKICAgICAgICBCUjIgLS4tPnxjYXRjaGVzIGl0PGJyLz5hbmQgcmV0cmllc3wgQlIzCiAgICAgICAgQlIyIC0uLT58aGFtbWVycyB0aGU8YnIvPk9QRU4gYnJlYWtlcnwgQlIzCiAgICBlbmQKICAgIHN1YmdyYXBoIEdPT0RbIuKchSBDb3JyZWN0IHN0YWNrIOKAlCBicmVha2VyIHdyYXBzIHJldHJ5Il0KICAgICAgICBkaXJlY3Rpb24gVEIKICAgICAgICBHUjFbQ2FsbGVyXSAtLT4gR1IyW2NpcmN1aXQgYnJlYWtlcl0KICAgICAgICBHUjIgLS0+IEdSM1tyZXRyeSBkZWNvcmF0b3JdCiAgICAgICAgR1IzIC0tPiBHUjRbb3V0Ym91bmQgY2FsbF0KICAgICAgICBHUjQgLS4tPnx0cmFuc2llbnQ8YnIvPmZhaWx1cmV8IEdSMwogICAgICAgIEdSMyAtLi0+fHJldHJ5IHdpdGg8YnIvPmppdHRlcnwgR1I0CiAgICAgICAgR1IzIC0uLT58Z2l2ZXMgdXA8YnIvPmFmdGVyIE4gYXR0ZW1wdHN8IEdSMgogICAgICAgIEdSMiAtLi0+fHRyaXBzIE9QRU48YnIvPm9uIHRocmVzaG9sZHwgR1IxCiAgICBlbmQKICAgIHN0eWxlIEJBRCBmaWxsOiNmZWUyZTIsc3Ryb2tlOiNkYzI2MjYKICAgIHN0eWxlIEdPT0QgZmlsbDojZGNmY2U3LHN0cm9rZTojMTZhMzRhCg==?type=png)

The inverted stack is the one a fresh reader writes first — composition order looks like a style preference until the breaker is `OPEN`. The `_filter_retry_on` invariant makes the wrong order fail safely instead of catastrophically: the retry decorator refuses to swallow the breaker's "stop" signal, even when the caller explicitly tells it to.

### `bc499da` — `asyncio.Event` rebinding on every restart

`RecoveryMonitor` held an `asyncio.Event` created in `__init__` and reused across every `start()`/`stop()` cycle. `asyncio.Event` doesn't bind to a loop at construction — it binds lazily, on the first `.wait()` or `.set()`. So the first start captured the loop. The second start, on a fresh loop, raised:

```
RuntimeError: <asyncio.locks.Event object at …> is bound to a different event loop
```

It surfaced in pytest-asyncio first — two FastAPI integration tests in one session, per-function loop scope. The second test's lifespan opened a fresh loop and called `monitor.start()`; the Event still held the first test's closed loop. Same shape of bug as Django's `runserver` autoreload, Gunicorn graceful restart, any ASGI lifespan that opens twice. The fix:

```python
# In src/resilience_kit/recovery.py, RecoveryMonitor.start()
def start(self):
    with self._lock:
        if self._task is not None and not self._task.done():
            return
        # Reassign rather than .clear() so the Event binds to the
        # *current* event loop. asyncio.Event lazily binds to
        # whichever loop first calls .wait()/.set() on it; reusing
        # the prior Event across loops (test harness, restarted
        # server) raises ``RuntimeError: bound to a different event
        # loop``. A fresh Event has no binding yet.
        self._stopping = asyncio.Event()
        self._task = asyncio.create_task(self._run(), name="resilience_kit.recovery_monitor")
    _logger.info("RecoveryMonitor started.")
```

Six lines changed. The lesson generalizes: `asyncio.Event`, `asyncio.Queue`, `asyncio.Lock` and several siblings capture their loop binding on first use, and `.clear()` doesn't unbind. **Loop-bound state is loop-scoped, not loop-local.** Construct fresh on every lifecycle restart; don't cache.

### `9f3d846` — `pybreaker` wrapper swallowed the tripping-call exception

The kit wraps `pybreaker.CircuitBreaker` as one of three breaker backends. On the call that trips the breaker (the one that crosses the failure threshold), `pybreaker` raises its own `CircuitBreakerError` to signal "I just opened" — discarding the underlying exception that *caused* the trip. The caller saw "breaker open" without knowing *why* it opened. Forensics suffered.

The fix: the wrapper captures the original exception in a small state object before letting `pybreaker` raise, then re-raises the original on the tripping call. The breaker still opens; the caller still sees the real failure cause. **Don't let third-party wrappers eat your forensic signal.**

### Honourable mentions

- `aaa5f41` — ASGI header bytes vs str. The request_id middleware did `.lower()` on `b"X-Request-Id"`. Worked on Starlette's str-typed headers, broke on raw ASGI bytes. Boundary type discipline.
- `1c8f834` — settings cache leaking the test source between tests, masking what `legacy_env_alias` actually resolved. The reset helper now restores the *default* `EnvSettingsSource`, not the last test's override.
- `e588e8c` — CodeQL flagged a dict whose keys were unioned `str | bytes`. The fix was three characters. The lesson: take CodeQL findings seriously even when "it works"; the static analyser sees the type-confusion path the runtime tests don't.

These are real fixes from the kit's git history. None of them were dramatic. All of them earned a small piece of the design.

---

## Building the Dangerous Parts First

Two areas in the kit are trivial to get wrong and catastrophic when they fail.

### SSRF guard + DNS rebinding TOCTOU

An SSRF guard that just validates a resolved IP is vulnerable to a Time-of-Check-to-Time-of-Use (TOCTOU) race. The attacker's DNS returns a safe public IP at check time; the SSRF guard approves; the underlying HTTP transport does its own DNS lookup; the attacker's DNS returns `127.0.0.1` this time; the request lands on localhost.

The fix is to make the check and the connect share state. `AsyncAPIClient` resolves the host, validates the IP against the SSRF allow-list, and pins the validated IP into a `contextvars.ContextVar`. A custom `httpx` transport reads the pinned IP and connects directly — skipping the second DNS lookup entirely while preserving the original `Host` header and TLS SNI hostname so cert verification still passes.

![SSRF Guard and DNS Pinning Flow](https://mermaid.ink/img/c2VxdWVuY2VEaWFncmFtCiAgICBwYXJ0aWNpcGFudCBDIGFzIENsaWVudAogICAgcGFydGljaXBhbnQgUksgYXMgUmVzaWxpZW5jZSBLaXQKICAgIHBhcnRpY2lwYW50IEROUSBhcyBETlMgUmVzb2x2ZXIKICAgIHBhcnRpY2lwYW50IEhUVFBZIGFzIEhUVFBZIFRyYW5zcG9ydAogICAgcGFydGljaXBhbnQgUyBhcyBUYXJnZXQgU2VydmVyCgogICAgQy0+PlJLOiBodHRweF9jbGllbnQuZ2V0KCJodHRwczovL2V4YW1wbGUuY29tIikKICAgIFJLLT4+RE5ROiBSZXNvbHZlICJleGFtcGxlLmNvbSIKICAgIEROUS0tPj5SSzogMS4yLjMuNAogICAgUkstPj5SSzogU1NSRkd1YXJkLnZhbGlkYXRlKDEuMi4zLjQpCiAgICBhbHQgU1NSRiBjaGVjayBmYWlscwogICAgICAgIFJLLS0+PkM6IFJhaXNlIFNTUkZHdWFyZEVycm9yCiAgICBlbmQKICAgIFJLLT4+Uks6IFBpbiAxLjIuMy40IHRvIENvbnRleHRWYXIKICAgIFJLLT4+SFRUUFk6IGdldCguLi4pCiAgICBIVFRQWS0+PlJLOiBSZWFkIHBpbm5lZCBJUCBmcm9tIENvbnRleHRWYXIKICAgIEhUVFBZLT4+UzogQ29ubmVjdCB0byAxLjIuMy40IChza2lwcyBETlMpCiAgICBTLS0+PkhUVFBZOiBSZXNwb25zZQogICAgSFRUUFktLT4+Uks6IFJlc3BvbnNlCiAgICBSSy0tPj5DOiBSZXNwb25zZQ==)

The check and the connect now share a single piece of state. TOCTOU bypassed.

### Fire-and-forget audit dispatch

Audit logs can't bring down the application. The pre-kit boilerplates wrote audit rows synchronously — if the database hiccupped, the request failed. The kit replaces that with a decoupled, fire-and-forget pipeline:

- An in-memory, bounded `asyncio.Queue` (default 10,000 events) acts as a buffer.
- A background worker flushes batches to the configured audit backend (`postgres`, `stdlib_logging`, or a custom entry-point backend).
- **If the database is down, the dispatcher degrades gracefully.** It retries with backoff; if it ultimately fails, it logs the batch to stderr and increments a `dropped` metric. The app survives.

The audit dispatcher is also one of the loop-owned resources from axiom 4 — the Django adapter spawns it inside its daemon thread's private loop and drains it via `atexit` on the same loop before closing.

---

## Verification: Testing What Unit Tests Can't See

The test suite splits three ways. Unit tests verify logic in isolation. Contract tests verify the same logic works against every backend. Integration tests verify the whole stack survives chaos.

**Contract tests: the same assertion, every backend.** The kit defines multiple backends — memory, Redis, `pybreaker` for circuit breakers. The same test suite runs against all of them, parametrized by `pytest.mark.parametrize("backend", …)`. A Lua script in `redis_impl.py` that diverges from the in-memory contract fails the same assertion that passes against `memory_impl.py`. The contract suite catches it the moment it ships.

**SSRF TOCTOU validation.** The integration test (`tests/integration/test_dns_rebinding.py`) mocks `socket.getaddrinfo` with a `_RebindingResolver` that returns a safe public IP on the first call and a private IP on the second. The SSRF guard validates the first IP and pins it. The connection should use only that pinned IP — no second resolution. The test asserts the underlying transport's request URL carries the pinned IP, not the attacker's late-arriving private one.

**Recovery monitor validation.** The integration test (`tests/integration/test_recovery_monitor.py`) spins up a real Redis with `testcontainers`, calls `docker_client.api.pause(container_id)` to freeze TCP without killing the connection, verifies that throttles degrade to in-memory and flag degradation in metrics, then unpauses and verifies the monitor restores the Redis-backed providers within the 5-second exit-gate window. This test is what caught the `bc499da` bug — the second invocation of the monitor across pytest's per-function loops blew up exactly as described above.

The kit ships with 52 test modules across unit, contract, and integration suites.

---

## What I'd Do Differently

Four regrets from the critical path:

1. **Chaos and failure injection should come before v0.2.** I built the core under happy-path assumptions, then spent the adapter phase hardening it. Testcontainers, container-pause cycles, Redis-down scenarios — these should have been mandatory from day one, not post-release validation. Every failure mode I caught during adapter work would have reshaped a primitive's design if I'd seen it earlier.

2. **CI matrix (Python 3.11, 3.12, 3.13) should be commit-one, not late hardening.** I assumed 3.11 compatibility, then discovered version-specific differences in asyncio behaviour and stdlib deprecations during release prep — exactly the work a GitHub matrix would have done on every PR for free. Fixes were mostly mechanical; the cost was the discovery cycle.

3. **Contract test suite should precede any backend implementation.** I wrote backends (memory, Redis, `pybreaker`) and then wrote tests to validate them. Tests written *after* backends encode the backends' assumptions as the spec, not the spec's expectations as the test. Contract tests written first force you to commit to the *behaviour* before you commit to the *implementation* — and they catch backend-specific divergence at the boundary rather than during integration. I got there eventually; getting there first would have been cheaper.

4. **`mypy --strict` should be enforced from commit-one, not applied retroactively.** I shipped the first half of the project without strict type hints, then made it a release gate. Retrofitting types to async code with complex control flow is error-prone, and the rewrite surfaced structural mismatches between `Protocol` declarations and the backends meant to satisfy them. Strict types from the start force the design right the first time.

These are design regrets, not dogfooding bugs. The dogfooding bugs fed v0.1.1 and v0.2. The design regrets are about how I'd budget time on the next from-scratch infrastructure project.

---

## What's Next

The convergent v0.2 wishlist from both migration reports is half-written for me:

- **`DjangoSettingsSource`** — read `settings.RESILIENCE` directly instead of forcing env-only config. Both reports rank this P1.
- **`resilience_kit.utils.*`** — five small modules (`log_sanitization`, `network`, `timing`, `function_logger`, `data`) every boilerplate re-implements. Promoting them into the kit saves ~900 LOC per consumer.
- **`MetricsSink` cardinality contract** — bound the labels at `record_*` time, not after a Prometheus cardinality explosion in prod.
- **`GlobalThrottle`** (Valkey-Lua) — process-wide cap. Nginx covers it for fleet deployments; the in-process belt is for laptop dev and single-pod deploys.
- **Multi-alias Redis topology** — `RESILIENCE_REDIS_URLS__<alias>` so cache, throttle, and breaker can point at separate instances.

The v0.2 exit gate is the same rule, raised: both boilerplates re-test against a v0.2 pre-release and score **≥ 8.5 / 10** — half a point higher than 0.1.0.

Full roadmap: [`ROADMAP.md`](https://github.com/prajwalmahajan101/resilience-kit/blob/main/docs/ROADMAP.md).

I started by trying to delete duplication. I ended up with five axioms, a dogfooding release rule, and a set of bug-earned invariants that are reusable far beyond this kit. The duplication was the symptom. The axioms are the fix.

---

## Notes & references

Each axiom and dangerous-parts decision has a one-page ADR in the repo. If you want the long-form reasoning — alternatives considered, consequences, usage notes — these are the load-bearing ones:

| Topic in this post | ADR |
|---|---|
| Protocols, not ABCs | [`0001-protocol-not-abc`](https://github.com/prajwalmahajan101/resilience-kit/blob/main/docs/adr/0001-protocol-not-abc.md) |
| Hand-rolled retry over `tenacity` | [`0002-handrolled-retry-not-tenacity`](https://github.com/prajwalmahajan101/resilience-kit/blob/main/docs/adr/0002-handrolled-retry-not-tenacity.md) |
| One package, many extras | [`0003-single-package-with-extras`](https://github.com/prajwalmahajan101/resilience-kit/blob/main/docs/adr/0003-single-package-with-extras.md) |
| Entry-point backends + precedence chain | [`0004-entry-points-for-third-party-backends`](https://github.com/prajwalmahajan101/resilience-kit/blob/main/docs/adr/0004-entry-points-for-third-party-backends.md), [`0009-entry-point-precedence-chain`](https://github.com/prajwalmahajan101/resilience-kit/blob/main/docs/adr/0009-entry-point-precedence-chain.md) |
| Fire-and-forget audit dispatch | [`0005-fire-and-forget-audit`](https://github.com/prajwalmahajan101/resilience-kit/blob/main/docs/adr/0005-fire-and-forget-audit.md) |
| Outer breaker, inner retry | [`0006-outer-breaker-inner-retry`](https://github.com/prajwalmahajan101/resilience-kit/blob/main/docs/adr/0006-outer-breaker-inner-retry.md) |
| DNS pin via `ContextVar` (TOCTOU) | [`0007-dns-pin-via-contextvar`](https://github.com/prajwalmahajan101/resilience-kit/blob/main/docs/adr/0007-dns-pin-via-contextvar.md) |
| Fernet env-guarded keys | [`0008-fernet-env-guard`](https://github.com/prajwalmahajan101/resilience-kit/blob/main/docs/adr/0008-fernet-env-guard.md) |
| FastAPI adapter shape | [`0010-fastapi-adapter-shape`](https://github.com/prajwalmahajan101/resilience-kit/blob/main/docs/adr/0010-fastapi-adapter-shape.md) |
| Django sync/async bridge | [`0011-django-sync-async-bridge`](https://github.com/prajwalmahajan101/resilience-kit/blob/main/docs/adr/0011-django-sync-async-bridge.md) |

Migration reports: [`docs/m8b-upgrade-reports/`](https://github.com/prajwalmahajan101/resilience-kit/blob/main/docs/m8b-upgrade-reports/) — `SUMMARY.md`, `fastapi_boilerplate.md`, `django_boilerplate.md`.

---

> The adapter is not the kit. The kit is what survives without the adapter.

---
*Source Code:* [resilience-kit GitHub Repository](https://github.com/prajwalmahajan101/resilience-kit)
*Documentation:* [Design Doc (PRD)](https://github.com/prajwalmahajan101/resilience-kit/blob/main/docs/PRD.md) | [Architecture (LLD)](https://github.com/prajwalmahajan101/resilience-kit/blob/main/docs/LLD.md) | [ADR Index](https://github.com/prajwalmahajan101/resilience-kit/blob/main/docs/adr/) | [Migration Reports](https://github.com/prajwalmahajan101/resilience-kit/blob/main/docs/m8b-upgrade-reports/)
