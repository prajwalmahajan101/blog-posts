The scariest bug in toymq v2 wasn't a crash. It was a feature.

A retention sweeper quietly deleted an acked message 40 minutes before it was due — because a *second* feature, delayed messages, had parked it in an old WAL segment the sweeper was allowed to reclaim. Two features, each correct alone, corrupting each other. No crash. No race. A message lost to a feature.

toymq v1 taught me: zero values are not sentinels.
toykv taught me: durations are not deadlines.
toymq v2 taught me: derived state is not a source of truth.

Same shape of bug every time — a value that looks authoritative until the system restarts in a state it didn't anticipate.

v1 built a single-node broker that survives SIGKILL with zero acked-message loss. v2 is the harder problem: bolt on eight features people actually need — auth, TLS, partitions, backpressure, retention, dead-letter queues, delayed messages, trace correlation — WITHOUT regressing the one property v1 spent four days earning.

Three rules I earned doing it:

- The durability contract is sacred. A feature negotiates around it, never through it. Batched fsync trades latency for throughput, but `committed` still advances only after the fsync returns — an OK never means less than it did in v1.
- You get one wire break per major version. Spend it on everything at once, gated behind the same major bump. Everything else stays additive.
- Design every feature around the seam the distributed future needs. The dedupe index rebuilds from the WAL — not a sidecar file — because that's exactly the seam a v3 Raft node will reuse to rebuild state.

Proof, not vibes: a 16-cell integration matrix (fsync × TLS × auth × partitions) green under the race detector, and the SIGKILL chaos soak re-run in batched-fsync mode — 3 restarts, zero acked-message loss.

Full retrospective: 3 diagrams, the handshake race that dropped 40% of rejections under load, the test that hung for 600 seconds (the bug was in the test), and the ADR supersede-trail that keeps the decision log honest.

Post: https://dev.to/prajwalmahajan101/building-toymq-v2-eight-features-one-durability-contract-and-the-bug-that-proved-it-held-19ff
Source: https://github.com/prajwalmahajan101/toymq

#golang #distributedsystems #softwareengineering #backend #softwarearchitecture
