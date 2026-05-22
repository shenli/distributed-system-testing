# Test Plan: {{slug}}

**Date:** {{YYYY-MM-DD}}
**Plan mode:** change-scoped | project-wide
**SUT:** {{system_name}}
**Change under test:** {{commit / PR link, OR "N/A — project-wide"}}
**Plan author:** {{agent or human}}
**Status:** draft | reviewed | executed

## 0. Architectural summary

A self-contained one-pager (≤ 30 lines) summarising the system as
it actually exists. A reviewer who has never seen the codebase
should be able to follow the rest of the plan from this section
alone. Cover:

- **Major components** (one line each, what they own).
- **Data flow** through the system for the canonical request.
- **Where state is durable** (and which backend stores which kind
  of state).
- **Where consensus runs** and what protocol.
- **Trust / tenancy boundaries.**
- **A picture** (text-art block diagram works fine if not generated).

This is NOT the technique catalog (catalog is reference material).
This is the system as deployed, written so a reviewer can spot
"you have nothing exercising the storage→index handoff" — a gap
that a flat scenario list would hide.

## 1. Scope

**Change-scoped:** one paragraph — what does the change add / modify /
remove, and which subsystems does it touch?

Files touched:
- `path/to/file` — what changed
- `path/to/file` — what changed

**Project-wide:** what slice of the project is in scope (which
crates / services / surfaces) and what is explicitly out of scope
(adapters, demo apps, deprecated paths). A project-wide plan that
tries to cover everything covers nothing well.

## 1b. Claims under test

The spine of the plan. List every guarantee the SUT promises its
users — extracted from docs, API reference, code comments, error
types, existing test names. Categorise (safety / liveness /
durability / performance-SLO / operational / idempotency /
isolation / ordering / membership /
boundary / fairness). Mark inferred claims as `(inferred)`.

The two newest categories disambiguate two distinct concepts:

- `isolation` means consistency-isolation anomalies (G2-item,
  serializability, Elle-detectable read/write anomalies). Tested
  with checkers from `oracle-patterns.md` §1 and §3.
- `boundary` means access-boundary semantics: tenant isolation,
  authz, namespace, routing, multi-protocol access, compatibility
  across API surfaces. Subsumes tenancy / authz / namespace /
  routing — they do not appear as separate categories. Tested per
  the surface-decomposition discipline (§7.M.S; see
  `references/boundary-and-isolation-testing.md`).

`fairness` covers per-group performance and noisy-neighbor
isolation. Group can be tenant, shard, queue, partition, region,
priority class, user, table, or workload class.

| ID | Claim | Category | Source | Inferred? |
|---|---|---|---|---|
| C1 | "every acknowledged append is durable and replayable after crash" | durability | docs/architecture/...md | no |
| C2 | "same idempotency key never produces two committed effects" | idempotency | code: `IdempotencyConflict` error | partial — docs are silent |
| C3 | "leader election completes within 5s of crash" | liveness | (inferred from `STALE_NODE_THRESHOLD_MS`) | yes |
| ... | ... | ... | ... | ... |

Every hypothesis in §4 and every scenario in §7 must reference at
least one claim by ID. Untethered hypotheses produce ceremonial
scenarios; drop them or surface the missing claim in §1c.

## 1c. Missing claims discovered

Behaviors the implementation relies on for which no documented or
inferred claim exists. Each row is a signal that docs and code
have drifted — surfacing them is one of the highest-value outputs
of this plan. Often these become the highest-priority new claims
for the maintainer to either document or remove.

| ID | Behavior the code relies on | Source / evidence | Suggested action |
|---|---|---|---|
| M1 | Unicode normalisation policy for text search | `agentdb-index` code path; no test asserts NFC vs NFKC | document the policy and add a property test |
| M2 | ... | ... | ... |

If §1c is empty, that itself is worth stating: "the docs and code
appear aligned on every behavior the plan exercised." Don't leave
the section out — explicit emptiness is the signal.

One paragraph each:
- **Tenancy / isolation:** how is multi-tenancy enforced?
- **Persistence:** what is durable, when, with what fsync/replication
  contract?
- **Replication / consensus:** which protocol, quorum size, leadership?
- **Ordering:** what ordering guarantee is exposed to clients?
- **Network boundaries:** which RPCs / streams are in scope?
- **Retry / idempotency contract:** what does the client retry, and what
  is the server's deduplication strategy?
- **Observability:** what logs, metrics, traces exist that an oracle can
  consume?

## 3. Existing test inventory (project-wide only — skip for change-scoped)

| Test / harness | Subsystem | Invariant pinned | Failure modes it catches |
|---|---|---|---|
| `path::test_name` | append | "no duplicate seq under retry" | retry storm, restart-then-retry |
| `ops/smoke/scripts/foo.sh` | cluster | recovery time under leader kill | leader crash |
| ... | ... | ... | ... |

Group by subsystem. This becomes the left column of the coverage
matrix in §5.

## 4. Failure-mode hypotheses

Number each so scenarios can link back.

H1. **{{title}}** — {{one-sentence statement of what could go wrong}}
   - Could falsify: C1, C3
   - Suspected because: {{reason — code path, prior bug, paper, intuition}}
   - Subsystem: {{name}}
H2. ...

Cover at minimum: correctness, durability, liveness, partial failure,
idempotency/replay, upgrade/rollback, configuration, performance/fairness.
If a category is genuinely N/A for the plan's scope, say so explicitly.

In project-wide mode group hypotheses by subsystem so the coverage
matrix below stays readable.

## 5. Coverage matrix (project-wide only — skip for change-scoped)

One row per claim × hypothesis pair. Sort by claim severity × gap,
highest at top.

| Claim | Hypothesis | Existing tests | Verdict | Gap kind | Recommended scenario |
|---|---|---|---|---|---|
| C1 | H1 | (none) | not covered | no test | S1 |
| C2 | H2 | `test::foo` | partial | oracle too weak | S2 |
| C3 | H3 | `test::bar`, `test::baz` | covered | — | — |
| ... | ... | ... | ... | ... | ... |

For very large systems (50+ claims or 100+ hypotheses), split this
into two tables: a per-claim summary (one row per claim, rolled-up
verdict + count of failing hypotheses) followed by the per-
hypothesis detail above. The summary is the maintainer's at-a-
glance view; the detail keeps the gap-kind granularity.

Verdict legend: covered / partial / not covered.
Gap-kind legend: no test / shallow test / oracle too weak / no
fault-injection variant / no scale variant.

## 6. Technique selection

For each chosen technique, state:
- **Hypotheses it addresses:** H1, H3, ...
- **What it would catch that other techniques miss**
- **Reference:** `references/{{technique}}.md`
- **Cost / wall-clock estimate**

## 6b. Environment requirements

What the executing skill needs on the test box to run this plan.
The executing skill reads this list at its capability-probe step
and either guides the operator through install or marks dependent
scenarios INCONCLUSIVE.

| Requirement | Version floor | Used by scenarios | Install hint |
|---|---|---|---|
| docker + docker compose | 2.20+ | S1, S2, S5 | `apt-get install docker.io docker-compose-plugin` |
| iptables (Linux) | any | S6 (asymmetric partition) | usually pre-installed; needs `CAP_NET_ADMIN` |
| Postgres 14+ | 14 | S4, S11 | brought up by SUT's compose stack |
| MinIO / S3-compatible | any | S5, S9 | brought up by SUT's compose stack |
| Go 1.21+ | 1.21 | S6, S8 (perf, workload) | `apt-get install golang-go` or asdf |
| ... | ... | ... | ... |

For each requirement, name the scenarios that depend on it so the
executing skill knows what to do if it's missing. Mark "brought up
by SUT" when the requirement is bundled in the SUT's compose /
make / nix setup rather than something the operator installs.

## 7. Scenarios

Each scenario closes one or more rows in §5 (project-wide) or pins
one or more H-rows in §4 (change-scoped).

Name each scenario after the claim it falsifies, not its setup —
`linearizable_per_session_under_partition` beats "3-node cluster
with chaos."

Each scenario is an **executable spec**. The prose declares
WHAT to test; the `Target test file` + `Skeleton` declare WHERE
the test lives and HOW it scaffolds. The executing skill (in
author mode) writes the skeleton to the target path, fills in
the workload / fault / oracle bodies from the prose, compiles,
and runs.

### Scenario S1: {{claim-shaped name, e.g. linearizable_append_under_partition}}
- **Falsifies if it FAILs:** C1, C3 (claim IDs from §1b)
- **Closes:** H1, H2 (rows in §5)
- **Technique:** {{from §6}}
- **Workload:** {{generator, rate, key/op distribution, duration}}
- **Faults:** {{schedule — what is injected, when, for how long}}
- **Oracle:** {{the property checked and how}}
- **Observability required:** {{logs, metrics, traces, dumps}}
- **Exit criteria:** {{pass / fail / inconclusive conditions, run duration,
  statistical confidence if applicable}}
- **Target test file:** {{relative SUT path, e.g. `crates/agentdb-core/tests/auto/S1_linearizable_append_under_partition.rs`}}
- **Skeleton language:** {{rust | go | python | typescript | shell}}
- **Skeleton:**

  ```{{language}}
  //! AUTO-GENERATED from test plan: {{path/to/plan.md}}
  //! Scenario: {{S1 name}}
  //! Falsifies: C1, C3 (see §1b of the plan for full text)
  //!
  //! REVIEW BEFORE TREATING AS A PERMANENT REGRESSION:
  //! this file was generated by executing-distributed-system-tests
  //! in author mode. Verify the workload, faults, and oracle below
  //! match the plan's intent before relying on it.

  // SUT imports — pick the canonical entry points from the
  // architectural summary (§0) and the SUT model (§2).
  use {{sut_crate_path}}::{{entrypoint};

  #[{{test_attr, e.g. tokio::test}}]
  async fn {{S1_slug}}() {
      // 1. Workload — from §7 Workload field
      // TODO: instantiate per the plan

      // 2. Faults — from §7 Faults field
      // TODO: inject per the plan

      // 3. Oracle — from §7 Oracle field
      // TODO: assert per the plan
  }
  ```

  The skeleton is intentionally minimal: imports, function
  signature, three TODO regions matching the prose fields. The
  executing skill expands the TODOs in author mode.

#### §7.M — Model / history / checker discipline

Mandatory if any claim referenced in this scenario's `Falsifies if it
FAILs` row is in `{safety, durability, idempotency, isolation,
ordering, membership}`. Otherwise: write `§7.M: not applicable (no
gated claim category falsified)` and skip the fields below.

- **Model under test:** `register | map | queue | log | lock | lease | session | membership-table | counter | ledger | other(<name>)`
  — see `references/history-discipline.md` for what each model implies
  about checker choice.
- **Operation history:** which of the default 11 fields (op id,
  process id, invoke/complete ts, op type, key, input, output, error,
  timeout marker, node seen, fault epoch) the recorder captures; any
  scenario-specific extensions; the recording mechanism (in-process
  / external probe / server-side audit / combined). Default schema
  reference: `references/history-discipline.md`.
- **Checker:** name from the executing skill's
  `references/oracle-patterns.md` (Checker picker table at the top
  maps model + claim category → checker). If this scenario has no
  checker, write a 1–2 sentence justification for why the alternative
  oracle (assertion / SLO threshold / final-state invariant) is
  strong enough on its own.
- **Nemesis + landing evidence:** the nemesis name (see executing
  skill's `references/fault-injection-howto.md` taxonomy), plus the
  *observable signal* that proves the fault landed (RPC timeout rate
  jump, iptables packet counter, log marker, partition-status
  metric). "The injector reported success" is not landing evidence.
- **Ambiguous outcomes:** how the recorder treats timeouts, unknown
  commits, retries, and duplicate responses. Default: timeouts get
  `timeout_marker = true` with `complete_ts = null`; retries are
  separate ops sharing `input`; duplicates are an error in the
  recorder. Any scenario-specific deviation goes here.
- **Reduction plan:** if this scenario FAILs, the minimization recipe.
  Target smallest cluster size + shortest fault window + minimum op
  mix + deterministic seed that still reproduces. After reduction,
  classify into one of SUT / harness / checker / environment per the
  executing skill's `references/test-case-reduction.md`.

### Scenario S2: ...

## 7b. Coverage adequacy argument

For each claim from §1b, demonstrate that the chosen scenarios
together would falsify the claim if it were violated. The form is:

| Claim | Threat model (how it could fail) | Scenarios that exercise the threats | Why these are sufficient |
|---|---|---|---|
| C1 | (a) race between two appenders, (b) crash mid-commit, (c) replica divergence | S_idempotent_replay_under_partition (a), S_durability_survives_fsync_loss (b), S_quorum_after_asymmetric_partition (c) | All three threat dimensions are exercised under the worst-case fault for each; no threat is left without a scenario. |
| ... | ... | ... | ... |

A reviewer should be able to read this row by row and either
accept the argument or point at a specific gap ("scenario Sa
doesn't actually inject T2"). If the table is hard to fill in for
some claim, that claim's scenarios are inadequate — either add
scenarios or surface the limitation in §7c residual uncertainty.

## 7c. Residual uncertainty

What this plan does NOT falsify, and why that's acceptable. One
row per claim or threat that the scenarios do not cover.

| Uncertainty | Why uncovered | Why acceptable today | When to revisit |
|---|---|---|---|
| C12 (geo-replication safety) not exercised under cross-region partition | No cross-region harness exists | Production is single-region for now | When multi-region GA is on the roadmap |
| ... | ... | ... | ... |

If §7c is empty, that itself is the signal — explicitly state "the
plan exercises every claim under at least one realistic threat;
the only residual uncertainty is bug density we cannot quantify."
Don't hide an empty §7c; explicit-empty is the confidence-builder.

## 7d. Confidence statement

A one-paragraph plain-language statement of what a reviewer should
believe if all scenarios in §7 pass. Worth 4–8 sentences.

Example:
> If every scenario in §7 passes against `main` at the validation
> sha, the reviewer should believe: (a) the system honors C1, C2,
> C3, C5 under their stated threat models; (b) C7's geo-replication
> arm is unverified (see §7c) but is not in the production scope;
> (c) the scenarios are independent enough that no single common-
> mode failure would mask multiple of them; (d) the harness itself
> has been exercised against known-bad code in the past (cite the
> closed F_* findings in the findings doc) and produced FAIL when
> it should — so PASSes here are evidence, not vacuous. The
> reviewer SHOULD NOT believe this run validates anything in §7c
> or anything in §8 ("what this plan does NOT cover").

This paragraph is what a stakeholder reads. The rest of the plan
is supporting evidence; this is the verdict.

**Use conservative claims.** Phrase what the run would establish as
"we have not observed X under the tested threats" rather than "X
cannot happen." Absolute negatives are stronger than any finite test
suite can support. Conservative phrasing keeps the confidence
statement defensible.

**Surface-coverage disclosure rule.** If any boundary-style claim
(category `boundary` or `fairness`) has scenario arms in its
§7.M.S block that are expected to be `NOT-RUN` or `PARTIAL-surface`
in the executed plan, the §7d statement MUST explicitly name those
untested surfaces. Silent omission of an untested surface is the
specific failure mode this rule prevents — a reader of the §7d
statement should never come away believing a boundary claim was
fully exercised when in fact only one or two of its surfaces were.
"We tested the API arm; the export and admin arms are out of
scope this round because the harnesses for those surfaces are not
yet built" is the expected shape.

## 8. What this plan does NOT cover

Bullet list of explicit non-goals so reviewers know where the holes
are. Distinct from §7c (which is about claims the scenarios don't
fully exercise) — §8 is about whole *subsystems* or *modes* that
the plan declares out of scope on purpose.

## 9. Open questions / followups

Aspirational / long-tail / further investigation. Anything that
shouldn't block the actionable scenarios in §7.

- {{question}} — owner, by when
