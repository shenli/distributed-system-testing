# Formal Methods (TLA+ and friends)

**Last verified:** 2026-05-16

## When to reach for it
Use formal methods when designing a protocol or invariant where getting it wrong is expensive: consensus, replication, leasing, transaction commit, crash-recovery workflows. The protocol must be small enough to model (typically single-component or interaction between a few components).

## What it detects well
- Protocol-level bugs: livelock, ordering violations, safety invariant violations.
- Mixed-version protocol issues and reconfiguration edge cases.
- Implicit assumptions made explicit and violated by counterexample.
- Fairness and liveness properties at design time, before implementing.

## What it misses
- Implementation bugs (the model is not the code; off-by-one errors, race conditions in Go goroutines etc.).
- Performance and wall-clock behavior.
- Anything outside the modelled state space (unmodelled message types, new fault classes).

## Concrete tools
- `TLA+` / `PlusCal` — Lamport's declarative specification language with TLC model checker and TLAPS prover — https://lamport.azurewebsites.net/tla/tla.html
- `Alloy` — bounded model checker with SAT-solver backend — https://alloytools.org
- `P` — state-machine-focused language specifically for distributed protocols — https://github.com/p-org/P
- `Coq` / `Verdi` — heavyweight frameworks for machine-verified distributed systems — https://github.com/uwplse/verdi

## Papers to cite
- "How Amazon Web Services Uses Formal Methods" (Newcombe et al., CACM'15) — industrial adoption at AWS S3 and DynamoDB — https://cacm.acm.org/magazines/2015/4/184701-how-amazon-web-services-uses-formal-methods/fulltext
- "Using Lightweight Formal Methods to Validate a Key-Value Storage Node in Amazon S3" (Bornholt et al., SOSP'21) — pragmatic approach combining property testing with model checking — https://www.amazon.science/publications/using-lightweight-formal-methods-to-validate-a-key-value-storage-node-in-amazon-s3

## Cost / wall-clock signal
Model-writing is days to weeks of human time; model-checking runs are minutes to hours depending on state-space size (exponential in unmodelled variables).

## How a plan typically uses it
1. State the invariants you must preserve and the liveness goals you want (no deadlock, fairness).
2. Model only the relevant slice—abstract storage, networking, and other implementation details.
3. Check both safety and liveness, documenting fairness assumptions (who moves when, who can stutter).
4. Keep the model in version control and re-check on every protocol-level change.
5. Write a mapping document that connects model state back to implementation state so the gap is visible and reviewable.
