# Finding Classification

Use this single-page taxonomy (after TaxDC, Leesatapornwongsa et al.,
ASPLOS'16) to label every finding. Consistent labels make findings
searchable across runs and across projects.

**Two orthogonal axes.** This file classifies the **type of bug**
(timing / ordering / partition / …) once a finding exists. It does
**not** classify the **test outcome** — for that, see
`verdict-taxonomy.md` (PASS-smoke / PASS-hardening / FAIL-reproducible
/ FAIL-nondeterministic / INCONCLUSIVE-env / INCONCLUSIVE-oracle-too-weak
/ INCONCLUSIVE-fault-not-proven / PARTIAL-surface / PARTIAL-model /
NOT-RUN).
Nor does it classify the **component** holding the bug — for that,
see the "Classify blame before filing" section of
`test-case-reduction.md` (SUT / harness / checker / environment).
A finding has all three: a verdict (from the run), a reduction
classification (after reduction), and a TaxDC type (the categories
below).

## Top-level category (pick one)

- **Timing** — depends on relative timing of events (slow node,
  GC pause, scheduling delay). Often non-deterministic.
- **Ordering** — incorrect handling of message / event order.
- **Partition** — partial / asymmetric / full network split.
- **Crash-recovery** — incorrect state after process death + restart.
- **Upgrade** — mixed-version cluster or migration bug.
- **Config** — bad / default / changed configuration silently
  changes behavior.
- **Fault-handling** — bug *in* the error-handling code path itself
  (most common per Yuan et al. OSDI'14).
- **Performance** — correct but slow; tail / fairness / throughput.

## Secondary tags (any that apply)

- **Replication** — propagated to / between replicas.
- **Leadership / election** — election timing or split-brain.
- **Idempotency / dedup** — duplicate-application bug.
- **Durability** — lost data after acknowledged write.
- **Liveness** — system never recovers.
- **Safety** — system returns wrong answer (treat as P0 by default).

## Severity (engineering judgement, not part of TaxDC)

- **P0** — data loss, data corruption, or safety violation.
- **P1** — availability loss; recovery requires operator action.
- **P2** — observable degradation; auto-recovers.
- **P3** — no user impact; minor anomaly worth noting.
