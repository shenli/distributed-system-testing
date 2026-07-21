# Findings: {{plan_slug}} @ {{UTC_timestamp}}

**Plan:** {{path/to/plan.md}}
**Commit under test:** {{git_sha}}
**Session dir:** {{absolute_path}}

## Summary

One paragraph: what was run, what was found, what was inconclusive.
State the headline result first — "1 oracle violation found", or
"all 5 scenarios passed with stated coverage", or "1 scenario could
not run because of {{reason}}".

## Scenario results

One row per scenario arm. If a scenario has multiple drivers (an
in-process test plus an external workload, for example), give each
its own row with `S1 Driver A` / `S1 Driver B` IDs so verdicts and
evidence do not get conflated.

For boundary-style scenarios with §7.M.S arms, each arm gets its own
row using an `S<n>/arm` ID (e.g., `S5/api`, `S5/admin`). The
scenario's aggregate row uses the parent ID (e.g., `S5`) and follows
the downgrade rule: any `NOT-RUN` or `PARTIAL-*` arm caps the
aggregate verdict at `PARTIAL-surface`.

| ID | Scenario | [Verdict][vt] | Oracle | Oracle execution evidence | Nemesis landing evidence | Artifact link |
|---|---|---|---|---|---|---|
| S1 | {{name}} | PASS-hardening / PASS-smoke / FAIL-reproducible / FAIL-nondeterministic / INCONCLUSIVE-env / INCONCLUSIVE-oracle-too-weak / INCONCLUSIVE-fault-not-proven / PARTIAL-surface / PARTIAL-model / NOT-RUN | {{property checked}} | {{e.g. "assertion fired 1000/1000", "Elle processed 4217 ops 0 anomalies"}} | {{e.g. "iptables counter 0→14,712 over the injection window; SUT raft log shows leader-lost", or "n/a — no fault (PASS-smoke)"}} | {{link to session log entry + artifacts}} |

[vt]: ../references/verdict-taxonomy.md

## Findings

### Finding F1: {{short title}}

- **Scenario:** S{{n}}
- **Hypothesis addressed:** H{{n}} from the plan
- **What happened:** one paragraph, factual.
- **Reproducer:** the minimal fault sequence that reproduces. If you
  did not minimize, say so and why.
- **Reduction classification:** SUT / harness / checker / environment
  (pick exactly one per the "Classify blame before filing" section of
  `references/test-case-reduction.md`). "Unknown — pending re-run on
  alternative harness/host" is acceptable as an interim value.
- **Classification:** timing / ordering / partition / crash-recovery
  / config / upgrade / fault-handling / performance (pick the closest
  from `references/finding-classification.md`).
- **Evidence:** log excerpts, trace IDs, metric snapshots, op history
  files. Link to artifacts under the session dir.
- **Hypothesised root cause:** one paragraph; which subsystem owns it.
- **Suggested next action:** one sentence.

(Repeat per finding.)

## Coverage summary

For each hypothesis in the plan, state which scenarios exercised it
and the result. Hypotheses that were not exercised must be listed
with the reason (out of scope, blocked by missing tooling, deferred).

## Surface coverage (boundary and fairness scenarios)

For scenarios with a §7.M.S block declared in the plan, list the
per-arm execution and verdict here. Aggregate rows are derived from
the per-arm verdicts via the downgrade rule.

| Scenario / arm | Surface         | Planned | Executed | Verdict           | Downgrade reason                       |
|---|---|---|---|---|---|
| S5/api    | API                 | yes     | yes      | PASS-hardening    | —                                      |
| S5/sdk    | SDK                 | yes     | yes      | PASS-smoke        | —                                      |
| S5/export | export pipeline     | yes     | no       | NOT-RUN           | export harness not yet built           |
| S5/admin  | admin API           | yes     | yes      | PARTIAL-surface   | no negative controls captured this run |
| S5        | (aggregate)         |         |          | PARTIAL-surface   | 1 NOT-RUN arm + 1 PARTIAL arm          |

**Missing surfaces this run did not cover:** name each one, with
the reason it was not covered. If every planned surface was
covered, render this line as "All planned surfaces were executed."

If the plan has no §7.M.S scenarios, render this section as a
single line: "No boundary or fairness scenarios in this plan."
Explicit-empty is the signal.

## Release-budget disclosures

Lift every scenario whose `Release budget` field in the plan was
filled with `not provided — <reason>. Revisit when: <…>.` into the
table below, carrying the reason and revisit-condition verbatim from
the plan.

| Scenario | Reason release budget was not provided | Revisit when                                           |
|---|---|---|
| S5      | SUT has no statistical-gate harness yet | the perf-bench rig in tools/agentdb-bench supports multi-day soak runs |
| S9/sdk  | release tier is the SDK team's plan      | their plan covers this claim                          |

If every scenario declared a concrete release budget, render this
section as a single line: "All scenarios declared a concrete release
budget." Explicit-empty is the signal.

## Adequacy assessment vs the plan's coverage argument

The plan's §7b ("Coverage adequacy argument") committed to a set
of claim → threat → scenario mappings. This section assesses how
well the actual run honoured them.

| Claim | Plan's adequacy argument | What actually ran | Adequacy after this run |
|---|---|---|---|
| C1 | Sa+Sb+Sc cover threats T1/T2/T3 | Sa PASS, Sb PASS, Sc INCONCLUSIVE (no asym-partition env) | Two of three threat dimensions falsifiable; T3 (replica divergence under asym partition) NOT exercised this run |
| ... | ... | ... | ... |

For any row where "what actually ran" ≠ "plan's argument" (e.g.
a scenario was INCONCLUSIVE, or its oracle was weaker than
planned), the residual uncertainty after this run is larger than
the plan declared. Surface those rows explicitly so the reviewer
can decide whether to accept the gap or re-run with the missing
piece.

## Confidence delta

A short paragraph that answers: given the plan's §7d confidence
statement, what should a reviewer believe MORE / LESS after
seeing this report?

- **More:** which claims are now better-validated than before.
- **Less:** which claims are now less-validated than the plan
  hoped (scenarios that went INCONCLUSIVE or surfaced new
  uncertainty).
- **Unchanged:** which the run did not move at all.

This is what the stakeholder reads — the rest of the report is
the supporting evidence trail.

## Green-but-broken check

State the result of each red-flag check from
`references/green-but-broken-red-flags.md`. A scenario marked PASS
without these checks completing is not actually PASS.

Per scenario, record the result of all ten numbered red-flag checks
from `references/green-but-broken-red-flags.md` (cite evidence per
checked item):

- [ ] 1. Workload generator produced the expected ops/sec throughout
- [ ] 2. Oracle ran on every intended cycle (cite the op count it
  consumed)
- [ ] 3. Faults verifiably injected (cite the landing-evidence signal
  declared in §7.M)
- [ ] 4. Fault did not silently no-op (rule out wrong chain / wrong
  interface / orchestrator-reversed restart)
- [ ] 5. Clock skew did not silently mask the timing assertion
- [ ] 6. Run duration met the budget tier being claimed (cite Smoke /
  Hardening / Release tier met)
- [ ] 7. No silent error suppression (grep'd SUT log for swallowed
  panics)
- [ ] 8. Recovery completed (post-fault, SUT returned to nominal —
  not "stayed up degraded")
- [ ] 9. Baseline comparison is fair (re-baselined if the harness
  changed)
- [ ] 10. Statistical claims replicated (one PASS is not PASS)

Per-scenario weak-oracle audit — record the result of all fourteen
items from the "Weak oracles" section of
`references/green-but-broken-red-flags.md` (any unchecked → not
eligible for `PASS-hardening`):

- [ ] Oracle is not "final state only"
- [ ] Oracle is not "logs only"
- [ ] Oracle is not "health checks only"
- [ ] If failover-based, more than one failover was exercised
- [ ] Not "no-error metrics" alone (SUT did not swallow the errors)
- [ ] Not "short runs" alone (duration justified by the claimed tier)
- [ ] At least one asymmetric partition variant covered (not
  symmetric-only) if applicable
- [ ] Recorder caught client-library-hidden retries (or N/A — the
  library does not hide retries)
- [ ] Timestamps are monotonic, not wall-clock
- [ ] More than one cluster topology run if the claim is "any
  quorum-respecting size"
- [ ] At least three PRNG seeds tried for any statistical claim
- [ ] Not single-surface only — boundary claims cover admin / export
  / SDK / background-job / observability paths (or N/A — not a
  boundary claim)
- [ ] Negative controls present, not positive-control only (or N/A —
  not a boundary claim)
- [ ] Fairness oracle has a per-group breakdown, not aggregate-p99
  only (or N/A — not a fairness claim)
