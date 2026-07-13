---
title: "Building toykv: a from-scratch persistent KV in Go, and why I took the opposite call from toymq three times"
published: true
description: "I shipped two server-shaped Go projects in adjacent weeks and answered three load-bearing questions in opposite directions. The crash matrix is what proves the answers were right — and one rule, 'absolute deadlines, never relative durations,' is the toykv-shaped sibling of toymq's zero-value-sentinel bug."
tags: go, distributedsystems, backend, architecture
series: "Building distributed systems"
canonical_url: https://dev.to/prajwalmahajan101/building-toykv-a-from-scratch-persistent-kv-in-go-and-why-i-took-the-opposite-call-from-toymq-5862

# dev.to metadata
# article_id: 3925472
# published_at: 2026-06-17T16:39:53Z
# url: https://dev.to/prajwalmahajan101/building-toykv-a-from-scratch-persistent-kv-in-go-and-why-i-took-the-opposite-call-from-toymq-5862
---

> A retrospective on [`toykv`](https://github.com/prajwalmahajan101/toykv) — a single-node, in-memory key-value store with append-only-file persistence, RESP2-compatible on the wire, three binaries, and a v1.0.0 I shipped after about four weeks of work.

## TL;DR — and the rule I want you to remember

Some rules generalise across systems by inversion. [`toymq`](https://github.com/prajwalmahajan101/toymq) taught me **zero values are not sentinels**. [`toykv`](https://github.com/prajwalmahajan101/toykv) taught me **durations are not deadlines**. Same shape of bug, opposite axis — and along the way I ended up answering three load-bearing questions in the exact opposite direction from `toymq`:

- I shipped `toymq` with **no version byte** on its WAL. `toykv`'s AOF starts with one.
- I gave `toymq` its **own wire protocol** and a separate WAL record frame. In `toykv` I write the same RESP arrays I speak on the wire straight into the AOF.
- `toymq` had its own **delivery semantics** to defend. For `toykv` I chose RESP2 specifically so `redis-cli` and `go-redis/v9` become free third-party test harnesses.

The proof those opposite answers were right is the **crash matrix** — nine rows, each pinned by a test file that exists today, each test owned by the part of the system that introduced the risk. Every row is cheap because of one of those three decisions.

The hero rule of `toykv` — sibling of `toymq`'s `lastAcked uint64 == 0` zero-value-sentinel bug, reached from the other direction — is one line:

> **Absolute deadlines, never relative durations.**

Store `EX 5` on the wire; write `PEXPIREAT 1718659200000` to disk. `toymq` taught me that zero values are not sentinels. `toykv` taught me that durations are not deadlines. Same shape of bug, opposite axis.

The numbers: ~5k LOC of Go, 6 ADRs (I budgeted four), 15 journal entries, three binaries (`toykv`, `toykv-cli`, `toykv-tui`), v1.0.0 tagged on 2026-06-17.

## What toykv is, and isn't

A learning artifact. Single-node, in-memory, no auth, no TLS, strings only. 18 commands from the Redis surface — `GET`, `SET` (with `NX`/`XX`/`EX`/`PX`/`EXAT`/`PXAT`), `DEL`, `INCR`/`DECR`, `KEYS`, `FLUSHDB`, `EXPIRE`/`TTL`/`PEXPIRE`/`PEXPIREAT`/`PERSIST`, `BGREWRITEAOF`, and a handful of metadata commands. Under `appendfsync=always` (the default), throughput tops out in the low thousands of SETs per second on commodity NVMe — a per-write `fsync` is the durability commit point and the cost is honest. If you need a real KV store, reach for Redis or Valkey. If you want to *understand* what "durable" means in one, build one.

## The crash matrix

I lay it out the way I read it: every row of fault surface has exactly one test file, and the test lives next to the layer that introduced the risk. Composed-fault tests sit on their own at the bottom because composing faults is a different job from proving any one of them.

| Fault surface | Risk | Test file | Invariant proven |
|---|---|---|---|
| AOF append + replay | acked SET / DEL lost on SIGKILL under `fsync=always` | `internal/server/aof_crash_test.go` | Every record acknowledged before the kill replays on restart |
| Partial-tail handling | crash mid-record corrupts replay state | `internal/aof/replayer_test.go` | Replay rejects a torn record; offset reported, server refuses to serve until truncated |
| TTL lock-upgrade race | sweeper drops an unexpired key under load | `internal/store/sweeper_test.go` | N writers × 1 Hz sweeper → zero spurious `(nil)` for unexpired keys |
| TTL crash round-trip | TTL records lost across SIGKILL + restart | `internal/server/aof_ttl_crash_test.go` | v2 replay accepts v1 records; expiry decodes round-trip |
| `BGREWRITEAOF` during writes | rewrite races a concurrent SET; tail loses data | `internal/aof/rewriter_test.go` | Side-buffer captures all live appends; rewrite + tail merge with no loss |
| Crash during rewrite | partial `.aof.tmp` left, no canonical AOF survives | `internal/server/aof_rewrite_crash_test.go` | Exactly one of `{old, new}` present after restart; replay consistent |
| Shipped binary protocol drift | `redis-cli` / `go-redis` regression invisible to in-process tests | `test/e2e/protocol_*_test.go` | Byte-compat for every shipped command |
| `BGREWRITEAOF` + restart | restart after compaction loses data | `test/e2e/rewrite_restart_test.go` | Mixed-workload state survives rewrite + restart |
| **Composed faults (kill + pause + rewrite + writes)** | failure modes only visible when multiple faults overlap | `test/chaos/invariants_test.go` | Acked-SET survival, monotonic INCR, no panic across soak |

The protocol-compatibility layer (rows 7–8) deliberately introduces *zero* new crash invariants. It verifies that the bytes leaving the socket match the spec third-party clients expect; everything below the socket has already been proven elsewhere. The chaos suite (row 9) is the only layer that *composes* faults. Composed-fault tests as the primary durability proof are slow and flaky; composed-fault tests as a release-confidence soak after each component is independently proven are exactly what you want.

**The thesis I want you to take away:** every row is cheap because of one of the three decisions below. Where a crash matrix becomes expensive — where the test file is 800 lines, or two rows have to share state — that's usually the symptom of an architecture decision made elsewhere.

![Crash matrix layer flow: storage → TTL → live rewrite → protocol compatibility → composed chaos](https://mermaid.ink/img/Zmxvd2NoYXJ0IExSCiAgICBMMVsiU3RvcmFnZTxici8+QU9GIGFwcGVuZCArIHJlcGxheTxici8+cGFydGlhbC10YWlsIl0gLS0+IEwyWyJUVEw8YnIvPmxvY2stdXBncmFkZSByYWNlPGJyLz5jcmFzaCByb3VuZC10cmlwIl0KICAgIEwyIC0tPiBMM1siTGl2ZSByZXdyaXRlPGJyLz5kdWFsLXdyaXRlPGJyLz5jcmFzaCBkdXJpbmcgcmV3cml0ZSJdCiAgICBMMyAtLT4gTDRbIlByb3RvY29sIGNvbXBhdDxici8+Ynl0ZS1jb21wYXQ8YnIvPnJld3JpdGUgKyByZXN0YXJ0Il0KICAgIEw0IC0tPiBMNVsiQ29tcG9zZWQgY2hhb3M8YnIvPmtpbGwgKyBwYXVzZSArIHJld3JpdGUiXQogICAgc3R5bGUgTDEgZmlsbDojZGJlYWZlLHN0cm9rZTojMWQ0ZWQ4CiAgICBzdHlsZSBMMiBmaWxsOiNkYmVhZmUsc3Ryb2tlOiMxZDRlZDgKICAgIHN0eWxlIEwzIGZpbGw6I2RiZWFmZSxzdHJva2U6IzFkNGVkOAogICAgc3R5bGUgTDQgZmlsbDojZmVmM2M3LHN0cm9rZTojZDk3NzA2CiAgICBzdHlsZSBMNSBmaWxsOiNkY2ZjZTcsc3Ryb2tlOiMxNmEzNGE?type=png)

The first three layers introduce new fault surface, each proven in isolation by a dedicated crash test. The protocol layer verifies the shipped binary speaks the spec; it adds no new crash invariants. The chaos layer composes everything underneath into one soak. That shape is the proof the architecture is paying its rent.

## Inversion 1 — a version byte you'll thank yourself for, and absolute deadlines

`toymq`'s WAL has no version byte. The argument I wrote in that retrospective is good: defending the wrong byte forever is worse than the one-line migration of adding one later.

`toykv`'s AOF starts with eight bytes: `"TOYKV\x00\x00"` plus a one-byte version. The reasoning is also good, for a different shape of problem: **the AOF outlives the binary that wrote it.** If I run `toykv` on a tempdir on Monday and the same dir with a newer binary on Friday, the Friday binary has to know what assumption the Monday bytes were committed under. There is no person to ask. The byte is the answer.

I made the call before I wrote a line of replay code, and wrote it up as an ADR alongside the first AOF commit. It paid for itself inside four weeks: I needed a new on-disk shape for TTL, and the version byte let v2 binaries accept v1 files transparently. The replay loop is the same eight-line dispatch table; v2 just knows about a `PEXPIREAT` token v1 didn't.

> **The rule:** the cheaper migration is the one your future self can opt into, not the one their replay code is forced into. A version byte you don't use is a no-op. A version byte you didn't write is a migration script.

### Absolute deadlines, never relative durations

The companion ADR made the second call: every TTL command I accept on the wire rewrites to absolute `PEXPIREAT <unix-ms>` before it touches the disk. A wire `SET k v EX 5` becomes an AOF `SET k v` + `PEXPIREAT k <now-ms + 5000>`. The wire stays ergonomic; the disk stays unambiguous.

Imagine the alternative. Store `EX 5` verbatim in the AOF. Crash. Restart twelve hours later. Replay reads `EX 5` and applies it against `time.Now()` — every TTL in the file just got extended by twelve hours. Worse, the bug is silent: keys don't disappear, they linger. The test that catches this is a multi-day fixture, because the only way to notice is to wait past the original expiry. **That's the toykv-shaped sibling of toymq's `lastAcked == 0` sentinel collision** — a value that looks valid until the system restarts in a state it didn't anticipate. Same shape of bug, different axis.

> **The rule:** absolute deadlines, never relative durations. The wire can speak in human-friendly relative time; the disk has to speak in moments. A duration without a reference point isn't a deadline — it's a wish.

This is the rule I earned. Everything else in the post is the architecture I built to make the rule cheap to enforce.

![Wire-to-disk TTL canonicalisation: relative durations on the wire collapse through now_ms to absolute PEXPIREAT records on disk](https://mermaid.ink/img/Zmxvd2NoYXJ0IExSCiAgICBzdWJncmFwaCBXSVJFWyJXaXJlIOKAlCBlcmdvbm9taWMsIHJlbGF0aXZlIl0KICAgICAgICBXMVsiU0VUIGsgdiBFWCA1Il0KICAgICAgICBXMlsiRVhQSVJFIGsgMzAiXQogICAgICAgIFczWyJQRVhQSVJFIGsgMTUwMCJdCiAgICBlbmQKICAgIHN1YmdyYXBoIENBTk9OWyJDYW5vbmljYWxpc2VyIOKAlCBjb21wdXRlcyBub3coKSArIM6UIl0KICAgICAgICBDMVsibm93X21zID0gdGltZS5Ob3coKS5Vbml4TWlsbGkoKSJdCiAgICBlbmQKICAgIHN1YmdyYXBoIERJU0tbIkFPRiDigJQgYWJzb2x1dGUsIGRlY2lzaW9uLWZyZWUiXQogICAgICAgIEQxQVsiU0VUIGsgdiJdCiAgICAgICAgRDFCWyJQRVhQSVJFQVQgayAxNzE4NjU5MjAwMDAwIl0KICAgICAgICBEMlsiUEVYUElSRUFUIGsgMTcxODY1OTIyNTAwMCJdCiAgICAgICAgRDNbIlBFWFBJUkVBVCBrIDE3MTg2NTkyMjE1MDAiXQogICAgZW5kCiAgICBXMSAtLT4gQzEgLS0+IEQxQQogICAgQzEgLS0+IEQxQgogICAgVzIgLS0+IEMxIC0tPiBEMgogICAgVzMgLS0+IEMxIC0tPiBEMwogICAgc3R5bGUgV0lSRSBmaWxsOiNkYmVhZmUsc3Ryb2tlOiMxZDRlZDgKICAgIHN0eWxlIENBTk9OIGZpbGw6I2ZlZjNjNyxzdHJva2U6I2Q5NzcwNgogICAgc3R5bGUgRElTSyBmaWxsOiNkY2ZjZTcsc3Ryb2tlOiMxNmEzNGE?type=png)

Replay reads `PEXPIREAT k 1718659200000`. The math works regardless of when replay runs — a year later, the answer is "this key was already expired"; a second after the crash, "this key has 4.999 seconds left." There is no clock to argue with. The wire stays ergonomic; the disk stays unambiguous.

![AOF replay header + version dispatch: 8-byte magic + version, then loop over RESP records, dispatching through the live server table](https://mermaid.ink/img/c2VxdWVuY2VEaWFncmFtCiAgICBhdXRvbnVtYmVyCiAgICBwYXJ0aWNpcGFudCBTIGFzIFNlcnZlci5OZXcKICAgIHBhcnRpY2lwYW50IFIgYXMgYW9mLlJlcGxheWVyCiAgICBwYXJ0aWNpcGFudCBEIGFzIFNlcnZlci5kaXNwYXRjaAogICAgcGFydGljaXBhbnQgU1QgYXMgU3RvcmUKCiAgICBTLT4+UjogT3BlbihkaXIpCiAgICBSLT4+UjogcmVhZCA4LWJ5dGUgaGVhZGVyCiAgICBOb3RlIG92ZXIgUjogIlRPWUtWXDBcMCIgKyB2ZXJzaW9uIGJ5dGUKICAgIGFsdCBoZWFkZXIgbWlzc2luZwogICAgICAgIFItLT4+UzogaW8uRU9GIChmcmVzaCBkaXIsIG5vIEFPRiB5ZXQpCiAgICBlbHNlIHZlcnNpb24gdW5rbm93bgogICAgICAgIFItLT4+UzogRXJyVmVyc2lvblVuc3VwcG9ydGVkCiAgICBlbmQKICAgIGxvb3AgbmV4dCByZWNvcmQKICAgICAgICBSLT4+UjogcGFyc2UgUkVTUCBhcnJheQogICAgICAgIFItPj5EOiByZXBsYXlBcHBseShhcmd2KQogICAgICAgIE5vdGUgb3ZlciBEOiBzYW1lIGRpc3BhdGNoIGFzIGxpdmU8YnIvPmFvZiA9PSBuaWwsIG5vIHJlLWFwcGVuZAogICAgICAgIEQtPj5TVDogU0VUIC8gREVMIC8gU0VURVggLyBQRVhQSVJFQVQgLi4uCiAgICBlbmQKICAgIFItLT4+UzogYnl0ZXMsIHJlY29yZHMsIGR1cmF0aW9u?type=png)

### The AOF on disk

The file format is deliberately boring:

```
data/
└── appendonly.aof    ← header + RESP-encoded record stream
```

One file. No manifest, no index, no segment rotation in v1. The header is eight bytes; everything after is a stream of RESP arrays. The recovery scan walks the file from offset zero on `Open` — O(disk), but correct by construction. A manifest would be a second consistency problem on top of the first.

| Field      | Type   | Size      | Notes                                                  |
|------------|--------|-----------|--------------------------------------------------------|
| `magic`    | bytes  | 7         | Literal `"TOYKV\x00\x00"` — fails fast on a wrong file |
| `version`  | `u8`   | 1         | `0x01` (strings only) or `0x02` (adds `PEXPIREAT` for TTL) |
| `record_*` | RESP   | variable  | Same `*<n>\r\n$<len>\r\n…` frame the wire speaks       |

There is no per-record CRC. RESP framing already catches truncated bulks and mismatched array counts; adding a CRC for a learning artefact is belt-and-braces I deliberately deferred. The first real-world corruption that slips past RESP-level parsing is the moment to add one — not before.

## Inversion 2 — write the bytes you'd send

The journal entry that wrote itself, `docs/journal/02-aof.md`, has this line:

> *"The whole AOF format is RESP arrays on disk. `Append` calls the existing `resp.Writer` against a buffered file handle; `Replay` calls the existing `resp.Reader`. One codec, two consumers."*

That's the architectural claim. Everything I did downstream of it is mechanical. The replay path is **nine lines of Go**: skip the header, read a RESP array, dispatch it, repeat to EOF. During replay `s.aof == nil`, so the `appendIfLive` call inside each mutating handler is a silent no-op. **The same nine lines handle real traffic and crash recovery.**

The AOF-replay invariant — *every record acknowledged before the kill replays on restart* — collapses to a single ordering question: **does the `+OK` cross the network before or after `file.Sync()` returns?** Under `appendfsync=always`, the answer is *after*. The proof is operational: the crash test (`internal/server/aof_crash_test.go`) writes ~90 SETs against a self-re-exec child server, sends `SIGKILL` mid-stream, restarts, and checks every `+OK` against the post-replay state. `-count=10` after the first pass found zero flakes.

![Durability commit point: handler writes RESP into the AOF, calls fsync, only then returns +OK to the client](https://mermaid.ink/img/c2VxdWVuY2VEaWFncmFtCiAgICBhdXRvbnVtYmVyCiAgICBwYXJ0aWNpcGFudCBDIGFzIENsaWVudAogICAgcGFydGljaXBhbnQgSCBhcyBTZXJ2ZXIuaGFuZGxlcgogICAgcGFydGljaXBhbnQgU1QgYXMgU3RvcmUKICAgIHBhcnRpY2lwYW50IFcgYXMgYW9mLldyaXRlcgogICAgcGFydGljaXBhbnQgRlMgYXMgS2VybmVsL0ZTCgogICAgQy0+Pkg6IFNFVCBrIHYKICAgIEgtPj5TVDogc3RvcmUuU2V0KGssIHYpCiAgICBTVC0tPj5IOiBvawogICAgSC0+Plc6IEFwcGVuZChTRVQsIGssIHYpCiAgICBXLT4+VzogZW5jb2RlIFJFU1AgYXJyYXkKICAgIFctPj5GUzogZmlsZS5Xcml0ZShieXRlcykKICAgIE5vdGUgb3ZlciBXLEZTOiBwYWdlIGNhY2hlIG9ubHk8YnIvPnBvd2VyIGxvc3MgaGVyZSBsb3NlcyB0aGVtCiAgICBXLT4+RlM6IGZpbGUuU3luYygpICAtLSBmc3luYygyKQogICAgTm90ZSBvdmVyIFcsRlM6IGR1cmFiaWxpdHkgY29tbWl0IHBvaW50PGJyLz5ieXRlcyBvbiBzdGFibGUgc3RvcmFnZQogICAgRlMtLT4+Vzogb2sKICAgIFctLT4+SDogbmlsCiAgICBILS0+PkM6ICtPSw?type=png)

I drew this diagram in the same shape as the WAL diagram in the `toymq` retrospective on purpose. A reader who has seen both will notice that **the durability ordering is identical** — that's the thing the two projects agree on. The codec choice is what's different, and the codec choice is what made the replay path nine lines instead of a hundred.

> **The rule:** write the bytes you'd send. One codec is cheaper than two consistent codecs, and the second codec is where the corruption story you don't want lives.

## Inversion 3 — RESP because the harness is free

For `toymq` I invented a protocol. The defence is sound: the protocol is the wire, the wire is the contract, an invented wire is the moment a project decides it can defend its own boundaries.

For `toykv` I chose RESP2. **RESP2 is a testing strategy disguised as a feature.** The reason is the protocol-compatibility layer of the crash matrix: integration tests against the shipped binary using third-party clients — `redis-cli` for byte-level conformance, `go-redis/v9` for the API a real Go consumer would see. That layer shipped without a line of client code I wrote. The clients ship for free, by people who have never heard of `toykv`. They don't care about the implementation; they care about the wire. That's exactly the property I wanted.

The crash-matrix tie-in is the load-bearing one: **the protocol layer owns zero new crash invariants.** It owns *protocol* invariants — TTL `-2/-1` sentinels, `EXPIRE` against a missing key returning `0`, the byte-exact framing of `*3\r\n$3\r\nSET\r\n…` — but every command was already covered by a crash test in the layer that introduced it. The protocol layer is the validation that the bytes leaving the socket match the spec the third-party clients expect; everything below the socket has already been proven.

> **The rule:** protocol compatibility is a testing strategy disguised as a feature. Two independent implementations of the wire shake out the spec. No single-author project has that property by default — you have to inherit it deliberately.

## Honourable mention 1 — the dual-write `bufio` bug

The interesting bug of the project was *not* the rename-plus-dir-fsync-plus-fd-swap dance for `BGREWRITEAOF` that I'd been bracing for. The invariants I'd written up in the rewrite ADR worked the first time. What broke was the dual-write `Append` path inside the rewriter: `sideBuf.Len() == 0` after what looked like a successful write.

Root cause: `resp.Writer` wraps its target in a `bufio.Writer`. The live path constructs `resp.NewWriter(outerBufio)`, and `bufio.NewWriter(outerBufio)` is smart enough to short-circuit — it returns the existing `*bufio.Writer` rather than wrapping it again. One buffer, one flush. When `Rewriter.BeginRewrite` constructed a fresh mirror writer against a `*bytes.Buffer`, the argument was no longer a `*bufio.Writer`. So `bufio.NewWriter(&scratch)` created a brand new inner bufio layer. **Two buffers, two flush points.** Records were "written" but pinned in an inner bufio nothing else was going to drain before `DrainAndSwap` ran. Two-line fix in [`736bf63`](https://github.com/prajwalmahajan101/toykv/commit/736bf63).

> **The rule:** any write path with two sinks has two flush points; both must be ordered against the swap. A bufio layer is invisible — the same line of Go behaves differently depending on the static type of its argument. Never construct a fresh wrapper of a swappable type against a non-bufio target without immediately scheduling its flush.

![BGREWRITEAOF dual-write + atomic rename: live Appends fan out to canonical + side buffer during snapshot, then DrainAndSwap before rename](https://mermaid.ink/img/c2VxdWVuY2VEaWFncmFtCiAgICBhdXRvbnVtYmVyCiAgICBwYXJ0aWNpcGFudCBIIGFzIFNlcnZlci5oYW5kbGVyCiAgICBwYXJ0aWNpcGFudCBXIGFzIGFvZi5Xcml0ZXIKICAgIHBhcnRpY2lwYW50IFIgYXMgYW9mLlJld3JpdGVyCiAgICBwYXJ0aWNpcGFudCBUTVAgYXMgLmFvZi50bXAKICAgIHBhcnRpY2lwYW50IEFPRiBhcyAuYW9mIChjYW5vbmljYWwpCiAgICBwYXJ0aWNpcGFudCBGUyBhcyBLZXJuZWwvRlMKCiAgICBOb3RlIG92ZXIgVyxSOiBTbmFwc2hvdCArIGR1YWwtd3JpdGUKICAgIEgtPj5SOiBCR1JFV1JJVEVBT0YKICAgIFItPj5XOiBCZWdpblJld3JpdGUoc2lkZUJ1ZikKICAgIE5vdGUgb3ZlciBXOiBsaXZlIEFwcGVuZHMgbm93IGZhbiBvdXQ6PGJyLz5jYW5vbmljYWwgKG91dGVyQnVmaW8pICsgc2lkZSAoaW5uZXJCdWZpbykKICAgIFItPj5SOiBTdG9yZS5TbmFwc2hvdCgpIOKAlCByZW5kZXIgY2Fub25pY2FsIFJFU1AKICAgIFItPj5UTVA6IHdyaXRlIGhlYWRlciArIHNuYXBzaG90IGJ5dGVzCiAgICBILT4+VzogQXBwZW5kIFNFVCBrIHYgKGNvbmN1cnJlbnQgd2l0aCBzbmFwc2hvdCkKICAgIFctPj5BT0Y6IHdyaXRlIHRvIGNhbm9uaWNhbCAoZHVyYWJsZSBvbiBmc3luYykKICAgIFctPj5XOiB3cml0ZSB0byBzaWRlQnVmIHZpYSBpbm5lckJ1ZmlvPGJyLz4qKm11c3QgRmx1c2goKSDigJQgdGhlIGR1YWwtd3JpdGUgYnVnKioKCiAgICBOb3RlIG92ZXIgVyxGUzogRHJhaW4gKyBzd2FwCiAgICBSLT4+VzogRHJhaW5BbmRTd2FwKCkKICAgIFctPj5XOiBpbm5lckJ1ZmlvLkZsdXNoKCkKICAgIFctPj5UTVA6IGFwcGVuZCBzaWRlQnVmIGJ5dGVzCiAgICBSLT4+RlM6IHJlbmFtZSguYW9mLnRtcCwgLmFvZikKICAgIFItPj5GUzogZnN5bmMoZGlyKQogICAgTm90ZSBvdmVyIEFPRjogaW52YXJpYW50OiBleGFjdGx5IG9uZSBvZjxici8+e29sZCAuYW9mLCBuZXcgLmFvZn0gcHJlc2VudCBhdCBhbGwgdGltZXM?type=png)

The invariant the rewrite ADR commits to is "the canonical file is durable and consistent at every instant until the rename." A silently-empty side buffer would have meant the rename swapped in a file missing the appends issued during the snapshot. The replay test was the green-light proof that both sides of the dual-write are actually dual-writing.

## Honourable mention 2 — the test was wrong, not the code

The TTL sweeper is a 1 Hz goroutine that samples 20 keys under a read lock, then upgrades to a write lock for the eviction pass — `sync.RWMutex` doesn't support upgrade-in-place, so the upgrade is *release-and-reacquire-and-double-check*. The race test (`internal/store/sweeper_test.go`, commit [`bab6033`](https://github.com/prajwalmahajan101/toykv/commit/bab6033)) failed on its first stress run with four violations in 468k reads.

The bug was **in the test, not the store.** The test captured `t0 := time.Now()` before `Get`, but Get's internal time check happens later — sometimes much later if the scheduler preempts. A Get that correctly returned `(nil)` because the entry's TTL had elapsed by the time Get *internally* checked would be flagged as "spurious nil for unexpired key." The fix forced the memory model onto paper: I now capture `tAfter` after Get (upper-bounds the internal check) and `atomic.Load(lastExpireAt)` before Get (synchronizes-with the writer's `atomic.Store`). Then `tAfter < loadedExpireAt` is a real violation; no false positives possible.

> **The rule:** if a sweeper and a writer can race, the test has to schedule them, not hope. A test whose pass/fail depends on the scheduler's whim is a test that finds the race detector's bug, not your code's.

## ADRs as crystallization — and the ADR I almost didn't write

I budgeted four ADRs for v1.0.0 and shipped six. The interesting one is the TUI ADR.

The TUI ADR (Bubble Tea with an injectable `Doer`) is where the budget actively pushed against the code. The TUI was the first part of the project that took a third-party dep beyond `go-redis/v9` in tests, and the choice — `bubbletea` vs `tview` vs hand-rolled — was non-trivial. I knew it was non-trivial because every two days, in a different context, I caught myself re-deriving the same argument for the same choice. That's the signal: when the same reasoning costs me a second time, I didn't decide; I waited.

I had two options. Force the choice into one of the existing four ADRs and write a worse ADR. Or break the budget and write the TUI ADR on its own. I broke the budget. The post-mortem is that **the budget itself was the wrong abstraction.** I had treated ADR count as a proxy for noise, when the actual signal was ADR *quality* — and a forced-fit ADR is noisier than two clean ones.

The discipline, identical to the one I wrote in the `toymq` post: write the ADR the moment the decision is forced by code, never before. ADRs written ahead of code force the code to fit the ADR. ADRs written *at* crystallization record what the code already decided. The reason that discipline matters is that the TUI ADR was a decision I'd already made three times by the day I wrote it down — I just hadn't admitted I'd made it. The ADR is the admission.

> **The rule:** budget the quality bar, not the count. The cost of an ADR I write is one afternoon. The cost of an ADR I didn't write is a future me re-deriving the same argument in three different files. Those costs aren't comparable, and counting ADRs treats them as if they are.

## What I'd do differently

Three regrets from the critical path:

1. **Chaos composition should have shipped with the AOF, not at the end.** The `test/chaos/` soak (kill + pause + rewrite + writes overlapped) felt expensive enough to defer. It wasn't. Two-thirds of the harness reused the self-re-exec pattern I already had for the AOF crash test. Running a composition soak from the start would have caught the dual-write bug at the point the rewriter landed, because the rewriter would have been driven by a scaffolded integration test instead of an isolated unit test.
2. **`Store.Snapshot()` should have shipped with the AOF, not landed as a refactor PR when the rewriter needed it.** Snapshot is a contract boundary — "give me the current state, expired entries already evicted, in a form I can serialize." I deferred it because the AOF didn't *use* it yet. The refactor PR that extracted it later was harder to review than the BGREWRITEAOF work itself, because it touched nine files mechanically. Shipping the boundary up front would have cost one extra journal entry.
3. **The ADR budget number was the wrong abstraction.** See above. Budget the bar, not the count.

## What's next

`toykv-go` — the SDK extraction. The strategy I committed to is: hold the SDK split until the wire format is locked, then extract `internal/client`, `respfmt`, and `cmdparse` into a sibling repo. v1.0.0 of `toykv-go` tagged the same day as `toykv` v1.0.0.

`toymessenger` — the consumer. The reason `toykv` and `toymq` exist as separate projects is that there is a third project, an E2E-encrypted chat, that uses both: KV for session state and presence, MQ for the message log. `toymessenger` is where the two projects' opposite decisions get tested against a single user.

A `toykv v2` only if real use justifies it. The roadmap lists candidate features (lists/sets/hashes, AUTH + TLS, RDB snapshots, observability with `INFO` + `/metrics`, `SCAN`, `RENAME`), but the rule is the same one Redis uses: don't add a feature until somebody is asking for it and willing to live with the trade-off. A single-node KV that survives a v1 cycle without a v2 ask is a single-node KV that earned the cut.

## The one-line version

Write the bytes you'd send. Stamp them with a version. The next person to read them is you.

---

*Source: [`prajwalmahajan101/toykv`](https://github.com/prajwalmahajan101/toykv). Deeper docs: [`docs/HLD.md`](https://github.com/prajwalmahajan101/toykv/blob/main/docs/HLD.md), [`docs/LLD.md`](https://github.com/prajwalmahajan101/toykv/blob/main/docs/LLD.md), [`docs/TESTING.md`](https://github.com/prajwalmahajan101/toykv/blob/main/docs/TESTING.md), [`docs/BENCHMARKS.md`](https://github.com/prajwalmahajan101/toykv/blob/main/docs/BENCHMARKS.md). ADRs: [`docs/adr/`](https://github.com/prajwalmahajan101/toykv/tree/main/docs/adr). Release notes: [`docs/release-notes/v1.0.0.md`](https://github.com/prajwalmahajan101/toykv/blob/main/docs/release-notes/v1.0.0.md). Corrections welcome — open a discussion or file an issue.*

*Companion post: [Building toymq](https://dev.to/prajwalmahajan101/building-toymq-a-from-scratch-persistent-message-broker-in-go-ob7) — the sibling project this one is in dialogue with.*
