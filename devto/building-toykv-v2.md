---
title: "Building toykv v2: seven features that didn't earn the major, one refusal that did"
published: true
description: "How toykv went from a localhost-only KV (v1) to a deployable, safe-by-default one (v2): RESP3, lists/hashes, AUTH/TLS, INFO/SCAN, atomic keyspace ops, observability — seven features, without breaking the byte-compat contract v1 earned. The one break that earns 2.0.0 is a server that refuses to start."
tags: go, distributedsystems, backend, architecture
series: "Building distributed systems"
canonical_url: https://dev.to/prajwalmahajan101/building-toykv-v2-seven-features-that-didnt-earn-the-major-one-refusal-that-did-4jo8

# dev.to metadata
# article_id: 4204932
# published_at: 2026-07-22T10:34:13Z
# url: https://dev.to/prajwalmahajan101/building-toykv-v2-seven-features-that-didnt-earn-the-major-one-refusal-that-did-4jo8
---

> A retrospective on the v1 → v2 arc of [`toykv`](https://github.com/prajwalmahajan101/toykv) — the jump from a single-node, RESP2-compatible KV that *works on localhost* to one that's safe to bind off it. The [first post](https://dev.to/prajwalmahajan101/building-toykv-a-from-scratch-persistent-kv-in-go-and-why-i-took-the-opposite-call-from-toymq-5862) was about earning a durability contract and taking the opposite call from `toymq` three times. This one is about adding seven features to that store without betraying the two contracts v1 earned — byte-compatibility on the wire, and durability under crash — and then discovering, in the phase I'd budgeted for *polish*, that the scariest bugs of the whole arc were still hiding.

## TL;DR — the decision, and the check, you should remember

[Part 1](https://dev.to/prajwalmahajan101/building-toykv-a-from-scratch-persistent-kv-in-go-and-why-i-took-the-opposite-call-from-toymq-5862) built the dangerous part first: AOF → replay → RESP wire, with one rule — *absolute deadlines, never relative durations* — and a nine-row crash matrix proving it held. v1.0.0 shipped a KV that survives `SIGKILL` with zero acked-write loss, speaks enough RESP2 that `redis-cli` and `go-redis` are free test harnesses, and does all of it on localhost with no auth.

v2 is the deployment problem. The store is correct; the job is to make it *reachable* — RESP3, lists and hashes, AUTH and TLS, `INFO` and `SCAN`, a rebuilt TUI, atomic keyspace ops, OpenTelemetry — **without regressing byte-compat or durability**, and without pretending the version number is honest when it isn't.

Here's the uncomfortable thing about that version number. By pure semver, **the whole feature set is a `1.x`.** RESP3 is opt-in behind `HELLO 3`; RESP2 stays byte-identical. The new AOF format (v3) replays v1 and v2 files unchanged. AUTH and TLS are off by default. Observability with no endpoint is a no-op. Every one of those is *additive* — a pre-v2 client is served exactly as before. Adding features, however many, does not earn a major.

The one thing that makes `2.0.0` *honest* is **protected mode**: v1 would bind `0.0.0.0` with no auth and serve the internet a plaintext database. v2 **refuses to start** that configuration — the only line in the whole arc a pre-2.0 operator can't ignore. That's the decision to remember: **you don't earn a major by adding features — you earn it by changing a contract**, and the only contract change in v2 is a server that says no. It's the toykv-shaped sibling of what [`toymq` v2](https://dev.to/prajwalmahajan101/building-toymq-v2-eight-features-one-durability-contract-and-the-bug-that-proved-it-held-19ff) calls "the durability contract is sacred" — same series move, different axis: toymq protects *durability*; toykv protects *byte-compat + safe-by-default*.

But the *check* to remember is the one that nearly cost me. The final phase was scoped as "bench + polish + tag." It's where I found the two scariest bugs of the arc — a pre-auth DoS a single packet could trigger, and a 20% regression in the "free" telemetry path — and then, *after* tagging, two more: a RESP3 codec that could encode but not decode its own frames, and hash commands returning fields and values from two different random iterations. None were caught by a suite green across a six-way OS/Go matrix. All were caught by *running* something for the first time.

That collapses to a one-liner that belongs to a family. Each post in this series earns one the same shape:

| Post | The hero rule | The trap it names |
|---|---|---|
| [`toymq` v1](https://dev.to/prajwalmahajan101/building-toymq-a-from-scratch-persistent-message-broker-in-go-ob7) | **zero values are not sentinels** | `lastAcked == 0` meant both "acked id 0" and "never acked" |
| [`toykv` v1](https://dev.to/prajwalmahajan101/building-toykv-a-from-scratch-persistent-kv-in-go-and-why-i-took-the-opposite-call-from-toymq-5862) | **durations are not deadlines** | `EX 5` on disk re-applies against `now` after every restart |
| [`toymq` v2](https://dev.to/prajwalmahajan101/building-toymq-v2-eight-features-one-durability-contract-and-the-bug-that-proved-it-held-19ff) | **derived state is not a source of truth** | a `dedupe.json` sidecar re-materialises the duplicate it was meant to kill |
| **`toykv` v2 (this post)** | **a green check is not a verified one** | a suite green on the wrong reader, a `t.Skip`ped compat sweep, and a benchmark nobody ran all show the same green |

Every rule in that table is the same shape: **a thing that looks authoritative until a restart, a bind, or a first real run calls its bluff.** A zero looks like a sentinel; a duration looks like a deadline; derived state looks like truth. And a green checkmark looks like verification — right up until you notice the test skipped, ran against the wrong surface, or was never run at all.

> **The rule:** a roadmap — and a CI dashboard — records *intended* state, not *verified* state. Green is not verified. The only check that verifies anything is the one you actually ran, this session, against the surface a user actually walks.

The numbers: ~5k LOC grew past 12k, seven design decisions recorded (one *amended* after release — the supersede trail below), the AOF at format v3, four binaries still, and a `2.0.0` whose CHANGELOG cut is dated 2026-07-18. The order:

```
RESP3 → lists/hashes + AOF v3 → AUTH/TLS → INFO/SCAN
   → TUI rebuild → protected mode + atomic ops → OpenTelemetry → release
```

![v2 feature ordering: RESP3, lists/hashes, INFO/SCAN, AUTH/TLS, and the TUI rebuild are additive (a v1 client can't tell); protected mode is the single red contract break that earns 2.0.0; then verify and tag](https://mermaid.ink/img/Zmxvd2NoYXJ0IFRCCiAgICBzdWJncmFwaCBBREQxWyJBZGRpdGl2ZSDigJQgYSB2MSBjbGllbnQgY2Fubm90IHRlbGwiXQogICAgICAgIEFbIlJFU1AzPGJyLz5vcHQtaW4gSEVMTE8gMyJdCiAgICAgICAgQlsibGlzdHMgLyBoYXNoZXM8YnIvPkFPRiB2MyByZXBsYXlzIHYxICsgdjIiXQogICAgICAgIENbIklORk8gLyBTQ0FOPGJyLz5zZXEgY3Vyc29yLCBubyBmb3JtYXQgYnVtcCJdCiAgICBlbmQKICAgIHN1YmdyYXBoIERFUExPWVsiTGlmdCB0aGUgZGVwbG95IGNlaWxpbmcg4oCUIG9wdC1pbiJdCiAgICAgICAgRFsiQVVUSCArIFRMUyJdCiAgICAgICAgRVsiVFVJIHJlYnVpbGQgKGNvbnN1bWVyKSJdCiAgICBlbmQKICAgIHN1YmdyYXBoIEJSRUFLWyJUaGUgT05FIGNvbnRyYWN0IGJyZWFrIl0KICAgICAgICBGWyJwcm90ZWN0ZWQgbW9kZTxici8+UkVGVVNFIFRPIFNUQVJUIl0KICAgIGVuZAogICAgc3ViZ3JhcGggU0VFWyJTZWUgaXQgcnVuIOKAlCBuby1vcCBieSBkZWZhdWx0Il0KICAgICAgICBHWyJPcGVuVGVsZW1ldHJ5Il0KICAgIGVuZAogICAgSFsidmVyaWZ5ICsgdGFnIHYyLjAuMCJdCiAgICBBIC0tPiBCIC0tPiBEIC0tPiBDIC0tPiBFIC0tPiBGIC0tPiBHIC0tPiBICiAgICBzdHlsZSBBREQxIGZpbGw6I2RiZWFmZSxzdHJva2U6IzFkNGVkOAogICAgc3R5bGUgREVQTE9ZIGZpbGw6I2ZlZjNjNyxzdHJva2U6I2Q5NzcwNgogICAgc3R5bGUgQlJFQUsgZmlsbDojZmVlMmUyLHN0cm9rZTojZGMyNjI2CiAgICBzdHlsZSBTRUUgZmlsbDojZjNlOGZmLHN0cm9rZTojN2MzYWVkCiAgICBzdHlsZSBIIGZpbGw6I2RjZmNlNyxzdHJva2U6IzE2YTM0YQo=?type=png)

Six additive steps in blue, amber, and purple; one red break in the middle. v1's rule was "risk sinks *down* — build the AOF first." This is that rule rotated: with the store frozen, risk drains *outward from the wire*, and the single red node is the only thing a pre-2.0 user feels. The point of this post, like the last one, is the order and the rules — not the code.

## Why the order is the design

With risk draining outward from the wire (above), three ordering principles did the work.

**The wire foundation lands before the thing that needs it.** RESP3 ships *before* the typed values, even though it has no typed values to carry yet, because `HGETALL` → map is the first real consumer of its frames — the same bottom-up logic that put the AOF before TTL in v1.

**The deployment ceiling lifts before the thing that needs it lifted.** Auth and TLS make the server safe off-host; `SCAN` and the TUI are what you'd actually *use* off-host. A browsable KV you can't safely bind is a tool for nobody.

**The one break clusters behind the major bump.** v2 has exactly one break a pre-2.0 user feels — protected mode refusing an unsafe bind. Everything else is additive by construction, so the major number rides on a single, nameable, defensible change.

> **The rule:** you get one contract break per major version. Spend it on the one thing that makes the version honest — for toykv, a refuse-to-start — and keep everything else additive so a v1 user upgrades without noticing.

And each feature owns its risk test, written where the risk was introduced — no "integration pass at the end" that discovers the value-types crash bug during the observability work. Which is exactly why it stung when the release pass found bugs anyway: it was the first time anyone tested the *seams between* features, and the seams were where the bugs lived.

## Seven rules v2 earned

Like part 1, I didn't write these up front. I read the decision records back and asked *what did I refuse to compromise on?*

1. **A major is earned by changing a contract, not counting features.** Five additive features don't add up to a `2.0.0` — they add up to a fat `1.x`. Protected mode is the single behavioural break, and its decision record says so: "one deliberate breaking change plus one correctness win, not feature-count padding."
2. **Additive means *provably* additive — the downgrade point is where you prove it.** Handlers return protocol-agnostic `resp.Value`s and a *single* writer function downgrades each rich kind to RESP2 — map→flat array, null→`$-1`. The compat sweep asserts every command's exact RESP2 bytes, so "unchanged" is enforced, not hoped. `HGETALL` returned `resp.Map(...)` and changed zero writer code.

![The single downgrade point: handlers return rich resp.Value kinds; WriteFrameProto emits native RESP3 frames on proto 3 and downgrades each to its RESP2 equivalent on proto 2, so the RESP2 bytes stay byte-identical to v1](https://mermaid.ink/img/Zmxvd2NoYXJ0IExSCiAgICBzdWJncmFwaCBIWyJIYW5kbGVycyByZXR1cm4gcmljaCByZXNwLlZhbHVlIl0KICAgICAgICBIMVsiR0VUIG1pc3MgLT4gTnVsbCJdCiAgICAgICAgSDJbIkhHRVRBTEwgLT4gTWFwIl0KICAgICAgICBIM1siSU5GTyAtPiBWZXJiYXRpbSJdCiAgICBlbmQKICAgIHN1YmdyYXBoIFdbIldyaXRlRnJhbWVQcm90byDigJQgdGhlIHNpbmdsZSBkb3duZ3JhZGUgcG9pbnQiXQogICAgICAgIFdQeyJwcm90byA9PSAzID8ifQogICAgZW5kCiAgICBzdWJncmFwaCBQM1siUkVTUDMgbmF0aXZlIl0KICAgICAgICBSM0FbIl8gbnVsbCJdCiAgICAgICAgUjNCWyIlTiBtYXAiXQogICAgICAgIFIzQ1siPSB2ZXJiYXRpbSJdCiAgICBlbmQKICAgIHN1YmdyYXBoIFAyWyJSRVNQMiDigJQgYnl0ZS1pZGVudGljYWwgdG8gdjEiXQogICAgICAgIFIyQVsiJC0xIl0KICAgICAgICBSMkJbIioyTiBmbGF0IGFycmF5Il0KICAgICAgICBSMkNbIiQgYnVsayBzdHJpbmciXQogICAgZW5kCiAgICBIMSAtLT4gV1AKICAgIEgyIC0tPiBXUAogICAgSDMgLS0+IFdQCiAgICBXUCAtLSBwcm90bzMgLS0+IFIzQQogICAgV1AgLS0gcHJvdG8zIC0tPiBSM0IKICAgIFdQIC0tIHByb3RvMyAtLT4gUjNDCiAgICBXUCAtLSBwcm90bzIgLS0+IFIyQQogICAgV1AgLS0gcHJvdG8yIC0tPiBSMkIKICAgIFdQIC0tIHByb3RvMiAtLT4gUjJDCiAgICBzdHlsZSBIIGZpbGw6I2RiZWFmZSxzdHJva2U6IzFkNGVkOAogICAgc3R5bGUgVyBmaWxsOiNmZWYzYzcsc3Ryb2tlOiNkOTc3MDYKICAgIHN0eWxlIFAzIGZpbGw6I2YzZThmZixzdHJva2U6IzdjM2FlZAogICAgc3R5bGUgUDIgZmlsbDojZGNmY2U3LHN0cm9rZTojMTZhMzRhCg==?type=png)
3. **The version byte's hardest case is the old file you're about to append to.** Everyone bumps the format for new files; the sleeper is the *existing* v2 file the new binary appends to. The instant an `LPUSH` lands in a file whose header says `0x02`, the header lies. So `Open()` pwrites the version byte at offset 7 and fsyncs *before* any append — a file's header version ≥ its newest record format at every instant.
4. **The durability contract is still sacred — and a feature is the most likely thing to break it.** The `aof.append` span is a *pure outer wrapper*: a slow fsync shows as latency, but the span can't reorder a byte inside the writer. Same lesson [`toymq` v2 learned from the other side](https://dev.to/prajwalmahajan101/building-toymq-v2-eight-features-one-durability-contract-and-the-bug-that-proved-it-held-19ff) — once you've earned durability, the thing most likely to break it isn't hardware; it's the next feature.
5. **Constant-time where it's secret, plain compare where it isn't.** AUTH uses `crypto/subtle.ConstantTimeCompare` on the password, a plain compare on the public `default` username, and returns the *identical* `-WRONGPASS invalid username-password pair or user is disabled.` for a wrong user or a wrong password — no oracle for which half failed.
6. **Off-by-default has to be *free*, and "free" is a benchmark, not an assumption.** The observability layer is a no-op by construction, and I asserted that made it free. **It didn't** (see the release pass, below): building an `attribute.Set` allocates whether the SDK records it or not — off by ~20% until measured.
7. **In-memory derived state keeps the on-disk format frozen.** `SCAN`'s cursor is a never-persisted per-key `seq`, re-derived in record order at replay; `RENAME`/`COPY` record verbatim under the v3 reader. Neither bumped the AOF. A format bump is a liability you pay forever; a derivation is a loop you pay once at `Open`.

---

## The dangerous parts of v2, one at a time

### Lists and hashes — the header byte you rewrite under the old binary's feet

Adding lists and hashes meant the store's value became a tagged union (`typ` + three concrete fields, no `any`, no interface — so no type assertion can panic), and the AOF took its second format bump to v3. The union was mechanical. The dangerous part was the header upgrade described in rule 3.

The `O_APPEND` gotcha en route is the kind of thing you only hit by doing it: the live append fd is opened `O_APPEND`, and Go *rejects* `WriteAt` on an append-only handle. So the one-byte header rewrite needs its own short-lived, non-append handle, used and closed before the append path takes over. The restart probe after a fresh dir:

```
header bytes: 54 4f 59 4b 56 00 00 03   →  "TOYKV\0\0" + version 0x03
```

The crash test for this work is model-based, not a spot check: the parent keeps its own model of acked mutations, `SIGKILL`s the child mid-stream of mixed list/hash ops under `fsync=always`, restarts, and asserts the replayed store equals the model *exactly, list order included* — the v1 durability discipline extended to the surface that could newly corrupt it.

### Honourable mention — the "flaky" test was telling the truth

This one surfaced during the auth + TLS work, because that's where CI caught it, but it's a v1 bug that had been *flakily caught since the live-rewrite path landed*. CI went red: `TestAOF_CrashInjection_DuringRewrite/mid-kill`, "2/82 acked SETs lost across rewrite+SIGKILL." The roadmap's release gate had this exact test pre-authorized for quarantine — "flaky, fix or quarantine." It would have been so easy to quarantine it.

The failure signature was the entire diagnosis. The lost keys were always `live000` and `live001` — the *first two* live writes during the rewrite, never random ones. A timing flake loses arbitrary keys; losing the *earliest* is an ordering bug. The mechanism is the diagram below: the rename unlinks the old inode before the side buffer is drained, so in the `[rename, drain]` window an acked write lives only in the doomed inode plus a volatile buffer. `-race` on CI widened the window enough to hit it; locally it passed 15/15 without `-race`.

Two things made it into the lesson. **The decision record had rationalized the bug** — the original rewrite design documented that window and called it "no worse than the single-file baseline," which was wrong: v1's single-file path never unlinked live data mid-write, and a crash-invariant table with a data-loss row *marked acceptable* is a design smell in a durability-first project. **A decision record written after the code can launder a bug into a decision.** Meanwhile **the low-level design had it right all along** — it specified fold-then-rename; the code shipped the reverse.

The fix (`901806d`) folds the side buffer into `.tmp` and fsyncs it *before* the rename, holding the append lock across the whole capture → rename → fd-swap. And because I couldn't reproduce the SIGKILL flake locally, I proved the mechanism deterministically: an in-process regression test with an `afterRenameHook` that replays the canonical file at the exact rename instant — it recovers only the snapshot and fails loudly on the old ordering, passes on the fix, **0/30 under `-race`**. A probabilistic crash test became a deterministic ordering assertion.

![The durability window: before the fix, the rename unlinks the old inode before the side buffer is drained, so a SIGKILL in the [rename, drain] window loses live000/live001; after 901806d, the side buffer is folded into .tmp and fsynced before the rename, so the canonical file is complete at every instant](https://mermaid.ink/img/Zmxvd2NoYXJ0IFRCCiAgICBzdWJncmFwaCBCQURbIkJlZm9yZTogcmVuYW1lLCB0aGVuIGRyYWluIOKAlCBsb3NlcyBsaXZlMDAwIC8gbGl2ZTAwMSJdCiAgICAgICAgZGlyZWN0aW9uIFRCCiAgICAgICAgQjFbInNuYXBzaG90IC0+IC50bXAiXSAtLT4gQjJbInJlbmFtZSAudG1wIG9udG8gY2Fub25pY2FsIl0KICAgICAgICBCMiAtLT4gQjNbIm9sZCBpbm9kZSBVTkxJTktFRDxici8+YWNrZWQgd3JpdGVzIGxpdmUgb25seSBpbiBSQU0iXQogICAgICAgIEIzIC0tPiBCNFsiZHJhaW4gc2lkZSBidWZmZXIiXQogICAgICAgIEIzIC0uICJTSUdLSUxMIGluIHRoZSBbcmVuYW1lLCBkcmFpbl0gd2luZG93IiAuLT4gQlhbImFja2VkIHdyaXRlIExPU1QiXQogICAgZW5kCiAgICBzdWJncmFwaCBHT09EWyJBZnRlcjogZm9sZCwgdGhlbiByZW5hbWUg4oCUIDkwMTgwNmQiXQogICAgICAgIGRpcmVjdGlvbiBUQgogICAgICAgIEcxWyJzbmFwc2hvdCAtPiAudG1wIl0gLS0+IEcyWyJmb2xkIHNpZGUgYnVmZmVyIGludG8gLnRtcCArIGZzeW5jIl0KICAgICAgICBHMiAtLT4gRzNbInJlbmFtZSAudG1wIG9udG8gY2Fub25pY2FsIl0KICAgICAgICBHMyAtLT4gRzRbImNhbm9uaWNhbCBpcyBDT01QTEVURSBhdCBldmVyeSBpbnN0YW50Il0KICAgIGVuZAogICAgc3R5bGUgQkFEIGZpbGw6I2ZlZTJlMixzdHJva2U6I2RjMjYyNgogICAgc3R5bGUgR09PRCBmaWxsOiNkY2ZjZTcsc3Ryb2tlOiMxNmEzNGEK?type=png)

> **The rule:** read the failure, not the "FAIL." A crash test that loses *always the earliest* writes isn't flaky infra — it's an ordering bug wearing a probability distribution. And when you can't reproduce the race, model the instant the crash would expose and assert the invariant there.

### AUTH + TLS — the feature that shipped in one boolean

The RESP3 work paid a churn I chose deliberately: every handler gained a `*connState` parameter most of them ignore (`_ *connState`). It looked like noise. It was the seam. When auth arrived, `connState` gained one `bool`, `dispatch` gained one condition, `HELLO`'s stub branch became a real check, and the listener got wrapped in `tls.NewListener`. **Zero handlers touched.** The plan's "verified against live code" pass even found `conn_state.go`'s doc comment already reading "authentication state lands here" — pre-plumbed a release early.

Two ordering calls carried weight. Gating goes *before* the command-table lookup, so an unauthenticated `NOSUCHCMD` gets `-NOAUTH`, not `-ERR unknown command` — the table leaks nothing pre-auth. And the no-auth path is the *degenerate* case: `newConnState(id, s.cfg.RequirePass == "")` authenticates every connection of an auth-less server at accept, so v1 behaviour is provably identical — the same trick that auth-exempts AOF replay by construction.

### SCAN — you can't port Redis's cursor to a Go map

`SCAN` reads like a command handler. The actual work was a store-model decision the language forced. Redis's cursor is reverse-binary iteration over its open-addressed hash buckets — it survives rehashing and gives the "every key present for the whole scan is returned" guarantee. A Go `map[string]entry` randomizes iteration and exposes no buckets. Redis's technique is simply *unavailable*.

The answer: stamp each entry with a monotonic `seq` at creation, preserve it across every update, and make the cursor a *seq value*. `Scan` returns live keys with `seq > cursor` sorted by seq; a key alive for the whole scan keeps its seq and is always reached. The subtlety is that `seq` survives *updates* but not *re-creation* — and the code nearly made that free, because read-modify-write paths copy the whole entry back, so `seq` rides along; only four sites build a *fresh* entry, and one `grep` for `entry{` up front meant the invariant held on the first test run. The owned risk test seeds low-seq noise, then the higher-seq must-see set, then deletes the noise *while scanning* — the exact index-shift a positional cursor fails and the seq cursor survives.

`INFO` taught the adjacent lesson. The roadmap said "RESP3 map when the client is on RESP3" — but real Redis returns `INFO` as a bulk/verbatim string, *never* a map, and both `go-redis .Info()` and `redis-cli info` parse the `# Section\nkey:value` text. A map breaks both. I surfaced it as a decision and corrected the roadmap. **A spec can be wrong; comply with reality, not the doc.**

### Protected mode — the refusal that earns the major

Everything above is additive. This is the break. Protected mode refuses to *start* — not to serve — when bound to a non-loopback address with neither `requirepass` nor TLS. The clean call was putting `checkProtectedMode` inside `server.New` rather than inline in `main`: unit-testable in isolation, it protects any embedder, and it runs *before* AOF replay so an unsafe bind never touches disk.

The API-shape detail that makes it safe by default is invisible until you construct the struct directly: `Config.ProtectedMode` is a **`string`** whose zero value `""` means *enabled*. A `bool` would default to `false` = *disabled*, so a bare `Config{}` would be unsafe — the opposite of the point (the decision record lists exactly that as the rejected alternative). It rhymes with the sibling project's "zero values are not sentinels": here the zero value has to mean the *safe* thing, so I chose a type whose zero value does.

The other half of this work — atomic `RENAME`/`RENAMENX`/`COPY`, a correctness win that rides v3 verbatim (rule 7) — produced the small relatable gotcha. The plan said "reject Redis's `DB` option outright." Then the go-redis round-trip failed: `go-redis` v9's `Copy` *always* sends `COPY src dst DB 0`. The Redis-faithful fix — accept index 0 (toykv's single DB), reject others with `-ERR DB index is out of range` — is both more correct and unblocks the client. **Verify the client's wire bytes, not just your server's spec, before deciding what to reject.**

### Observability, then release — no-op providers are not free, and the auth review missed a pre-auth DoS

The observability layer shipped with the no-op-by-construction design from rule 6, plus a churn call worth naming: the instrument inventory implied threading `context.Context` into ~30 store methods to parent the `store.<op>` spans — **~250 edits, almost all in tests, for byte-identical telemetry**. So the span is created at the server→store boundary instead: same trace tree, ~0 test churn, store package stays context-free. *The means (ctx in store) was never the goal (the span).*

Then the release pass — "bench + polish + tag" — turned into the most instructive phase of the arc, because it was the first time anything was *verified* rather than *built*. Two gate items marked green were regressions:

**A security review scoped at auth found a pre-auth DoS one layer below it.** The review of the auth/TLS/protected-mode surface came back clean — but it looked *under* the auth gate, at the RESP codec that runs before it, and found `readArray` doing `make([]Value, n)` with `n` bounded only against negatives (a `*1000000000\r\n` header is a multi-GB OOM from one packet) and unbounded recursion depth (a nested-array stack-exhaustion *fatal* panic you can't `recover`). Both reachable **before** any credential, defeating the "safe-by-default" claim that earns `2.0.0`. Fixed in `ef8a423`: `MaxArrayLen = 1<<20` (matching Redis's `proto-max-multibulk-len`), `MaxDepth = 32`, both rejecting with `ErrTooLarge` before allocation.

> **The rule:** a clean bill on the *named* surface is not a clean bill on the *reachable* surface. The auth review found the DoS precisely because it didn't stop at auth — it asked "what runs before this?"

**The "free" telemetry path cost 20%.** The parity benchmark *existed* (`BenchmarkObserveCommand_Disabled`, 29 allocs/op in isolation) — but nobody had run the actual A/B against the pre-observability binary (`a3d62e1`). Doing it: **SET −21%, GET −18%**, every round. Root cause: building an `attribute.Set` allocates against no-op providers too, so `observeCommand` rebuilding options per command taxed the disabled path. The fix memoized per-command attributes in a `cmdInstr` cache — **no `if enabled` guard added** — dropping the path to **14 allocs/op, 1757 ns/op**, SET back to parity.

Both would have shipped in `2.0.0` if I'd trusted the roadmap's "gates green." That's the headline of the whole arc: **the phase I expected to rubber-stamp is where I found the scariest bugs, because it was the first time I checked instead of built.**

### After the tag — writing the manual to test it found two more

The sequel to that lesson is that even the release pass didn't finish the verification. I wrote `docs/TEST_CASES.md` as a "local testing guide," and *running it by hand* against a live server surfaced two more already-shipped defects the six-way CI matrix never caught.

**The RESP3 codec was asymmetric.** The writer encoded all seven RESP3 kinds (`% ~ , # _ = >`); `ReadFrame` decoded only RESP2. `internal/client` uses that reader — so `HELLO 3` under `toykv-cli`/`toykv-tui` died on the first RESP3 reply with `resp: unknown prefix '%'`. The server's RESP3 was correct and go-redis round-tripped fine in e2e, because **every existing test exercised a reader that wasn't ours.** The missing gate was obvious in hindsight — an encode→decode round-trip over the codec's own two halves — and because it never existed, the asymmetry was invisible. And "decoded" wasn't "done": after the reader fix, the CLI printed `(unknown kind '%')` because `internal/respfmt` couldn't *render* the map. A ninth change I'd never have found without running the binary.

**HKEYS/HVALS/HGETALL didn't correspond.** The hash was a bare `map[string][]byte`, and each reader ranged it independently — so `HKEYS[i]` and `HVALS[i]` came from two separate randomized iterations, a silent divergence from Redis's pairing guarantee. Fixed with an insertion-order `fieldOrder []string` on the entry, threaded through HSET/HDEL, AOF replay (free — records replay in order), and BGREWRITEAOF.

> **The rule:** a codec needs a test that composes its two halves. A writer-test and a reader-test that never meet can both stay green while the thing between them is broken. Round-trip your own inverse operations.

The last skip was the most on-the-nose. Every section of `TEST_CASES.md` passed *except* §5 (redis-cli compat), which **skipped** — no `redis-cli` on the box. `TestRedisCLI_ByteCompat` does the honest thing (`exec.LookPath`, `t.Skip` if absent), but "green because it skipped" is the exact blind spot. Rather than weaken the test, I gave it the tooling: a `scripts/redis-cli` shim wrapping `docker run --rm --network host … redis-cli "$@"` (the benchmark already runs `valkey/valkey:8-alpine`, which ships a `redis-cli` symlink), plus `make compat` prepending `scripts/` to `PATH` so the test's own `LookPath` resolves to Docker. No test change, no native dependency, all 20 cases pass — the `PATH`-shim-over-`LookPath` seam un-skips any "needs an external binary" test.

---

## Honourable mentions — three smaller bugs that earned a line

- **The teatest flake that was a wrong assertion.** The TUI paging smoke first asserted "after `]`, page-1 key `k000` is *gone* from the view." It passed alone and failed under the full `-race` suite. Not timing — `teatest`'s `Output()` is the **cumulative** frame stream, not the current screen, so once `k000` was ever drawn, `!contains("k000")` can never hold again. Replaced with a *positive* assertion on `k050` (which lives only in page 2's SCAN range). **Assert on what appears in a cumulative stream, never on what should have vanished.**
- **The PING whitelist the readiness probe depended on.** Auth whitelists `{AUTH, HELLO, PING}` pre-auth — a deliberate deviation from Redis, which gates `PING`. I'd written it down as an operability-over-parity call, and then the e2e harness's own readiness probe turned out to *depend* on it: it PINGs before the test can AUTH. The deviation argued for itself — a liveness check shouldn't need credentials.
- **`go mod tidy` prunes deps until first import.** The observability work fetched the whole OTel module set up front; the first `tidy` dropped everything not yet imported. Rather than fight it, I let each commit's first import re-add its module — so the `go.mod` delta in each commit is exactly what that commit uses. A cleaner history fell out of not fighting the tool.

## Proof it composes

Rules and war stories are claims. Three artifacts back them.

**Byte-compat is mechanical.** The dual-protocol sweep replays every command over a live socket on both protocols and asserts the exact bytes. Because the reader is RESP2-only, RESP3 replies are asserted as raw bytes via `io.ReadFull` — which is *more* honest for a compat sweep than round-tripping through a parser. A live probe against the built binary, one connection, `HELLO 3` then `GET nope` then `PING`:

```
%7\r\n$6\r\nserver\r\n$5\r\ntoykv\r\n … $5\r\nproto\r\n:3\r\n$2\r\nid\r\n:1\r\n …
_\r\n
+PONG\r\n
```

Native `%` map on `HELLO 3`, native `_` null on the miss, `+PONG` unchanged — and the RESP2 client gets `$-1` for that same miss, byte-identical to v1.

**Durability holds with tracing compiled in.** The crash suite and the `test/chaos` soak stay green with spans wrapping the AOF path (rule 4) — the latency signal survives, the ordering is untouched. And **off-by-default is measured, not assumed**: `TestObserveCommand_Disabled_AllocBudget` pins the disabled path at ≤16 allocs/op, turning "OTel-off is free" from an assumption I'd got *wrong* into a gate-enforced benchmark (rule 6).

## Decision records as crystallization — the supersede trail

Both earlier posts landed on the same discipline: write the decision record the moment the decision is *forced by code*, so it records what the code already decided. [`toymq` v2 added the version-2 wrinkle](https://dev.to/prajwalmahajan101/building-toymq-v2-eight-features-one-durability-contract-and-the-bug-that-proved-it-held-19ff): most v2 records *retire* an earlier one, and the honest move is to write a *new* record with a `Supersedes` header rather than quietly editing the old one to look prescient. toykv v2 has the same trail — but it also has the sharpest example yet of *why* the discipline matters, because one of its records was amended by a bug found **after release**.

The tagged-union store's decision record originally said the hash iterated the Go map independently, "leaving order unspecified." When the hash-order divergence surfaced from writing the test guide, I didn't delete that sentence. I amended the consequences with the correction, dated, naming the fix branch: "**Corrected post-v2.0.0** (`fix/resp3-reader-hash-order`): the entry now tracks field insertion order … so `HKEYS[i]↔HVALS[i]↔HGETALL` correspond as Redis guarantees." The record now carries its own mistake, legibly, next to the fix. A future reader asking "why is there a `fieldOrder` slice?" gets the whole timeline — the wrong assumption, the bug, the fix — instead of a clean story that pretends the divergence never shipped.

The same thing happened to the observability record: its post-release amendment records that "no-op providers make it free" was false, keeps the original claim visible, and appends the measurement that disproved it plus the memoization that fixed it. Two records that carry their own corrections.

> **The rule:** supersede and amend, don't overwrite. A decision record you edited to look right is a decision with no history — and the most valuable thing a version-2 decision log can teach is *when you were wrong and how you found out.*

## What I'd do differently

- **The codec round-trip test should have shipped with the codec.** The moment you have an encoder and a decoder, the *first* test is `decode(encode(x)) == x` — cheaper than every golden I wrote while `HELLO 3` stayed broken under our own client.
- **The parity A/B belonged in the observability work, not the release pass.** The benchmark existed the whole time; a gate you re-*read* instead of re-*run* is theatre. "Run the A/B" should have been an exit criterion of the work that introduced the risk.
- **The security review should have been scoped one layer wider from the start.** It found the pre-auth DoS *because* it looked below auth — but that was luck of framing. "What runs before the gate you're auditing?" should be step one of any auth review.
- **Verification is a phase, not a suffix on "polish."** Folding it into the release is why two bugs almost shipped and two more shipped anyway. Budget it as its own phase — and treat *writing the manual test guide* as a run, because it is one.

## What's next

v2 makes toykv *deployable* single-node: safe by default, byte-compatible, observable. Going multi-node stays open on purpose — like [`toymq`'s v3 question](https://dev.to/prajwalmahajan101/building-toymq-v2-eight-features-one-durability-contract-and-the-bug-that-proved-it-held-19ff), the roadmap keeps three honest trajectories: **A** stop at v1.x; **B** v1 → v2, usable single-node, stop here; **C** go distributed via `ToyRaft`, only if it ships as a vendorable library. The default is B, reviewed after each major tag — and the `SCAN` seq-cursor and atomic keyspace ops are already shaped so a future `HSCAN`/`SSCAN` or Raft-applied op fits without a rewrite, keeping the option cheap to exercise and cheap to decline.

The nearer work is the backlog the decision records are honest about: TLS in the shared client (the TUI and CLI still can't reach a TLS-only server — a deliberate deferral), rate-limiting on AUTH, and env/file sourcing for `-requirepass` so it stops showing up in `ps`. None of them earn a major. All of them are additive. That's the point.

## The one-line version

The series, in four rules that are the same rule. toymq v1: *zero values are not sentinels.* toykv v1: *durations are not deadlines.* toymq v2: *derived state is not a source of truth.* toykv v2: **a green check is not a verified one.** Each is a thing that looked authoritative until something — a restart, a bind, a first real run — called its bluff. Part 1 said *write the bytes you'd send, and stamp them with a version.* Part 2 says **you earn a major by changing one contract, not by adding seven features — and the roadmap that calls all seven done is recording what you intended, not what you ran.**

---

*Source: [`prajwalmahajan101/toykv`](https://github.com/prajwalmahajan101/toykv). The v2 arc in detail:* [`docs/ROADMAP.md`](https://github.com/prajwalmahajan101/toykv/blob/main/docs/ROADMAP.md), [`CHANGELOG.md`](https://github.com/prajwalmahajan101/toykv/blob/main/CHANGELOG.md), [`docs/TEST_CASES.md`](https://github.com/prajwalmahajan101/toykv/blob/main/docs/TEST_CASES.md), and the seven v2 decision records in [`docs/adr/`](https://github.com/prajwalmahajan101/toykv/tree/main/docs/adr) — each captures one decision above, and two carry their own post-hoc corrections. Corrections welcome — open a discussion or file an issue.*

*Companion posts in this series: [Building toykv (part 1)](https://dev.to/prajwalmahajan101/building-toykv-a-from-scratch-persistent-kv-in-go-and-why-i-took-the-opposite-call-from-toymq-5862) — the durable core this one builds on — and [Building toymq v2](https://dev.to/prajwalmahajan101/building-toymq-v2-eight-features-one-durability-contract-and-the-bug-that-proved-it-held-19ff) — the sibling project that earned its major by protecting a different contract.*
