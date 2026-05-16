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
