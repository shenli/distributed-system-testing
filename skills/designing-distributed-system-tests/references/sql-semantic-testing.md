# SQL Semantics and Transaction-Local Testing

Use for SQL engines, database compatibility layers, or changes to query,
type, constraint, transaction, or client-protocol behavior. For a narrow
change, select the affected cases; this is not a mandatory SQL campaign
for every stateful service.

Motivation: [Jepsen's ParticleDB analysis, 2026-10-31](https://jepsen.io/analyses/particledb-1.1.4)
found 96 issues, all but three without injected faults on a single node.
It used differential SQL generation and transaction-local checking in
addition to transactional histories. Testing stopped at 1.1.18.11.rc-22;
the subsequent fixes attributed to 1.1.19-rc.13 were vendor reports,
not Jepsen verification. The procedures below are recommendations for
designing our own tests, not claims of additional ParticleDB testing.

## Establish the contract and baseline

Record the artifact digest/build, client and checker versions, protocol,
startup flags, session settings, and durability mode for each test arm.
Preserve conflicting documentation with its version/date. A mode name,
alias, or successful SET statement does not establish its semantics:
capture effective settings when available and include a behavioral probe
that distinguishes the claimed guarantee. Record unavailable introspection
and ambiguous guarantees explicitly; do not resolve them by choosing the
strongest interpretation of a marketing label.

Start with a supported minimal deployment and one client, then concurrent
clients without faults, then targeted fault scenarios. A single-node
control says nothing about replication. Keep sequential failures separate
from concurrency failures so that malformed values or predicates do not
get misdiagnosed as isolation anomalies.

For each in-scope feature, include a valid case and a case that would fail
if the feature were missing or ignored. For example, a uniqueness test
needs a conflicting write as well as a successful insert. Calibrate the
oracle with a known-bad result/history; use a disposable mutant or known
broken build when available. Do not alter production state for calibration.

## Differential campaign

Choose an independent reference engine and pin its version and settings.
Generate a small supported grammar first, with deterministic seeds and
fresh equivalent state on both engines. Run each statement sequence
sequentially on both. Record statement results, errors, transaction state,
and subsequent reads, including after a failed statement or rollback.
A sequential reference execution is not an oracle for arbitrary concurrent
schedules; use transaction-local checks or an isolation checker there.

Define equivalence before running:

- Compare unordered results as **multisets**, preserving duplicate rows.
  Compare order only when specified, and account for ties. LIMIT/OFFSET
  needs a deterministic total order for exact cross-engine comparison.
- Preserve types, NULLs, column positions, and metadata. Do not collapse
  duplicate column labels into a map or stringify every value. Declare
  numeric tolerances only where the contract allows them; never round
  exact integers or decimals to conceal a discrepancy.
- Align collation and other relevant session settings. Classify documented
  dialect differences separately from incorrect state or violated claims.
  Compare error categories and post-error effects; byte-identical error
  prose is rarely the contract.

Bound shrinking per failure signature. Save the original case before
minimization and retain schema, seed, bindings, configuration, and outputs.
Deduplicate known divergences so that one compatibility mismatch does not
consume the campaign. Any exclusion needs a narrow signature, rationale,
version scope, retained reproducer, and recheck condition. Report excluded
case counts and continue independent invariant checks; do not suppress
all errors or all queries touching the same feature.

## Transaction-local model and semantic coverage

For single-client cases, maintain a simple independent table model and
compare after each statement, at commit, after rollback, and from a fresh
session. Under concurrency, track only facts justified by the declared
isolation level, observed reads, and the transaction's own operations.
Do not demand repeatable reads under a model that permits nonrepeatable
reads, or assume every transaction shares one initial global state.

Select relevant combinations from this matrix and name the combinations
left untested:

| Dimension | Cases and oracle targets |
|---|---|
| Access path | Primary-key lookup, secondary predicate, scan, aggregate, join; compare equivalent observations of the same logical state in a stable snapshot or quiescent single-client case |
| Mutation | Insert, update matched/unmatched rows, delete, delete then reinsert, column reordering; verify actual effects as well as affected-row counts |
| Constraints | Valid and invalid inserts and updates, multi-row conflicts, NULL handling; check statement atomicity according to the contract |
| Values | Empty/zero, NULL, integer bounds, exact decimal vs float, supported special floats, padded strings and collation-sensitive text |
| Lifecycle | Autocommit vs explicit transaction, error then commit, rollback, repeated begin where supported, drop/recreate with clean schema and data |
| Interface | Each supported driver/protocol; literal vs bound parameters, repeated execution across automatic prepare thresholds, reconnect/session reset |

Use distinguishable keys and values so a wrong-row response cannot look
correct by coincidence. Vary physical encodings and access paths while
keeping the same logical workload. Specify the transformation's validity
conditions rather than assuming every SQL rewrite is equivalent.

## Evidence and limits

Record generated/executed/checked counts, useful nonempty cases, successful
commits, excluded divergences, and coverage per selected matrix arm.
Preserve raw typed observations before adapters normalize them. Bound
client waits so observers can sample the intended window, recording
timeouts and hidden retries as ambiguous outcomes rather than aborts.

For storage scenarios, require evidence that the intended table/index,
flush, or compaction path actually ran. If a supported test configuration
can lower a threshold, record it and retain a production-like follow-up;
a WAL-only run does not cover on-disk table recovery. See
`crash-recovery-and-upgrade.md` for fault-model distinctions.

A clean result covers only the tested grammar, configurations, interfaces,
and histories. Preserve minimal regressions, rerun them on the exact fixed
artifact, and rerun nearby variants before expanding the confidence claim.
