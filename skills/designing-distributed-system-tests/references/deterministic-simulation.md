# Deterministic Simulation Testing (DST)

**Last verified:** 2026-05-16

## When to reach for it
Use DST for concurrency-heavy or async-IO-heavy code where you can plumb all IO and clock through a controllable runtime; you want bug reproducibility from seed alone and don't need to test the real network or kernel behavior.

## What it detects well
- Race conditions and ordering bugs in concurrent code.
- Retry storms and exponential-backoff pathologies.
- Deadlocks and hot-loop livelocks.
- Bugs that surface only under specific message interleavings.
- Found bugs are minimally reproducible from a single seed.

## What it misses
- Anything outside the simulated boundary: kernel bugs, NIC driver bugs, disk firmware bugs.
- Real-world timing weirdness and latency distributions.
- Performance pathologies that depend on real CPU cache behavior.
- Interactions with OS scheduling and resource limits.

## Concrete tools
- `FoundationDB DST harness` — the original; deeply integrated into FDB — https://apple.github.io/foundationdb/testing.html
- `TigerBeetle VOPR` — open-source DST harness with property assertions — https://github.com/tigerbeetle/tigerbeetle
- `madsim` (RisingWave) — Rust DST runtime for async — https://github.com/madsim-rs/madsim
- `Antithesis` — commercial autonomous testing built on DST — https://antithesis.com
- `shuttle` — Rust deterministic scheduler for concurrent code — https://github.com/awslabs/shuttle
- `turmoil` — Rust distributed-systems simulator — https://github.com/tokio-rs/turmoil

## Papers to cite
- "What's the big deal about Deterministic Simulation Testing?" (Phil Eaton) — accessible intro — https://notes.eatonphil.com/2024-08-20-deterministic-simulation-testing.html
- "Is something bugging you?" (Antithesis) — DST history and autonomous extension — https://antithesis.com/blog/is_something_bugging_you/

## Cost / wall-clock signal
Large upfront infrastructure cost to plumb IO and clock; per-scenario runs are cheap (seconds to minutes); total wall-clock scales with seed budget you choose to explore.

## How a plan typically uses it
1. Confirm the SUT uses a simulated runtime or state the work needed to introduce one—DST without that plumbing is fiction.
2. Define the property assertions and oracles inside the simulation, not just outside.
3. Set a seed budget per scenario (e.g., 10k seeds per run).
4. Record seeds of any failures for deterministic replay.
5. Include time-warp scenarios (slow node, fast node), not just partition.
