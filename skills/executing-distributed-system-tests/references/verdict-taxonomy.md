# Verdict Taxonomy

A test outcome is not three-valued (PASS / FAIL / INCONCLUSIVE) in a
distributed system. Three values collapse distinctions that matter:
"the chaos script ran cleanly and nothing crashed" is not the same as
"the linearizability checker consumed the full history under the
proven fault and saw no anomaly." This file defines the ten
verdict states the execute skill assigns at run end.

**Distinct from `finding-classification.md`.** That file classifies
the *type* of bug (TaxDC categories: timing / ordering / partition /
…). This file classifies the *outcome* of the test run.

**Budget tiers gate PASS verdicts.** A scenario whose run met only
the smoke budget (smallest config, shortest duration, single seed)
can produce at most `PASS-smoke`, never `PASS-hardening`, regardless
of oracle output. The hardening verdict requires the plan's declared
hardening-tier configuration, duration, seed count, and fault count
to have all been met. The release verdict, similarly, requires the
release-tier budget (or its explicit "not provided" disclosure to
have been resolved). See the plan template's per-scenario budget
fields.

## The ten states

### `PASS-smoke`

Happy path. No fault injected. Oracle ran on a complete history. No
anomaly.

Use as a sanity baseline only. A `PASS-smoke` is not evidence that
the system honours the claim under fault — it is evidence that the
test harness works at all. Multiple smokes in a row gives confidence
the harness is stable; it does not give confidence the SUT is robust.

### `PASS-hardening`

Full plan: the declared nemesis fired, landing evidence was captured,
the checker consumed a complete history covering the fault window, no
anomaly was detected.

This is the strongest PASS the skill produces. The §7d confidence
statement should lean on `PASS-hardening` verdicts; `PASS-smoke`
verdicts do not support hardening claims.

### `FAIL-reproducible`

The checker found a violation that re-runs deterministically on the
same seed / config / fault sequence.

File as a finding immediately. Apply `references/test-case-reduction.md`
and the SUT/harness/checker/environment classification before moving on.

### `FAIL-nondeterministic`

A violation was observed, but a re-run with identical inputs did not
reproduce it. The system *can* produce the violation under some
interleaving, but the test does not reliably surface it.

Flag for deterministic-simulation pursuit (see
`references/deterministic-simulation.md` in the design skill) or
longer-run statistical pursuit. Do not file as if it were
`FAIL-reproducible`; the reproducer story is missing.

### `INCONCLUSIVE-env`

The scenario could not run because a required capability is missing:
no `iptables`, no Docker, no `tc`, no `libfaketime`, no kernel feature.

The fix is to install or substitute. The plan committed to the
capability in its §6b environment-requirements section; the gap is
between plan and operator environment, not between plan and SUT.

### `INCONCLUSIVE-oracle-too-weak`

The scenario ran. Faults injected. History captured. But the chosen
checker cannot distinguish PASS from FAIL given what was captured.

Most common cause: a weak history (see `references/history-discipline.md`
in the design skill) — missing `fault_epoch`, missing `timeout_marker`,
missing `node_seen`. Fix the recorder before re-running.

Second most common cause: a checker mismatched to the model. Asking
a linearizability checker to validate a session-consistency claim
will produce false PASSes. Pick the right checker from
`oracle-patterns.md`.

### `INCONCLUSIVE-fault-not-proven`

The fault was scheduled. The injector reported success. But the
landing evidence signal declared in §7.M is absent or ambiguous in
the recorded data.

Common: iptables rule installed in the wrong chain, `tc` qdisc on
the wrong interface, Toxiproxy enabled but the SUT was already
talking direct to the backend, container restart immediately
reversed by an orchestrator.

This verdict means the scenario did not test what it claimed to
test. The PASS or FAIL of the oracle is irrelevant — the fault was
not proven to have landed, so the run does not move the confidence
needle.

### `PARTIAL-surface`

Surface-level checks ran cleanly — no error metrics, recovery
completed, final state looks normal — but the model-level checker
declared in §7.M did not run.

Cannot be quoted as hardening evidence. Can be quoted as "no obvious
regression" in the report. The §7d confidence statement should
disclaim that `PARTIAL-surface` scenarios did not falsify the claim
they were aimed at.

### `PARTIAL-model`

The model-level checker ran, but on a subset of operations the
scenario produced — e.g., per-key linearizability but not cross-key
serializability; per-replica reconciliation but not majority-quorum
consensus.

Surface the subset explicitly in the report so the reviewer can
decide whether the un-checked subset matters. `PARTIAL-model` is
stronger evidence than `PARTIAL-surface`; it is still not full
hardening.

### `NOT-RUN`

The arm (or scenario) was declared in the plan but was not attempted
in this session. The skill is being honest about deferred work: a
scenario or arm that the operator planned but the run did not reach.

Distinct from `INCONCLUSIVE-env`, which means "we tried and a
required capability was missing." `NOT-RUN` means "we did not try."
Common causes: time-bounded session ran out before reaching this
arm; the arm is gated behind a future release-tier budget; the
harness for this surface is not yet built.

The aggregation rule: any `NOT-RUN` or `PARTIAL-*` arm caps the
scenario-level verdict at `PARTIAL-surface`. The findings report names the
deferred arm explicitly in the Surface coverage table; it does not
silently fold into the aggregate.

## Decision tree (assign at run end)

```
Was the arm or scenario attempted at all?
├─ no, declared in the plan but never started → NOT-RUN
└─ yes
    Was `INCONCLUSIVE-env` triggered before the run started?
    ├─ yes → INCONCLUSIVE-env
    └─ no
        Did any FAIL register?
        ├─ yes
        │   ├─ reproducer obtained on re-run with same seed?
        │   │   ├─ yes → FAIL-reproducible
        │   │   └─ no  → FAIL-nondeterministic
        │   └─ (no other branches; FAIL takes precedence)
        └─ no
            Was the planned fault proven to land?
            ├─ no
            │   ├─ planned fault was scheduled? → INCONCLUSIVE-fault-not-proven
            │   └─ no fault planned (smoke run)  → continue below
            └─ yes (or smoke)
                Did the declared §7.M checker run?
                ├─ no  → PARTIAL-surface
                └─ yes (full)
                │   Did the history meet the checker's required fields?
                │   ├─ no   → INCONCLUSIVE-oracle-too-weak
                │   └─ yes
                │       Did the checker cover all ops the scenario produced?
                │       ├─ no  → PARTIAL-model
                │       └─ yes
                │           Was a fault active in the run?
                │           ├─ no  → PASS-smoke
                │           └─ yes → PASS-hardening
```

**Non-serious scenarios.** If §7.M was marked "not applicable" (the
scenario does not falsify a claim in the gated set), the tree's
checker-required nodes do not apply. Use the scenario's Oracle
field as the success criterion: smoke run → `PASS-smoke`; fault
active + Oracle passed → `PASS-hardening`.

The pre-execution branches at the top of the tree (NOT-RUN and
INCONCLUSIVE-env) are mutually exclusive. NOT-RUN means the arm or
scenario was declared in the plan but the session did not attempt
it. INCONCLUSIVE-env means the attempt was made but a required
capability was missing before the scenario could proceed.

## Findings-report column

The findings report's scenario-results table uses these ten values
in its `Verdict` column. The column header links here. The session-
level summary (DONE / DONE_WITH_CONCERNS / INCONCLUSIVE / FAIL /
BLOCKED) is computed from the per-scenario verdicts via the rules in
SKILL.md step 7.
