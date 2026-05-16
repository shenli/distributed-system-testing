# Catalog Index — Pick a Technique by Symptom

This page is for the agent to skim BEFORE opening any specific technique
reference. Match the suspicion on the left to the technique on the right;
read that file for "when / what it catches / what it misses / tools / papers
/ cost". Most plans use 2–4 techniques in combination, not one.

## Before picking techniques: walk the pitfall catalog

Open [`common-distributed-systems-pitfalls.md`](common-distributed-systems-pitfalls.md)
first. It lists 16 failure modes that recur across the Jepsen analyses
corpus and adjacent literature (lost updates, stale reads, replica
divergence, linearizability under skew, lost-ack vs. lost-commit,
reconfiguration races, crash-recovery divergence, schema migration,
sequence-number collision, watch loss/duplication, cross-shard non-
atomicity, clock-skew safety, auth/quota divergence, lease expiry
under contention, idempotency replay bypass, outbox head-of-line block).

For each pitfall, decide whether it applies to your SUT. If yes, you have
a hypothesis to write into the plan's §4 — each pitfall entry includes a
hypothesis template you can paste-adapt. Then come back to this index to
pick the technique that exercises it.

## Pick a technique by symptom

| Suspect this could break... | Reach for... |
|---|---|
| Linearizability / serializability under concurrent ops + faults | `jepsen-and-elle.md` |
| Crash mid-operation, replay, or fsync loss correctness | `crash-recovery-and-upgrade.md` |
| Non-deterministic interleavings that "passed in CI" | `deterministic-simulation.md` or `fuzzing.md` |
| Partial network partitions, asymmetric loss, clock skew | `chaos-and-fault-injection.md` |
| Input parser / state-machine bugs from unexpected bytes or call sequences | `fuzzing.md` |
| Algorithm-level correctness of a protocol you wrote | `formal-methods-tla.md` |
| Whole-system invariants that should hold across many inputs | `property-and-metamorphic.md` |
| Tail latency, head-of-line blocking, throughput collapse under load | `performance-and-benchmarking.md` |
| Mixed-version cluster, rolling upgrade, downgrade, schema migration | `crash-recovery-and-upgrade.md` |
| Limping (degraded but not dead) node | `chaos-and-fault-injection.md` + `performance-and-benchmarking.md` |
| Config typos / defaults silently changing behavior | `property-and-metamorphic.md` (config space) |

## How to combine

- **Jepsen + chaos:** Jepsen drives a workload + oracle; the chaos layer
  injects realistic faults. Use both together for distributed correctness.
- **DST + property:** deterministic simulation gives reproducibility;
  property assertions give the oracle. Both layered is how FoundationDB
  and TigerBeetle catch most regressions before merge.
- **Formal + tests:** TLA+ proves the design; tests catch the
  implementation's drift from the design. One does not replace the other.
- **Fuzzing + sanitizers:** fuzz inputs are only as useful as the bug
  detectors running underneath (ASan, MSan, UBSan, race detectors).

## Quick "do not skip" rules from the literature

- Most production failures come from a small number of underlying causes
  that simple testing can catch — see "Simple Testing Can Prevent Most
  Critical Failures" (Yuan et al., OSDI'14). Always test error-handling
  paths, not just happy paths.
- Random testing is effective at finding partition-tolerance bugs even
  without deep symbolic reasoning — see "Why Is Random Testing Effective
  for Partition Tolerance Bugs?" (Majumdar et al., POPL'18). When in
  doubt, run more random scenarios.
- Redundancy does not imply fault tolerance — replicas can all mishandle
  the same corrupted byte. Always test the fault, not just the failover.
