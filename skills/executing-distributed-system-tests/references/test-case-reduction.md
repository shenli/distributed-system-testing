# Test-case Reduction

A 3-hour distributed run that hits a bug is hard to act on; reduce
it to the smallest fault sequence that still reproduces.

## When to reduce
- Always for FAIL findings before filing.
- Not for INCONCLUSIVE — fix the inconclusiveness first.

## Approach (after Scott et al., MCS, NSDI'16)

1. **Capture the full event log:** all faults, all client ops, with
   timestamps, in order.
2. **Bisect by halves:** drop the first half of events. Still
   reproduces? Recurse on the second half. Doesn't reproduce?
   Keep first half and recurse.
3. **Drop one event at a time** from the reduced set; if the bug
   still reproduces without that event, leave it out.
4. **Stop when:** removing any single remaining event makes the
   bug stop reproducing.
5. **Record the budget:** "reduced from N events to k events over
   M re-runs" goes in the finding.

## When to give up
- Bug requires a rare interleaving that does not reproduce on
  every run. Record what you tried, mark "non-deterministic" in
  the finding, and either change tactic (DST / deterministic
  scheduler) or accept the larger reproducer.

## Avoid over-reducing
- Reduction can drop events that are causally required but happen
  to not change the outcome on a given seed. Keep at least one
  event from each fault category in the reproducer if the bug
  depends on the category.

## Classify blame before filing

A reduced reproducer is not a bug report yet. The same reproducer
can mean different things depending on what's broken — and sending
the bug to the wrong place wastes engineering time. After reduction,
classify the bug into exactly one of:

- **SUT bug.** The system under test honoured the workload but violated
  the claim. Goes to the SUT team.
- **Harness bug.** The workload generator, fault injector, or scheduler
  introduced an artifact the SUT did not actually produce. Goes back to
  the test-harness queue.
- **Checker bug.** The oracle reported a violation that the data does
  not in fact support — wrong model, off-by-one in the property,
  mishandled timeout marker. Goes back to the checker queue.
- **Environment bug.** Kernel scheduler quirk, driver bug, filesystem
  config, cgroup limit, virtualization artifact. Goes to the
  infrastructure / platform queue.

### Representative tells

| Class | Tells |
|---|---|
| SUT bug | Reproducer survives swapping the workload generator. Reproducer reproduces against the SUT's stated single-binary smoke harness. Bug aligns with a known TaxDC category. |
| Harness bug | Reproducer disappears when the workload generator is replaced with a different one targeting the same SUT API. Logs show the generator sent operations the SUT API does not accept (or in the wrong format). |
| Checker bug | The "violation" makes sense if you allow a documented relaxation the checker does not encode (e.g., bounded staleness allowed; checker assumed linearizable). Re-running the checker on a known-good history also flags a violation. |
| Environment bug | Reproducer disappears on a different host / kernel / FS / cloud-region. Bug timing aligns with host-level events (cgroup throttle, hypervisor pause, NTP step). |

### Recording the classification

Each FAIL finding's report block carries a `Reduction classification:`
field (one of SUT / harness / checker / environment) immediately before the
TaxDC `Classification:` field. The two are orthogonal — TaxDC says
*what kind* of bug; reduction classification says *which component*
holds the bug. Both are required.

A classification you are not yet sure about is `Reduction
classification: unknown — pending re-run on alternative harness/host`.
That is honest. A guess that turns out wrong is not.
