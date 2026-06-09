---
title: "Building a message broker in 4 days — and why the order mattered more than the code"
seoTitle: "Risk-ordered development: a 4-day Go message broker post-mortem"
seoDescription: "A SHA-by-SHA retrospective on toymq, a single-node persistent message broker built in Go. Why WAL-first, protocol-second, broker-third is the order that lets the code survive itself."
slug: building-toymq-risk-ordered
tags: go, distributed-systems, message-broker, software-engineering, software-architecture

---

> A retrospective on `toymq` — a single-node persistent message broker I built in Go over four days. The point of this post isn't the broker. It's the order I built it in. Source: [github.com/prajwalmahajan101/toymq](https://github.com/prajwalmahajan101/toymq). SHA-by-SHA breakdown: [`docs/STORY.md`](https://github.com/prajwalmahajan101/toymq/blob/main/docs/STORY.md).

## TL;DR

`toymq` is 51 commits on `main`, 9 feature branches, 12 ADRs, 88 tests across 19 files, and the whole thing came together between 2026-06-05 20:15 IST and 2026-06-09 13:29 IST. About 3 days 17 hours of clock time. None of that is interesting. What's interesting is that I built it in this order:

```
WAL → wire protocol → broker (×2) → server/session → cmd wiring → integration tests → chaos
```

That order is the whole post. If you skip the rest, take this away: **build risk-first, not feature-first.** Build the thing that, if it's wrong, makes everything above it have to be rewritten. Then build the thing above it.

## Why "build the bottom first" is non-obvious advice

Every tutorial-flavored project I've ever seen does the opposite. You build a "hello world" server, you make it accept connections, you parse a request, you stub a handler, you add real handlers, you add persistence last. The blogger photographs each step. By the time you get to durability, you've already shipped a wire protocol that assumes you don't have one, a handler API that assumes nothing crashes, and a test suite that mocks the part of the system that actually matters.

The reason this happens is that the bottom of the stack is the least visible. Nobody opens GitHub to admire a write-ahead log. They open it to see the API. So the temptation is to start where the screenshot lives.

`toymq` is the opposite. The first real commit isn't a server. It's a framed record format and a recovery scan. Day 1 is "what is on disk after a crash" and nothing else. Day 2 is "what goes over the wire." The broker — the thing that does the actual brokering — doesn't show up until Day 3.

The reason this works is selfish: **the bottom of the stack is the thing that, if I get it wrong, forces a rewrite of everything above it.** A WAL bug means the broker is wrong. A protocol bug means every client and the broker are wrong. A broker bug means the server is wrong. A server bug means… the server is wrong. Risk shrinks as you go up. So I do the risky thing first, while my budget for being wrong is highest.

## Branch 1 — WAL: lock durability before touching anything else

I gave myself one rule for the first branch: **the on-disk format is a contract.** Anything that lands on disk in this branch has to survive forever, because the recovery scan is going to assume it. If I want to change the format later, I'm writing a migration.

Three ADRs came out of this branch:

- **[ADR-0001](https://github.com/prajwalmahajan101/toymq/blob/main/docs/adr/0001-framed-record-format.md)** — length-prefixed records with a CRC. Boring on purpose. The interesting part is what's *not* in the record: no version byte. Adding a version byte after the fact is a one-line migration; defending the wrong version byte forever is not.
- **[ADR-0002](https://github.com/prajwalmahajan101/toymq/blob/main/docs/adr/0002-per-message-fsync.md)** — yes, per-message `fsync`. Yes, it's slow. The broker is single-node and this is a learning artifact, not a Kafka competitor; correctness budget gets spent here, not on throughput tricks I can't defend.
- **[ADR-0003](https://github.com/prajwalmahajan101/toymq/blob/main/docs/adr/0003-recovery-by-scan.md)** — recover by scanning the segment, not by reading a manifest. Manifests are a separate consistency problem; scans are O(disk) but they're correct by construction.

Each ADR has the same shape: **Context, Decision, Consequences.** I didn't write any of them before the code that forced the decision. ADRs as crystallization, not ceremony — I'll come back to that.

## Branch 2 — protocol: lock the wire shape before consumers exist

[ADR-0004](https://github.com/prajwalmahajan101/toymq/blob/main/docs/adr/0004-proto-sealed-types.md) is a "sealed" frame union. You can't add a new frame type without recompiling the broker *and* every client. That sounds annoying. It is. It's also the only way to actually catch wire-shape drift at compile time instead of in production.

Two patterns from this branch are worth naming:

**Tests as contracts.** The protocol tests don't just verify round-trip encode/decode. They verify that **every frame variant** round-trips, which means adding a new variant without a test fails compilation in the test file too. The test is the contract, not the docstring.

**Cheap primitives.** No state machines, no fancy framing libraries. Length prefix, type byte, payload. The whole encoder fits in your head. The cheapest thing that's correct beats the cleverest thing that's almost correct.

## Branches 3+4 — the broker: split work into "structure" and "semantics"

The broker took two branches, not one, and the split mattered.

Branch 3 was **structure** — topics, partitions (one per topic in v1, but the interface assumes more), in-memory indexes over the WAL. The acceptance criterion was "you can produce a message and consume it; if you crash, recovery works." No visibility timeouts, no consumer groups, nothing fancy.

Branch 4 was **semantics** — visibility timeouts, redelivery on timeout, "has this consumer acked?" tracking. [ADR-0007](https://github.com/prajwalmahajan101/toymq/blob/main/docs/adr/0007-visibility-timeout-redelivery.md) and [ADR-0011](https://github.com/prajwalmahajan101/toymq/blob/main/docs/adr/0011-consumer-state-hasacked.md) live here.

I tried to do this as one branch first. I rolled it back at the second merge conflict against myself. **The 2-merge rule:** if a branch produces two conflicts against `main` before it's done, the branch is too big. Split it. The cost of splitting is one extra merge commit; the cost of not splitting is that everything in the branch ships together with whatever bug got introduced halfway through.

## Branch 5 — server/session: the part everyone thinks they're building

By the time the server existed, the hard part was done. The session layer is a thin shell over the broker; it handles connection lifecycle, dispatches frames, and stays out of the way. [ADR-0008](https://github.com/prajwalmahajan101/toymq/blob/main/docs/adr/0008-session-concurrency-model.md) — one goroutine per session for reads, one for writes, no shared state between them — is the entire concurrency story. Boring. Boring is good here.

If I'd started with this layer, I would have built a session abstraction that assumed an in-memory broker, then rewritten it when the WAL showed up, then rewritten it again when visibility timeouts shipped. By inverting the order, the session layer was written *once*, against an interface that already existed and already passed tests.

## Branches 6, 7, 8 — wiring, integration tests, chaos

The last three branches are where you finally start to look at the project the way an outside reader will. `cmd/` wiring, end-to-end tests that drive the real server with real clients ([ADR-0010](https://github.com/prajwalmahajan101/toymq/blob/main/docs/adr/0010-integration-test-architecture.md)), and a chaos branch ([ADR-0012](https://github.com/prajwalmahajan101/toymq/blob/main/docs/adr/0012-chaos-test-architecture.md)) that runs concurrent producers and consumers against a process that gets killed and restarted mid-flight.

The chaos tests caught two bugs that the unit tests didn't, both in the same place: the boundary between "I've written this message to the WAL and called `fsync`" and "I've acknowledged it to the producer." Concurrency-adjacent, recovery-adjacent. The kind of bug that only shows up when the timing is wrong, which is exactly what chaos tests exist to find. **Integration and chaos tests aren't a coverage exercise; they're the only way to test the seams.**

## ADRs as crystallization, not ceremony

Twelve ADRs in four days sounds like ceremony. It wasn't.

The rule I followed: **write the ADR the moment the decision is forced by code, never before.** Not when I had a hunch. Not when I was sketching. When I had to type the words "we are doing it this way" into a function, *then* the ADR went next to it.

This sounds like a small distinction. It isn't. ADRs written before the code force the code to fit the ADR, which means you spend a week defending a decision you made when you knew the least. ADRs written *at* the moment of crystallization record what the code already decided, which means the ADR is always honest. If the code later changes its mind, you supersede the ADR — you don't try to make the old ADR retroactively make sense.

The one ADR I broke this rule for is on a branch that isn't merged. [ADR-0013](https://github.com/prajwalmahajan101/toymq/blob/main/docs/adr/0013-pkg-client-architecture.md) on `feat/pkg-client` was written before the client package was real. It shows. The decisions in it are the right shape, but the prose is sketchier than the others. That's the tell.

## What I'd do differently

- **Per-message `fsync` is too slow for anything real.** I knew this; the ADR says so. The thing I'd change isn't the decision, it's the documentation — a benchmark in the README would make the trade-off visible from the first page instead of buried in ADR-0002.
- **The protocol's sealed-union is annoying for plugin authors.** It's the right call for a single-node broker maintained by one person, but if I ever ship this for other people to extend, that decision has to be revisited. ADR follow-up needed.
- **Chaos tests should have come one branch earlier.** They caught the seam bugs, but they caught them late. Integration tests on Branch 7 didn't find what chaos found on Branch 8; that's a hint chaos was the right tool, just deployed one branch too late.

## What's next

- `v1.1` — benchmark suite + the docs follow-up on `fsync` cost.
- `feat/pkg-client` — the client library currently un-merged; once the API stabilizes, ADR-0013 gets rewritten properly and the branch lands.
- Eventually: WAL compaction, multiple partitions per topic, consumer groups. None of those are this weekend's problem.

## The one-line version

If a tutorial tells you to build the server first, build the storage first. The order is the project.

---

*Source repo: [`prajwalmahajan101/toymq`](https://github.com/prajwalmahajan101/toymq). Full SHA-by-SHA story: [`docs/STORY.md`](https://github.com/prajwalmahajan101/toymq/blob/main/docs/STORY.md). ADR index: [`docs/adr/README.md`](https://github.com/prajwalmahajan101/toymq/blob/main/docs/adr/README.md). Comments and corrections welcome — [open a Discussion](https://github.com/prajwalmahajan101/toymq/discussions) or file an issue.*
