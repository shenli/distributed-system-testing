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

| ID | Scenario | [Verdict][vt] | Oracle | Oracle execution evidence | Nemesis landing evidence | Artifact link |
|---|---|---|---|---|---|---|
| S1 | {{name}} | PASS-hardening / PASS-smoke / FAIL-reproducible / FAIL-nondeterministic / INCONCLUSIVE-env / INCONCLUSIVE-oracle-too-weak / INCONCLUSIVE-fault-not-proven / PARTIAL-surface / PARTIAL-model | {{property checked}} | {{e.g. "assertion fired 1000/1000", "Elle processed 4217 ops 0 anomalies"}} | {{e.g. "iptables counter 0→14,712 over the injection window; SUT raft log shows leader-lost", or "n/a — no fault (PASS-smoke)"}} | {{link to session log entry + artifacts}} |

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

Per scenario (cite evidence per checked item):

- [ ] Workload generator produced the expected ops/sec throughout
- [ ] Oracle ran on every intended cycle (cite the op count it
  consumed)
- [ ] Faults verifiably injected (cite the landing-evidence signal
  declared in §7.M)
- [ ] Clock skew did not silently mask the timing assertion
- [ ] Run duration met the plan's exit criterion
- [ ] No silent error suppression (grep'd SUT log for swallowed
  panics)
- [ ] Recovery completed (post-fault, SUT returned to nominal —
  not "stayed up degraded")

Per-scenario weak-oracle audit (any unchecked → not eligible for
`PASS-hardening`):

- [ ] Oracle is not "final state only"
- [ ] Oracle is not "logs only"
- [ ] Oracle is not "health checks only"
- [ ] If failover-based, more than one failover was exercised
- [ ] Recorder caught client-library-hidden retries (or N/A — the
  library does not hide retries)
- [ ] Timestamps are monotonic, not wall-clock
- [ ] At least one fault-set variant (asymmetric, mixed-version,
  etc.) was covered if applicable
- [ ] At least three PRNG seeds tried for any statistical claim
