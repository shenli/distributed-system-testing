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
