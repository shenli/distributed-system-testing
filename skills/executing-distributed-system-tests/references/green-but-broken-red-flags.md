# Green-but-Broken Red Flags

A scenario that "passed" without these checks completing did not
actually pass. Run this list before declaring a scenario PASS.

1. **Workload actually generated the intended load.** Throughput
   chart shows the expected ops/sec for the expected duration.
   If the generator silently stopped or rate-collapsed, the test
   tells you nothing.
2. **Oracle actually ran on every intended cycle.** Cite the
   number of times the property fired or the number of ops the
   linearizability checker consumed.
3. **Faults actually injected.** Cite evidence from the injector
   side AND from at least one victim side. "Toxiproxy was enabled"
   is not evidence; "RPC timeout count went from 0/s to 142/s
   over the injection window" is.
4. **Fault did not silently no-op.** Common: iptables rule in
   the wrong chain, tc qdisc on the wrong interface, container
   restart that the orchestrator immediately reversed.
5. **Clock skew did not mask a timing assertion.** If the
   assertion depends on monotonic time, the timing source must
   be monotonic — not wall clock.
6. **Run duration met the plan's exit criterion.** Short runs
   often miss bugs that only manifest after sustained pressure.
7. **No silent error suppression.** Grep the SUT log for
   exception / panic / "Internal error" lines that the oracle
   did not see because they were swallowed by a higher layer.
8. **Recovery completed.** Post-fault, the SUT returned to
   nominal — not "stayed up but in degraded mode that the oracle
   didn't recognize as degraded".
9. **Baseline comparison is fair.** Comparing today's run to
   last week's baseline is fine only if nothing else has changed
   in the harness. Re-baseline when the harness changes.
10. **One PASS != PASS.** If the test is statistical, replicate.

## Weak oracles — do not trust these alone

The checks above guard against "the test never ran." This list guards
against "the test ran but the oracle could not tell PASS from FAIL."
Each item below is a signal that, **in isolation**, is too weak to
prove a hardening claim. Pair every item below with a checker from
`oracle-patterns.md` if you need to claim more than "no obvious
regression."

- **Final state only.** "Read the database at the end, see if it
  looks right." Misses every transient anomaly the system recovered
  from. Pair with: `oracle-patterns.md` §10 (no-lost-ack), §12
  (invariant-over-final-state).
- **Logs only.** Greppable error lines are noise; the absence of
  errors is not the presence of correctness. Pair with: an actual
  property checker from `oracle-patterns.md` §1–§13.
- **Health checks only.** `/health` returning 200 is a liveness
  check, not a correctness check. Pair with: any consistency
  checker.
- **Single successful failover.** One leader kill + recovery is a
  smoke test, not a hardening test. Pair with: repeated failover
  + reconciliation-across-replicas (`oracle-patterns.md` §13).
- **No-error metrics.** "Error rate stayed 0" — except the SUT
  swallowed the errors. Pair with: client-side history with
  `timeout_marker` set, then a no-lost-ack check.
- **Short runs.** 60-second chaos windows miss bugs that only
  manifest after sustained pressure (compaction races, slow
  leaks, queue overflow). Plan exit criteria must justify the
  duration.
- **Symmetric partitions only.** Real production partitions are
  often one-way or asymmetric. Cover at least one asymmetric
  variant. See `fault-injection-howto.md` Network faults table.
- **Client libraries that hide retries.** If the library transparently
  retries, the in-process history undercounts the network ops the
  SUT actually saw. Pair in-process recording with a network-side
  tap (see `history-discipline.md` in the design skill, Recording
mechanisms section).
- **Wall-clock timestamps.** Two clients with skewed clocks make
  a correct system look inconsistent. Use a monotonic clock source
  per recorder, or use server-receive timestamps as the canonical
  order.
- **One cluster topology.** A bug that requires N=5 will not surface
  at N=3. Run the plan against at least two topologies if the claim
  is "linearizable for any quorum-respecting size".
- **One seed.** A single PRNG seed is one interleaving. Treat any
  PASS-hardening claim built on a single seed as `PARTIAL-model`
  until you have at least three seeds with the same verdict.
