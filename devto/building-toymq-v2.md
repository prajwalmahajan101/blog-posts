---
title: "Building toymq v2: eight features, one durability contract, and the bug that proved it held"
published: true
description: "How toymq went from a single-node WAL broker (v1) to a usable one (v2): auth/TLS, partitions, backpressure, retention, DLQ, delayed messages, correlated observability — eight milestones, 10 new ADRs, without ever betraying the durability contract v1 earned."
tags: go, distributedsystems, backend, architecture
series: "Building distributed systems"
canonical_url: https://dev.to/prajwalmahajan101/building-toymq-v2-eight-features-one-durability-contract-and-the-bug-that-proved-it-held-19ff

# dev.to metadata
# article_id: 4130948
# published_at: 2026-07-13T08:50:47Z
# url: https://dev.to/prajwalmahajan101/building-toymq-v2-eight-features-one-durability-contract-and-the-bug-that-proved-it-held-19ff
---

> A retrospective on the v1 → v2 arc of [`toymq`](https://github.com/prajwalmahajan101/toymq) — the jump from a single-node persistent message broker that *works* to one you can actually *deploy*. The first post was about building the dangerous parts first. This one is about adding features to a durable system without breaking the durability.

## TL;DR — the decision, and the bug, you should remember

[Part 1](https://dev.to/prajwalmahajan101/building-toymq-a-from-scratch-persistent-message-broker-in-go-ob7) built the bottom of the stack first: WAL → protocol → broker. The thesis was *build the thing that, if it's wrong, forces a rewrite of everything above it.* v1.0 shipped a broker that survives `SIGKILL` with zero acked-message loss.

v2 is a different problem. The dangerous part is already built and *correct*. The job now is to bolt on eight features people actually need — auth, TLS, partitions, backpressure, retention, DLQs, delayed messages, trace correlation — **without regressing the one property v1 spent four days earning.** Half of them touch the WAL or the wire format, and any one can quietly reintroduce a duplicate, a lost ack, or a torn recovery.

The decision to remember is the smallest one. v2 M1 needed the dedupe index to survive a restart. The obvious move — the move the roadmap originally wrote down — was a sidecar `dedupe.json`, debounced and atomically swapped, exactly like the offsets file already does. **I threw it out before writing a line of it** (the rebuild landed in [`e943980`](https://github.com/prajwalmahajan101/toymq/commit/e943980)). A sidecar is a *lossy cache* of a truth the WAL already holds: it lags by its debounce window, so a crash between a dedupe insert and the next flush drops the key and re-materialises the exact duplicate it was supposed to prevent. The right answer was to rebuild the index from the WAL on `Open` — the scan already visits every record, via a `WithRecoveryVisitor` hook the broker funnels each recovered record through. Zero new files, zero new fsync path, zero gap.

That collapses to a one-liner that belongs to a family. The other posts in this series each earned one the same shape:

| Post | The hero rule | The trap it names |
|---|---|---|
| [`toymq` v1](https://dev.to/prajwalmahajan101/building-toymq-a-from-scratch-persistent-message-broker-in-go-ob7) | **zero values are not sentinels** | `lastAcked == 0` meant both "acked id 0" and "never acked" |
| [`toykv`](https://dev.to/prajwalmahajan101/building-toykv-a-from-scratch-persistent-kv-in-go-and-why-i-took-the-opposite-call-from-toymq-5862) | **durations are not deadlines** | `EX 5` on disk re-applies against `now` after every restart |
| **`toymq` v2 (this post)** | **derived state is not a source of truth** | a `dedupe.json` sidecar re-materialises the duplicate it was meant to kill |

> **The rule:** when you're about to persist derived state, ask whether the source of truth already persists it. Usually it does, and the sidecar is a second consistency problem wearing a helper's clothes.

All three are the same bug wearing different clothes: **a value that looks authoritative until the system restarts in a state it didn't anticipate.** v1 built the dangerous part first; toykv took the *opposite* call three times on a sibling project. This post turns the axis again — same project, one year of design decisions later — and the question is no longer "how do I get durability right?" but "how do I keep it while everything around it changes?"

And the bug to remember is the one where I almost didn't. In M6, two features I'd built and tested *separately* — segment retention and delayed messages — turned out to quietly corrupt each other: the retention sweeper would evict an old WAL segment that still held a `DELAY`ed message scheduled to fire in the future, silently deleting an *acked* publish forty minutes before it was due. Not a crash. Not a race. A message lost to a **feature.** That's the whole thesis in one bug: once you've earned a durability contract, the thing most likely to break it isn't hardware — it's the next feature. The full story is in the M6 section; the rule it earned runs through the whole post.

The numbers: ~17.6k lines of Go (up from ~10k), 27 ADRs (up from 17), 53 test modules, four binaries, 110 commits across eight milestones (M1–M8), two deliberate wire breaks clustered behind one major bump, and a cross-product integration matrix — `{per-message, batched} × {plain, TLS} × {auth, no-auth} × {1, 4 partitions}` — that has to stay green before the tag moves.

The order:

```
M1 dedupe-persist → M2 batched-fsync → M3 HELLO/auth/TLS → M4 partitions
   → M5 backpressure → M6 retention/DLQ/delay → M7 correlated telemetry
   → M7.5 Grafana stack → M8 integration matrix + tag v2.0.0
```

The point of this post, like the last one, is the order and the rules — not the code.

![v2 milestone ordering: correctness gaps (M1–M2) → lift the deploy ceiling (M3) → scale on top (M4–M6) → see it run (M7–M7.5) → tag (M8); the two red WIRE BREAK nodes are clustered behind one major bump](https://mermaid.ink/img/Zmxvd2NoYXJ0IFRCCiAgICBzdWJncmFwaCBDT1JSWyJDb3JyZWN0bmVzcyBnYXBzIGZpcnN0Il0KICAgICAgICBNMVtNMSBkZWR1cGUgcGVyc2lzdDxici8+cmVidWlsZCBmcm9tIFdBTF0KICAgICAgICBNMltNMiBiYXRjaGVkIGZzeW5jPGJyLz5ncm91cCBjb21taXRdCiAgICBlbmQKICAgIHN1YmdyYXBoIERFUExPWVsiTGlmdCB0aGUgZGVwbG95IGNlaWxpbmciXQogICAgICAgIE0zW00zIEhFTExPICsgYXV0aCArIFRMUzxici8+V0lSRSBCUkVBS10KICAgIGVuZAogICAgc3ViZ3JhcGggU0NBTEVbIlNjYWxlIG9uIHRvcCBvZiBpdCJdCiAgICAgICAgTTRbTTQgcGFydGl0aW9uczxici8+V0lSRSBCUkVBS10KICAgICAgICBNNVtNNSBiYWNrcHJlc3N1cmU8YnIvPmFkZGl0aXZlXQogICAgICAgIE02W002IHJldGVudGlvbiAvIERMUSAvIGRlbGF5PGJyLz5hZGRpdGl2ZV0KICAgIGVuZAogICAgc3ViZ3JhcGggU0VFWyJTZWUgaXQgcnVuIl0KICAgICAgICBNN1tNNyB0cmFjZXBhcmVudCArIGNvcnJlbGF0aW9uPGJyLz5hZGRpdGl2ZV0KICAgICAgICBNNzVbTTcuNSBHcmFmYW5hIExHVE1dCiAgICBlbmQKICAgIE04W004IDE2LWNlbGwgbWF0cml4PGJyLz50YWcgdjIuMC4wXQogICAgTTEgLS0+IE0yIC0tPiBNMyAtLT4gTTQgLS0+IE01IC0tPiBNNiAtLT4gTTcgLS0+IE03NSAtLT4gTTgKICAgIHN0eWxlIENPUlIgZmlsbDojZGJlYWZlLHN0cm9rZTojMWQ0ZWQ4CiAgICBzdHlsZSBERVBMT1kgZmlsbDojZmVlMmUyLHN0cm9rZTojZGMyNjI2CiAgICBzdHlsZSBTQ0FMRSBmaWxsOiNmZWYzYzcsc3Ryb2tlOiNkOTc3MDYKICAgIHN0eWxlIFNFRSBmaWxsOiNmM2U4ZmYsc3Ryb2tlOiM3YzNhZWQKICAgIHN0eWxlIE04IGZpbGw6I2RjZmNlNyxzdHJva2U6IzE2YTM0YQo=?type=png)

In v1 the rule was "risk sinks *down* — build the bottom first." This diagram is that rule rotated ninety degrees: with the bottom already frozen, risk drains *left-to-right*, and the two red WIRE BREAK nodes sit adjacent on purpose so the one major bump pays for both.

## Why the order is the design

v1's ordering rule was "risk sinks down; build the bottom first." v2's is subtler, because everything sits *on top* of a bottom that's already frozen. Three principles did the work.

**Correctness gaps land before performance wins.** M1 (dedupe persistence) closes a durability hole; M2 (batched fsync) is a throughput win. M1 ships first, because a performance win layered on a correctness gap just loses data faster. Spend the correctness budget before the speed budget.

**Lift the deployment ceiling before building what needs it lifted.** M3 (auth + TLS) makes the broker safely reachable off-host; M4 (partitions) scales it. Partitions before auth would ship a multi-partition broker nobody can run off localhost — scaling a thing you can't deploy.

**Cluster the wire breaks behind one major bump.** v2 has exactly two: the HELLO handshake (M3) and the partition arity on `PUB`/`MSG`/`ACK`/`NACK` (M4). Everything else — batched fsync, `PAUSE`/`RESUME`, retention, DLQ, `DELAY`, `TRACEPARENT` — is strictly additive, so a pre-v2 client is processed byte-for-byte as before.

> **The rule:** you get one wire break per major version. Spend it on everything at once, gated behind the same bump.

Each milestone also owns its own crash/concurrency/wire-compat tests. There is no catch-all "integration pass at the end" that discovers M3's bug during M7's work. The risk test lives in the milestone that introduces the risk.

## Five rules v2 earned

Like part 1, I didn't write these up front. I extracted them by reading the ADRs and asking *what did I refuse to compromise on?*

### 1. The durability contract is sacred. A feature negotiates around it, never through it.

M2's batched fsync coalesces many appends into one `fsync`. The temptation is to let `committed` advance when bytes hit the write buffer. It doesn't — **`committed` still advances only *after* the fsync returns**, exactly as in v1. Batching changes *when* the fsync happens, never *what an `OK` promises*. The one genuine weakening, `--fsync=none` (page-cache only), is opt-in, never the default, and documented loudly. If a feature has to weaken durability, make the weakening explicit and impossible to hit by accident. Retention (M6) is the same rule from the other side — the M6 war story below — where a sweeper almost deleted an acked message a feature had promised to deliver.

### 2. Every feature is designed around the seam the distributed future needs.

This is what makes v2 more than a feature dump. v3 is `toyraft` — multi-node replication where the **Raft log is the source of truth** and each node rebuilds local state by `Apply`-ing committed entries. That model reached back and shaped v2's single-node code:

- **Dedupe rebuild-from-WAL (M1)** is *exactly* the `raft.StateMachine.Restore` seam v3 needs — a follower must rebuild dedupe from the log, never from a per-node file Raft never replicated. The sidecar would have been torn out; the WAL-rebuild seam is reused verbatim.
- **A partition (M4) is the unit v3 wraps a Raft group around.** `Topic` became a thin router over a `Partition` type that owns the WAL, dedupe, consumers, and offsets — not only for single-node parallelism, but so v3 gets one Raft group per `(topic, partition)` without re-architecting.
- **The determinism debt is written down where it'll be paid.** `topic.go`'s `time.Now()` and local `MsgID` counter must move to the *proposer* under Raft (where `Apply` must be byte-identical on every node) — recorded as v3 M1's first task, not silently deferred.

The rule: when you know where the project is going, let the destination break ties between designs that look equal today. The one that becomes a seam beats the one that becomes a rewrite.

### 3. Breaking an old client must fail safe or opt in — never silently misbehave.

Both of v2's wire breaks have an escape hatch that's a *design* choice, not a courtesy: HELLO ships with a `--require-hello=false` migration window (M3), and 1-partition topics keep the pre-M4 on-disk layout byte-for-byte, so a v1 data directory recovers unchanged (M4). Both are detailed below. Additive features follow the same discipline — `TRACEPARENT` (M7) is an optional line, and a *malformed* one degrades to a fresh root span rather than erroring the connection. The parser stays dumb; the W3C spec stays the single source of truth.

### 4. Backpressure is about the slow half you aren't looking at.

The obvious bug and the real bug differ. Delivery *did* have backpressure — against a client that stops *reading* (the send channel and TCP buffer fill). The real gap is a client that reads *eagerly* and acks *slowly*: the buffers drain, delivery keeps producing, and inflight grows unbounded. The fix ties delivery rate to the **ack** rate, not socket readability (the credit window in M5 below). The rule generalises: when you add backpressure, find the consumer that's fast at the thing you're measuring and slow at the thing you aren't.

### 5. Observability that needs its own dependency in the hot path is the wrong observability.

v1's rule was "observability does not change behavior." v2 extends it: it doesn't get to change *dependencies* either. M7 needed W3C trace propagation on the wire, but `pkg/client` is deliberately stdlib-only — so rather than drag OpenTelemetry into every consumer's dependency closure to carry one header, propagation is a callback (`client.WithTraceparentFunc`) the caller wires to their own otel context. The header crosses the wire; the dependency doesn't cross the boundary.

---

## The dangerous parts of v2, one milestone at a time

### M3 — the handshake response that got dropped 40% of the time

The HELLO handshake looked trivial: read the first line, validate, reply `HELLO 1 OK` or `ERR AUTH` and close. The first design routed that reply through the session's normal response channel — the same channel the async writer goroutine drains. On a *rejection*, `close(quit)` raced the queued `ERR` in the writer's `select`, and the error lost about 40% of the time under load. The client got an EOF instead of a reason: "connection closed" instead of "bad token."

The fix is structural, not a lock. **Do the handshake synchronously, inline, before the async writer goroutine exists** ([`ca2b7cd`](https://github.com/prajwalmahajan101/toymq/commit/ca2b7cd)): write the response straight to the buffered writer — there's no channel to race and no `quit` to lose to, because the concurrency that caused the race hasn't been spawned yet. The client mirrors it. Auth itself is boring on purpose — bearer tokens compared with `crypto/subtle.ConstantTimeCompare`, never logged.

![HELLO handshake sequence: the session validates and writes the response directly to the socket while the async writer goroutine does not yet exist, so there is no respCh and no quit for the rejection to race](https://mermaid.ink/img/c2VxdWVuY2VEaWFncmFtCiAgICBhdXRvbnVtYmVyCiAgICBwYXJ0aWNpcGFudCBDIGFzIENsaWVudAogICAgcGFydGljaXBhbnQgU0ggYXMgU2Vzc2lvbiAoaGFuZHNoYWtlKQogICAgcGFydGljaXBhbnQgVyBhcyBhc3luYyBXcml0ZXIKICAgIE5vdGUgb3ZlciBTSDogd3JpdGVyIGdvcm91dGluZSBOT1Qgc3Bhd25lZCB5ZXQKICAgIEMtPj5TSDogSEVMTE8gMSBBVVRIIDx0b2tlbj4KICAgIFNILT4+U0g6IFBhcnNlSGVsbG8gKyBDb25zdGFudFRpbWVDb21wYXJlCiAgICBhbHQgYmFkIC8gbWlzc2luZyB0b2tlbgogICAgICAgIFNILS0+PkM6IEVSUiBBVVRIICAoZGlyZWN0IHdyaXRlLCB0aGVuIGNsb3NlKQogICAgICAgIE5vdGUgb3ZlciBTSDogbm8gcmVzcENoLCBubyBxdWl0IHRvIHJhY2UKICAgIGVsc2Ugb2sKICAgICAgICBTSC0tPj5DOiBIRUxMTyAxIE9LICAoZGlyZWN0IHdyaXRlKQogICAgICAgIFNILT4+VzogTk9XIHN0YXJ0IGFzeW5jIHdyaXRlcgogICAgICAgIE5vdGUgb3ZlciBXOiBhc3luYyB3cml0ZXIgb3ducyB0aGUgc29ja2V0CiAgICBlbmQK?type=png)

> **The rule:** a response-delivery race is best solved by not having the response-delivery machinery yet. If a message must not be lost, send it before you spawn the goroutines that could lose it.

### M4 — Topic becomes a router, and the flat layout survives

Partitions could have been a rewrite. They weren't, because the publish/deliver/ack/offset logic didn't change — it got *re-scoped*. Everything the old `Topic` owned (WAL, dedupe, consumers, `pubMu`, offsets) moved to a `Partition`; `Topic` became a thin round-robin router over `[]*Partition`. MsgID stays monotonic *per partition* — total order within a partition, none across — so a `(partition, msgID)` pair is the message identity now.

![Partition routing: PUB enters the Topic router; an explicit #n wins, else the routing key hashes fnv1a % N, else keyless publishes round-robin — each landing in one of N independent ordered logs with its own WAL, dedupe, offsets, and per-partition monotonic MsgID](https://mermaid.ink/img/Zmxvd2NoYXJ0IFRCCiAgICBQVUJbIlBVQiBvcmRlcnMgazEgLi4uIFsjbl0iXQogICAgc3ViZ3JhcGggVE9QSUNbIlRvcGljID0gdGhpbiByb3V0ZXIiXQogICAgICAgIFJLeyJleHBsaWNpdCAjbj8ifQogICAgICAgIEhBU0hbImZudjFhKHJvdXRpbmdLZXkpICUgTiJdCiAgICAgICAgUlJbInJvdW5kLXJvYmluIGN1cnNvciJdCiAgICBlbmQKICAgIHN1YmdyYXBoIFBBUlRTWyJOIGluZGVwZW5kZW50IG9yZGVyZWQgbG9ncyJdCiAgICAgICAgUDBbIlBhcnRpdGlvbiAwPGJyLz5XQUwgKyBkZWR1cGUgKyBvZmZzZXRzPGJyLz5Nc2dJRCBtb25vdG9uaWMiXQogICAgICAgIFAxWyJQYXJ0aXRpb24gMTxici8+V0FMICsgZGVkdXBlICsgb2Zmc2V0czxici8+TXNnSUQgbW9ub3RvbmljIl0KICAgICAgICBQMlsiUGFydGl0aW9uIG48YnIvPi4uLiJdCiAgICBlbmQKICAgIFBVQiAtLT4gUksKICAgIFJLIC0tIHllcyAtLT4gUDEKICAgIFJLIC0tICJubywga2V5ZWQiIC0tPiBIQVNIIC0tPiBQMAogICAgUksgLS0gIm5vLCBrZXlsZXNzIiAtLT4gUlIgLS0+IFAyCiAgICBzdHlsZSBUT1BJQyBmaWxsOiNmZWYzYzcsc3Ryb2tlOiNkOTc3MDYKICAgIHN0eWxlIFBBUlRTIGZpbGw6I2RiZWFmZSxzdHJva2U6IzFkNGVkOAo=?type=png)

Routing is layered: explicit `<topic>#<n>` wins; else the routing key hashes (`fnv1a % N`); else keyless publishes round-robin. Because `wal.Open` was already per-directory, the WAL needed *no change* — each partition points it at its own subdir. And back-compat falls out of one decision: `meta.json` is written *only* when N > 1, so its absence *is* the "flat, pre-M4 layout" signal. The two shapes on disk:

```
data/topics/orders/          # N > 1: partitioned
├── meta.json                #   {"partitions": 4}   ← the only new artifact
├── 0/ 000000.log  offsets.json
├── 1/ 000000.log  offsets.json
└── ...
data/topics/logs/            # N == 1 (or any pre-M4 dir): FLAT, unchanged
├── 000000.log               #   no meta.json — byte-for-byte the v1 layout
└── offsets.json
```

```go
// topicPartitionCount: meta.json when present, else 1 for a flat/legacy
// layout — so a pre-M4 data dir recovers unchanged as a 1-partition topic.
data, err := os.ReadFile(filepath.Join(topicDir, "meta.json"))
if err == nil {
    var m topicMeta
    _ = json.Unmarshal(data, &m)
    return m.Partitions, true, nil          // N > 1: partitioned
}
if os.IsNotExist(err) {
    // No meta.json: a flat/legacy 1-partition topic if the dir exists.
    if _, statErr := os.Stat(topicDir); statErr == nil {
        return 1, true, nil                 // byte-for-byte the old layout
    }
}
```

### M5 — a wake channel, because `sync.Cond` can't say no

The mechanism (rule 4's fix) has a sharp edge worth keeping. The credit gate must honour context cancellation — a subscription teardown has to unblock a goroutine parked waiting for credit. `sync.Cond` is the textbook "block until a condition changes" tool, but you can't cancel a `cond.Wait()` from a `select`/`ctx`. So the gate is a buffered-1 *coalescing* channel plus a re-check loop ([`022eaea`](https://github.com/prajwalmahajan101/toymq/commit/022eaea)), with the receive in a `select` alongside `ctx.Done()`:

```go
// awaitCredit blocks until this consumer may deliver one more message —
// not paused and fewer than window inflight — or ctx is cancelled.
func (c *Consumer) awaitCredit(ctx context.Context) bool {
    for {
        c.mu.Lock()
        ready := !c.paused && len(c.inflight) < c.window
        c.mu.Unlock()
        if ready {
            return true
        }
        select {
        case <-c.wake:      // Ack/RESUME nudged us — loop and re-check.
        case <-ctx.Done():  // subscription torn down — the exit sync.Cond can't do.
            return false
        }
    }
}
```

`PAUSE`/`RESUME` rode along as additive session-scoped verbs — with one honest caveat the flow-control ADR spells out: PAUSE is best-effort to within one in-flight message, because a goroutine already parked in the WAL read will still deliver the one record it was awaiting. Cooperative flow control, not a hard barrier — and said so.

> **The rule:** `sync.Cond` blocks but can't be cancelled. When a blocked goroutine has to honour `ctx`, reach for a coalescing channel and a re-check loop, not a condition variable.

The gate was right the first time; the *test* was the war story. My first "does PAUSE actually stop delivery?" test asserted "no message arrives while paused" — and hung for the full 600-second timeout. I went hunting for a deadlock in the gate. There wasn't one: the test never acked what it received, so the 100ms redelivery ticker re-pushed the same messages forever, and the drain loop's idle timer never got the quiet gap it fires on. The gate did exactly what I asked; the test asked a question that could never come true. The fix set the window to 1 so the goroutine parks in the gate (zero PAUSE slip) and the drain gets its silence. **A test that hangs isn't always a deadlock in the code — sometimes it's a liveness bug in the test's own premise**, and the redelivery ticker I forgot to account for was the tell.

### M6 — three features that had to not corrupt each other

Retention, DLQ, and delayed messages shipped as three stacked PRs, and their interactions were the whole game.

- **Retention** turns the single WAL file into rolling numbered segments and drops whole sealed segments past a size/duration bound (the active segment is never touched). A resuming consumer whose offset fell below the retained floor gets an explicit `ERR OUT_OF_RANGE` — not a silent skip, not a crash. Segment boundaries are also the natural v3/toyraft snapshot reference (rule 2 again).
- **DLQ** republishes a message that fails N times onto an auto-created `<topic>.dlq`, via a deterministic `dlqMove` seam. `.dlq` topics are loop-guarded so a poison message can't ping-pong. In v3 the *leader* proposes the move; the seam is already shaped for it.
- **Delayed messages** stamp an append-only `VisibleAtNs` at propose time; the delivery goroutine parks until it fires, preserving per-partition order. It's persisted across restart *for free* because it lives in the WAL — and, per rule 1, retention learned not to evict a segment holding an un-fired delayed record.

That last clause is the near-miss from the intro ([`8d0c9dd`](https://github.com/prajwalmahajan101/toymq/commit/8d0c9dd)). Retention drops the *oldest* sealed segments — and a delayed message scheduled for the future lives in an *old* segment (published long ago, fires later). Composed naïvely, the sweeper evicts a message the broker had *promised* to deliver. The fix makes the evictable floor the minimum of the retention bound *and* the oldest un-fired `VisibleAtNs`. Two features that each looked correct alone would have corrupted each other; the invariant — retention respects delivery — is the thing that had to be designed, not either feature.

### M7 — the continuation that the parser refused

The plan was full producer→consumer trace stitching. It hit a wall that was correct to exist. Continuing the trace onto the delivered `MSG` frame meant a sixth token on a line the client's parser requires to be exactly five fields — an instant break for every pre-M7 client. And *genuine* stitching across a durable log needs the producer's trace context **persisted in the WAL record** (delivery can be minutes later, or post-restart) — a record-format change with all the determinism weight of the dedupe-recovery decision. So I scoped the continuation *out*, shipped the tested high-value half (producer→broker), and recorded the deferral in the traceparent ADR. Sometimes the right move is to defer a wire change *loudly*, reason on record, rather than force a break to look complete.

M7 also left a footgun I'll name so the next person doesn't rediscover it: log↔trace correlation only fires on `slog.*Context` calls, because the handler reads the span off the record's context. A bare `slog.Info` in a traced path silently drops the `trace_id`. There's a test that asserts the join to catch regressions, but the real fix would be a lint rule: *traced paths log with `*Context`.*

---

## Proof it composes

Rules and war stories are claims. Three artifacts back them.

**The M8 matrix — every feature crossed with every other.** The v2 features aren't independent: batched fsync must survive TLS framing, partitions must fan in under auth. So M8's owned risk test is the full cross-product `{per-message, batched} × {plain, TLS} × {auth, no-auth} × {1, 4 partitions}` — 16 cells, each fanning keyless publishes back in over `<topic>#*` and asserting exactly-once delivery plus per-partition MsgID monotonicity. Verbatim, under the race detector:

```
$ go test ./internal/integration/ -run TestM8CrossProductMatrix -race -v
=== RUN   TestM8CrossProductMatrix/per-message/plain/no-auth/p1
=== RUN   TestM8CrossProductMatrix/per-message/tls/auth/p4
=== RUN   TestM8CrossProductMatrix/batched/plain/no-auth/p4
=== RUN   TestM8CrossProductMatrix/batched/tls/auth/p1
    ... (16 cells: 2×2×2×2)
--- PASS: TestM8CrossProductMatrix (0.00s)
ok      github.com/prajwalmahajan101/toymq/internal/integration   1.185s
```

**The durability cost is real and honest.** Per-message durability is *measured*, not hand-waved. On commodity NVMe, `toymq-bench` with 8 producers pushing 20k×128-byte messages under the default `--fsync=per-message`:

```
$ toymq-bench --producers 8 --msgs 20000 --size 128 --fsync per-message
throughput  45162.6 msg/s   5.51 MiB/s
latency     min=27µs  p50=149µs  p95=349µs  p99=593µs  max=2.2ms
```

Every one of those 45k/s publishes crossed an `fsync` before its `OK` — the rule-1 contract, priced. `--partitions 4` holds the aggregate (~11.4k/s per partition) because each partition is an independent log. The number that matters isn't the throughput; it's that it's *honest about what it bought.*

**And the contract holds in the new fsync mode under crash.** The chaos harness runs the SIGKILL soak in `batched` mode too — a producer publishing through three `SIGKILL`/restart cycles, a consumer recording every id, the invariant being that every acked publish is eventually delivered:

```
$ CHAOS_FSYNC=batched CHAOS_DURATION=100s go test -tags chaos ./test/chaos/ -v
chaos: consumer uniqueSeen=10560 totalDeliveries=10584 errors=6 resubs=4
chaos: supervisor restarts=3
--- PASS: TestChaosSurvivesSIGKILL (100.26s)
```

**3 restarts, zero acked-message loss.** `totalDeliveries` exceeds `uniqueSeen` by exactly the 24 at-least-once duplicates the contract *permits*; the 6 `errors` are publishes that never got an `OK` because the broker was dead mid-kill, so they were never owed durability. Same guarantee v1 proved with per-message fsync — now with group commit on. The feature negotiated around the contract; it didn't go through it.

---

## ADRs as crystallization — the supersede trail

Both earlier posts landed on the same ADR discipline: write it the moment the decision is *forced by code*, so it records what the code already decided. v2 adds the wrinkle only a version-2 project can teach. v1's ADRs recorded *first* decisions; most of v2's *retire* an earlier one — batched fsync supersedes per-message-only, partitions supersede the single-log frame, traceparent supersedes the "no trace context" limitation. The temptation with each was to open the old ADR and quietly *edit* it, making past-me look like he'd planned for this. I didn't: the old ADR stays untouched, and the *new* one carries a `Supersedes` header explaining what changed and why now. The log stays a truthful timeline of when I changed my mind, not a re-touched photo.

That matters for the reason the durability contract matters. The contract is sacred; the *decisions around it* evolve — but only in the open. A superseded ADR has a visible history; an overwritten one pretends it never had an alternative. When v3 asks "why rebuild dedupe from the WAL, not a sidecar?", the answer isn't folklore — it's an ADR with the sidecar sketch it beat still legible inside it.

> **The rule:** supersede, don't overwrite. An ADR you edited to look right is a decision with no history — and a decision with no history is indistinguishable from luck.

## What I'd do differently

- **The MSG-continuation dead-end was checkable at design time.** One `grep` at the client's MSG parser would have shown the fixed arity before the spec promised a continuation. Check the strictest consumer of a format before you promise to extend it.
- **The `*Context`-only correlation constraint wants a lint, not a test.** A test catches the regression after you write it; a lint stops you writing it. Silent-miss footguns deserve mechanical enforcement.
- **The PAUSE/redelivery interaction should have been in the brainstorm.** Both flaky M5 test drafts trace to not accounting for the redelivery ticker up front. "How does this feature interact with redelivery?" belonged in the design gate.
- **Stand the full LGTM stack up before calling M7.5 done.** The configs are statically validated (`docker compose config`, `promtool check rules`, a metric cross-check) — strong, but a live `up --build` click-through is the difference between "should pivot" and "does pivot." Static validation is necessary, not sufficient, for a UI.

## What's next

v2 makes toymq *useful* single-node. v3 is where it earns the "distributed broker" framing, gated on a real dependency — `toyraft` — existing and being embeddable. The roadmap is honest that three of `toyraft`'s capabilities (snapshots/compaction, membership changes, ReadIndex reads) are stubbed today, so parts of v3 are *upstream-blocked* until they land.

And it keeps one decision open: **A** — stop at v1.x; **B** — v1 → v2, usable single-node, stop here; **C** — go distributed, *only* if toyraft is real. The default is A, reviewed after each major tag. Shipping v2 doesn't commit me to v3 — it just makes v3 buildable if the dependency shows up. That's the whole point of designing every v2 seam around where v3 would need it: the option stays cheap to exercise and cheap to decline.

## The one-line version

The series, in three rules that are the same rule: v1 — *zero values are not sentinels.* toykv — *durations are not deadlines.* v2 — *derived state is not a source of truth.* Each is a value that lied about being authoritative until a restart called its bluff. Part 1 said *the storage is the project.* Part 2 says **the durability contract is the project — and once it's earned, every feature negotiates around it, never through it.**

---

*Source: [`prajwalmahajan101/toymq`](https://github.com/prajwalmahajan101/toymq). The v2 arc in detail:* [`ROADMAP.md`](https://github.com/prajwalmahajan101/toymq/blob/main/docs/ROADMAP.md), [`CHANGELOG.md`](https://github.com/prajwalmahajan101/toymq/blob/main/CHANGELOG.md), and the ten v2 ADRs (0018–0027) in [`docs/adr/`](https://github.com/prajwalmahajan101/toymq/tree/main/docs/adr) — each records one decision above, with the alternatives it beat and the ADR it supersedes. Part 1: [Building toymq](https://dev.to/prajwalmahajan101/building-toymq-a-from-scratch-persistent-message-broker-in-go-ob7).

*Companion posts in this series: [Building toymq (part 1)](https://dev.to/prajwalmahajan101/building-toymq-a-from-scratch-persistent-message-broker-in-go-ob7) — the durable core this one builds on — and [Building toykv](https://dev.to/prajwalmahajan101/building-toykv-a-from-scratch-persistent-kv-in-go-and-why-i-took-the-opposite-call-from-toymq-5862) — the sibling project that took the opposite call three times.*
