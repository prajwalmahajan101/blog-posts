---
title: "Building toymq: a from-scratch persistent message broker in Go"
published: true
description: "A retrospective on building toymq, a single-node persistent message broker in Go. Why building the dangerous parts first (WAL → protocol → broker) keeps the stack correct under SIGKILL — with diagrams, 17 ADRs, and a chaos suite."
tags: go, distributedsystems, backend, architecture
series: "Building distributed systems"
cover_image: https://cdn.hashnode.com/uploads/covers/66489439ca1dde2125af89da/d197116d-b313-4d27-b894-1e6defc1031b.jpg
canonical_url: https://dev.to/prajwalmahajan101/building-toymq-a-from-scratch-persistent-message-broker-in-go-ob7

# dev.to metadata
# article_id: 3859846
# published_at: 2026-06-09T20:04:45Z
# url: https://dev.to/prajwalmahajan101/building-toymq-a-from-scratch-persistent-message-broker-in-go-ob7
---

> A retrospective on [`toymq`](https://github.com/prajwalmahajan101/toymq) — a single-node persistent message broker in Go, recreated by hand to understand one of the smallest functional units in a distributed system.

## TL;DR — and the bug you should remember

**Build the dangerous parts first.** Build the thing that, if it's wrong, forces a rewrite of everything above it. Then build the thing above it.

The worst bug of the project took six commits to surface and an integration test to catch: a `uint64` consumer offset where `0` meant *both* "message id 0 was acked" *and* "never acked." Unit tests never restarted the broker between Ack and Subscribe, so the collision stayed invisible. **Zero values are not sentinels.** That's the lesson the post earns.

The numbers: ~10k lines of Go, 17 ADRs, four binaries, 90.3% test coverage, a chaos harness that pushed 14,000 messages through three `SIGKILL`/restart cycles with zero acked-message loss. v1.0 in four days; v1.3 plus post-release hardening in another two.

The order:

```
WAL → wire protocol → broker → session → server → cmd wiring
   → integration tests → chaos → post-v1.0 hardening
```

The point of this post is the order, not the code.

## What toymq is, and isn't

A learning artifact. Single node. No replication. No auth. No TLS. Throughput tops out at a few hundred messages per second because every publish does a per-message `fsync`. If you need a real broker, reach for NATS or RabbitMQ. If you want to *understand* what a broker is — what `durable` actually means, what `at-least-once` costs, what crash recovery requires — build one. That's what this post is about.

## Why "build the bottom first" is non-obvious

Every tutorial does the opposite. You build a server, accept connections, parse a request, stub a handler, add real handlers, add persistence last. By the time you reach durability, you've shipped a wire protocol that assumes you don't have one, a handler API that assumes nothing crashes, and a test suite that mocks the part of the system that matters.

`toymq` inverts that. The first commit isn't a server — it's a framed record format and a CRC check. Day 1 is "what is on disk after a crash" and nothing else.

This works for a selfish reason: **the bottom of the stack, if I get it wrong, forces a rewrite of everything above it.** A storage bug means the broker is wrong. A protocol bug means every client *and* the broker are wrong. Risk shrinks as you go up. So you spend the budget for being wrong at the bottom, while the budget is highest.

## Durability: what should "on disk" actually mean?

The storage piece of any persistent system is the **write-ahead log** — a file you append records to, fsync, and never go back to mutate. On crash, you scan it from the start and rebuild your state.

The one rule for the WAL: **the on-disk format is a contract.** Anything that lands on disk has to survive forever, because the recovery scan will assume it. Changing the format later means writing a migration.

### On-disk layout

```
data/
└── topics/
    ├── orders/
    │   ├── segment.log     ← append-only WAL, one record per PUB
    │   └── offsets.json    ← per-consumer state, written by debouncer
    └── events/
        ├── segment.log
        └── offsets.json
```

One directory per topic. Two files per topic. No metadata file, no index, no manifest. The recovery scan walks every segment from offset zero on `Open`. Scans are O(disk) but correct by construction — the WAL is its own truth, and a manifest would be a second consistency problem on top of the first.

### The record frame

| Field         | Type    | Size            | Notes                              |
|---------------|---------|-----------------|------------------------------------|
| `length`      | `u32`   | 4 bytes         | Bytes that follow, excluding self  |
| `msg_id`      | `u64`   | 8 bytes         | Monotonic per topic                |
| `ts_ns`       | `u64`   | 8 bytes         | Append timestamp, ns since epoch   |
| `key_len`     | `u16`   | 2 bytes         | 0 if the message has no key        |
| `key`         | bytes   | `key_len` bytes | Dedupe key                         |
| `payload_len` | `u32`   | 4 bytes         | Payload length in bytes            |
| `payload`     | bytes   | `payload_len`   | Message body                       |
| `CRC32`       | `u32`   | 4 bytes         | Checksum over all preceding bytes  |

Deliberately boring. The interesting part is what's *not* there: **no version byte.** Adding a version byte later is a one-line migration; defending the wrong version byte forever is not. Don't add knobs the format doesn't yet need.

### `fsync` is the durability commit point

`Log.Append` is the only function in toymq that calls `fsync`. Every `PUB` goes through it. Every `OK` the broker emits is, by construction, preceded by a successful `fsync` of the corresponding record:

![WAL Append sequence: write → fsync → committedOffset → release](https://mermaid.ink/img/c2VxdWVuY2VEaWFncmFtCiAgICBhdXRvbnVtYmVyCiAgICBwYXJ0aWNpcGFudCBUIGFzIFRvcGljCiAgICBwYXJ0aWNpcGFudCBMIGFzIFdBTCBMb2cKICAgIHBhcnRpY2lwYW50IEZTIGFzIEtlcm5lbC9GaWxlc3lzdGVtCgogICAgVC0-Pkw6IEFwcGVuZChyZWNvcmQpCiAgICBMLT4-TDogZW5jb2RlIChDUkMgKyBsZW4gKyBib2R5KQogICAgTC0-Pkw6IGFjcXVpcmUgcHViTXUKICAgIEwtPj5GUzogZmlsZS5Xcml0ZShieXRlcykKICAgIE5vdGUgb3ZlciBMLEZTOiBCeXRlcyBpbiBwYWdlIGNhY2hlLjxici8-UG93ZXIgbG9zcyBoZXJlIGxvc2VzIHRoZW0uCiAgICBMLT4-RlM6IGZpbGUuU3luYygpIOKAlCBmc3luYygyKQogICAgTm90ZSBvdmVyIEwsRlM6IEJ5dGVzIG9uIHN0YWJsZSBzdG9yYWdlLjxici8-RHVyYWJpbGl0eSBjb21taXQgcG9pbnQuCiAgICBGUy0tPj5MOiBvawogICAgTC0-Pkw6IGNvbW1pdHRlZE9mZnNldC5TdG9yZShuZXdPZmZzZXQpCiAgICBMLT4-TDogY29uZC5Ccm9hZGNhc3QoKQogICAgTC0-Pkw6IHJlbGVhc2UgcHViTXUKICAgIEwtLT4-VDogbXNnSUQsIG5ld09mZnNldA?type=png)

The committed-offset update happens *after* fsync. Any reader that sees `committedOffset == X` is guaranteed the bytes through `X` are on stable storage. Cost: ~1–2 ms p99 on commodity NVMe. The correctness budget gets spent here, not on throughput tricks I can't defend. A broker that loses an acked message is not a broker.

### How this compares to Kafka

Kafka does almost none of this. Its durability story is page cache + replication: trust the OS to flush eventually, trust the replicas to catch up. It pushes hundreds of thousands of messages per second per broker as a result. toymq has neither replicas nor the throughput budget to assume the page cache wins, so it trusts `fsync` directly. **Different problem, different answer — and the gap between those answers is exactly what hand-rolling teaches you.**

The first nasty bug landed here, too: a `make([]byte, payloadLen)` that ran *before* checking `payloadLen <= maxPayload`. A malicious client could OOM the broker with a single packet claiming a 100 GB payload. The fix is one line; the rule is general — **validate before allocating, always.**

## The wire protocol: agreeing on what happened

Once the storage layer holds, the next question is how clients and broker talk. A protocol is the contract that lets the two ends disagree about *when* something happened (network is unreliable) without disagreeing about *whether* it happened.

toymq's protocol fits in a paragraph: a command word, a few arguments, a length-prefixed payload. The full `PUB` happy path, end to end:

![PUB happy path: Client → Reader → Handler → Broker → Topic → WAL fsync → OK](https://mermaid.ink/img/c2VxdWVuY2VEaWFncmFtCiAgICBhdXRvbnVtYmVyCiAgICBwYXJ0aWNpcGFudCBDIGFzIENsaWVudAogICAgcGFydGljaXBhbnQgU1IgYXMgU2Vzc2lvbiBSCiAgICBwYXJ0aWNpcGFudCBTSCBhcyBTZXNzaW9uIEgKICAgIHBhcnRpY2lwYW50IFNXIGFzIFNlc3Npb24gVwogICAgcGFydGljaXBhbnQgQiBhcyBCcm9rZXIKICAgIHBhcnRpY2lwYW50IFQgYXMgVG9waWMKICAgIHBhcnRpY2lwYW50IFcgYXMgV0FMIExvZwoKICAgIEMtPj5TUjogUFVCIG9yZGVycyBrMSA1XG5oZWxsb1xuCiAgICBTUi0-PlNIOiBQdWJDb21tYW5ke1RvcGljLCBLZXksIFBheWxvYWR9CiAgICBTSC0-PkI6IFB1Ymxpc2godG9waWMsIGtleSwgcGF5bG9hZCkKICAgIEItPj5UOiBQdWJsaXNoKGtleSwgcGF5bG9hZCkKICAgIFQtPj5UOiBEZWR1cGUuTG9va3VwKGtleSkg4oaSIG1pc3MKICAgIFQtPj5XOiBBcHBlbmQocmVjb3JkKQogICAgTm90ZSBvdmVyIFc6IGZzeW5jKCkg4oCUIGR1cmFiaWxpdHkgY29tbWl0CiAgICBXLS0-PlQ6IG1zZ0lELCBvZmZzZXQKICAgIFQtPj5UOiBEZWR1cGUuSW5zZXJ0KGtleSwgbXNnSUQpCiAgICBULS0-PkI6IG1zZ0lECiAgICBCLS0-PlNIOiBtc2dJRAogICAgU0gtPj5TVzogV3JpdGVPSyhtc2dJRCkKICAgIFNXLS0-PkM6IE9LIDBcbg?type=png)

The `OK` only crosses the network *after* `fsync` returns. That's the durability promise made visible: the client never sees an `OK` for a message that isn't on disk.

Two design choices shape everything above this layer:

**A sealed `Command` type.** The parser dispatches on an interface with an unexported marker method. You cannot add a new command type without editing the file the parser lives in. The compiler keeps the protocol and its handler in sync — if you add a command and forget to handle it, the build breaks. Free safety.

**Clean EOF vs torn header.** The parser distinguishes a stream closed cleanly *between* commands (propagate `io.EOF`) from a stream closed *mid-line* (return `ErrBadFraming`). That distinction recurs everywhere downstream: the session loop uses it to tell "client gracefully disconnected" from "client crashed and dropped." Most tutorials skip this and pay for it forever.

## Brokering, in two parts: structure vs semantics

A broker has to do two things: route messages from producers to consumers, and guarantee something about the delivery (`at-least-once`, in toymq's case). I split this into two branches and the split mattered.

**Structure** came first: topics (auto-created on first publish), a WAL per topic, an LRU dedupe index keyed by `(producer-id, msg-id)`, in-memory consumer state. Acceptance bar: "you can publish a message and consume it; if the broker restarts, recovery works." No visibility timeouts, no NACKs.

**Semantics** came second. Every message a consumer has been told about lives in one of three states:

![Visibility-timeout state machine: Pending ↔ Inflight → Acked](https://mermaid.ink/img/c3RhdGVEaWFncmFtLXYyCiAgICBbKl0gLS0-IFBlbmRpbmcgOiBXQUwuQXBwZW5kIHByb2R1Y2VzIE1zZ0lECiAgICBQZW5kaW5nIC0tPiBJbmZsaWdodCA6IHJ1bkRlbGl2ZXJ5IHNlbmRzIE1TR1xuKFNlbnRBdCArIEF0dGVtcHRzKyspCiAgICBJbmZsaWdodCAtLT4gQWNrZWQgOiBDbGllbnQgQUNLCiAgICBJbmZsaWdodCAtLT4gUGVuZGluZyA6IHZpc2liaWxpdHkgdGltZW91dCBmaXJlc1xuKG5vdyAtIFNlbnRBdCA-IFZpc2liaWxpdHkpCiAgICBJbmZsaWdodCAtLT4gUGVuZGluZyA6IENsaWVudCBOQUNLCiAgICBBY2tlZCAtLT4gWypd?type=png)

A message can bounce between `Pending` and `Inflight` many times before reaching `Acked` — that's the at-least-once contract. The chaos suite's no-loss invariant verifies that every acked `MsgID` reaches at least one consumer's seen set. Duplicates above one are allowed; loss is not.

The redelivery ticker is the load-bearing piece, and it forced the rule that became the hot-path discipline for the entire broker:

**Never send while holding the inflight lock.**

```go
c.inflightMu.Lock()
pending := append([]*Message(nil), c.inflight...)
c.inflightMu.Unlock()

for _, msg := range pending {
    select {
    case c.SendCh <- msg:        // safe — no lock held
    case <-ctx.Done():
        return
    }
}
```

A blocking send on a channel, while holding a lock the redelivery scan also wants, is deadlock bait. Take the snapshot under the lock, release it, *then* send. The symmetric rule on the ticker side: the scan **does** hold the lock for the full pass — releasing per-entry lets a concurrent `ACK` delete an entry mid-iteration and panic the map. `-race` finds the second one in five seconds; code review catches the first.

I tried to do structure + semantics in one branch first. I rolled it back at the second self-merge conflict. **The two-merge rule:** if a branch conflicts against itself twice before it's done, the branch is too big. Split it.

## Concurrency: serving many connections without corruption

The broker is one process; clients are many. The design question is *how many goroutines per connection, and who owns what*.

toymq's answer: **three goroutines per session, one channel of truth between them.** A fourth — the broker's per-subscription delivery worker — lives on the broker side but writes into this session's outbound channel.

![Per-session goroutines: Reader → Handler → Writer, with broker-side runDelivery feeding outbound channel](https://mermaid.ink/img/Zmxvd2NoYXJ0IFRECiAgICBOZXRbKCJuZXQuQ29ubjxici8-KGNsaWVudCBzb2NrZXQpIildCgogICAgc3ViZ3JhcGggU2Vzc2lvblsiT25lIFNlc3Npb24iXQogICAgICAgIGRpcmVjdGlvbiBUQgogICAgICAgIFJbIlNlc3Npb24gUjxici8-cmVhZHMgYnl0ZXMsIHBhcnNlcyBDb21tYW5kIl0KICAgICAgICBIWyJTZXNzaW9uIEg8YnIvPnJ1bnMgYnJva2VyIGNhbGxzLCBidWlsZHMgcmVzcG9uc2UiXQogICAgICAgIFdbIlNlc3Npb24gVzxici8-YnVmZmVycyArIGZsdXNoZXMgbmV0LkNvbm4iXQogICAgZW5kCgogICAgc3ViZ3JhcGggQnJva2VyWyJCcm9rZXIgc2lkZSJdCiAgICAgICAgRFsicnVuRGVsaXZlcnk8YnIvPihvbmUgcGVyIHN1YnNjcmlwdGlvbik8YnIvPldBTCBSZWFkZXIg4oaSIE1TRyBmcmFtZXMiXQogICAgZW5kCgogICAgTmV0IC0tPnxieXRlcyBpbnwgUgogICAgUiAtLT58ImluYm91bmQgY2hhbiBDb21tYW5kInwgSAogICAgSCAtLT58Im91dGJvdW5kIGNoYW4gcmVzcG9uc2UifCBXCiAgICBEIC0tPnwib3V0Ym91bmQgY2hhbiByZXNwb25zZSJ8IFcKICAgIFcgLS0-fGJ5dGVzIG91dHwgTmV0?type=png)

- **Session R** owns the socket's `bufio.Reader`. Nothing else reads.
- **Session W** owns the socket's `bufio.Writer`. Nothing else writes.
- **Session H** dispatches parsed commands to the broker and feeds responses into `outbound`.
- **`runDelivery`** is the broker's delivery goroutine for *this* consumer; it also writes into the session's `outbound` channel.

`outbound` is the only place where two goroutines (H and runDelivery) might race to write the socket — and the channel itself serializes them, no lock needed. `bufio.Writer` has no concurrent-safe contract, and you don't lock around it; you funnel through a channel. Closing the socket, draining `outbound`, and joining all goroutines becomes one operation: cancel a context, wait on a WaitGroup.

The rule that came out of this is sharp and reusable: **every `go`-spawned goroutine must be accounted for in exactly one `WaitGroup` or `doneCh`.** A "WaitGroup race" in the listener — `wg.Wait` returning before a goroutine that called `wg.Add` had a chance to register — was the bug that taught me to write the rule down explicitly.

The TCP accept loop adds two subtleties most servers get wrong: an `EMFILE`-aware exponential backoff (a burst of connections can exhaust file descriptors and tight-loop the broker), and an explicit `listener.Close()` on context cancel (without it, `Accept()` blocks forever and shutdown hangs).

`cmd/toymq` keeps `main.go` thin: a separate `internal/config` package owns flag validation, a testable `run(ctx, args, stdout, stderr) int` lets tests drive shutdown via context cancellation, and `signal.NotifyContext` wires `SIGTERM`/`SIGINT` into the context in one line. The smoke test boots `run`, sends a fake `SIGTERM`, and asserts a clean exit. It catches "shutdown leaks a goroutine" the moment it happens.

## Testing what unit tests can't see

By this point every package had unit tests. Coverage was 78%. `-race` clean. But the most interesting bugs in any networked system live in the *seams* — what happens when the broker sends a 300-byte `MSG`, the client receives bytes 1..200, and then disconnects?

I built in-process integration tests next: a real broker + server on a kernel-assigned TCP port, a stripped-down test client that queues incoming `MSG`s so asynchronous deliveries don't deadlock the response-reading. Six scenarios: round-trip ACK, 1000-message restart, visibility-timeout redelivery, NACK redelivery, dedupe, subscribe takeover. Coverage went from 78% to 90.3%.

The suite caught the worst bug of the project on its first run. The test:

1. Publish message `id=0`. Subscribe. Receive MSG. ACK.
2. Restart the broker.
3. Subscribe again. **Expect no replay.**

It failed. Message `id=0` replayed. With `id=1` as the first message, no replay. The bug was specifically the zero msg-id.

Root cause: consumer state on disk stored a single `lastAcked uint64`. Recovery treated `lastAcked == 0` as "never acked" (the Go zero value) instead of "acked msg id 0." On the first restart after acking the very first message, the broker couldn't tell the difference. The fix adds a `hasAcked bool` and persists both:

```go
type ConsumerState struct {
    LastAcked uint64
    HasAcked  bool   // ← the field that should have been there from day 1
}
```

The Go-language lesson: **zero values are not sentinels.** They collide with valid data the moment your domain is non-empty. The systems lesson is sharper — **the bug existed for six commits before anyone noticed.** Unit tests never restarted the broker between Ack and Subscribe. By the time integration tests ran, the broken state was sealed in code-review history weeks earlier. The integration test caught it on its first run.

Then I added **chaos**: a supervisor that `SIGKILL`s the broker on schedule and restarts it; a producer that keeps publishing through restarts; a consumer that records every msg-id it ever sees. Invariant: for every msg-id the producer got an `OK` for, the consumer must receive it at least once, eventually. A 90-second smoke pushes 14,000 messages through three `SIGKILL` cycles with zero acked loss. That's not a proof of correctness — it's evidence the WAL + fsync + offset design holds up under realistic crash patterns.

Chaos also found a data race in the chaos *test itself* — a `bytes.Buffer` capturing stderr being written by the supervisor and read by the assertion. The broker was correct; the test had the race. **`-race` is for everything you ship, including tests.** A race in test infrastructure can mask real bugs or manufacture phantom ones.

## After v1.0: the part where the work gets easier

Once the bottom half held, the top half was craft, not danger. A Go client library (`pkg/client`) on top of the protocol, with a single read goroutine that demuxes incoming frames by type — separate channels per call would have raced two reads on the socket, a deadlock recipe. A CLI (`toymqctl`) and a latency harness (`toymq-bench`). A Bubble Tea TUI (`toymq-tui`) with one sharp lesson worth keeping: **boolean state for "is X active?" is a smell.** A `bool` for "is a modal open?" broke the day the second modal shipped on top of the first. A modal *stack* fixed it. Lists, stacks, enums — anything but a boolean — almost always model the actual shape.

Then Prometheus metrics and OpenTelemetry tracing, with the design rule: **observability does not change behavior.** No locks, no goroutines, no allocations on the hot path. `prometheus.CounterVec.WithLabelValues` is cached; `otel.Tracer().Start` is a no-op when no provider is attached.

Finally, CI hardening after `main` went red twice in a row with `gofmt` drift. Two recurrences is the line where you stop fixing instances and start fixing the system: a `Makefile`, an opt-in pre-commit hook, `golangci-lint` with a conservative ruleset, and a Go 1.25 + 1.26 CI matrix. The linter's first run flagged 31 things; seven were real bugs (`commited`, `exhuastion`, three dead helpers, one `cap`-shadowing parameter). A linter that flags 30 things on its first run is doing its job.

## ADRs as crystallization, not ceremony

Seventeen ADRs sounds like ceremony. It wasn't.

The rule: **write the ADR the moment the decision is forced by code, never before.** Not when I had a hunch. When I had to type "we are doing it this way" into a function, *then* the ADR went next to it.

ADRs written before code force the code to fit the ADR, which means defending decisions made when you knew the least. ADRs written *at* crystallization record what the code already decided. If the code later changes its mind, you supersede the old ADR — you don't try to retroactively justify it. The point is that the ADR is always honest about *when* the decision was made.

## What I'd do differently

- **Run chaos one branch earlier.** Integration tests didn't find what chaos found one branch later. Chaos was the right tool, just deployed one branch too late.
- **CI matrix from day one.** A Go 1.25 + 1.26 matrix takes 30 minutes to set up and prevents an entire class of "works on my machine" failure. It should have been part of the first `chore(ci)` commit, not a retrofit at the end.
- **Wire trace context through the protocol from v1.3.** Spans today are root spans inside the broker. Cross-process correlation needs a `TRACEPARENT` line in the wire format. Wire-format changes are exactly what you regret deferring.
- **Treat zero values as a code smell from day one.** The `lastAcked` bug was the most embarrassing of the project. Whenever the answer is "use 0 / `""` / `nil` / `-1` to mean absent," the right answer is `(value, present bool)`.

## What's next

**Persisting the dedupe LRU to disk without losing the boring-by-design property of the WAL.** Right now the LRU is in memory only — survives one broker lifetime, lost on `SIGKILL`. The fix mirrors the atomic-swap pattern used for offsets; the interesting question is what happens on a torn write of the LRU snapshot itself, since unlike the WAL there's no CRC-framed record stream to scan. That's the next post.

After that, the series moves to **tinykv** (a Redis-subset KV store in Go) and **tinyraft** (a 3-node Raft consensus cluster). Same risk-first sequencing. Same ADRs-as-crystallization. The arc ends where you compose them: replicated state machine semantics on top of a real consensus log.

## The one-line version

If a tutorial tells you to build the server first, you've already started in the wrong place. **The storage is the project.**

---

*Source: [`prajwalmahajan101/toymq`](https://github.com/prajwalmahajan101/toymq). Deeper docs (with every diagram in this post and more): [`ARCHITECTURE.md`](https://github.com/prajwalmahajan101/toymq/blob/main/docs/ARCHITECTURE.md), [`PERSISTENCE.md`](https://github.com/prajwalmahajan101/toymq/blob/main/docs/PERSISTENCE.md), [`CONCURRENCY.md`](https://github.com/prajwalmahajan101/toymq/blob/main/docs/CONCURRENCY.md), [`REDELIVERY.md`](https://github.com/prajwalmahajan101/toymq/blob/main/docs/REDELIVERY.md), [`FLOWS.md`](https://github.com/prajwalmahajan101/toymq/blob/main/docs/FLOWS.md). ADR index: [`docs/adr/`](https://github.com/prajwalmahajan101/toymq/tree/main/docs/adr). Open ideas: [`IDEA.md`](https://github.com/prajwalmahajan101/toymq/blob/main/IDEA.md). Corrections welcome — open a discussion or file an issue.*
