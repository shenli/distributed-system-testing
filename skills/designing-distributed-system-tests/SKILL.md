---
name: designing-distributed-system-tests
description: Use when designing a test plan for a distributed or stateful system — anything with persistence, replication, consensus, retries, idempotency, async messaging, multi-tenancy, or partial failure. Plans are claim-driven: investigate the product's claimed guarantees first, then design hypotheses and scenarios that try to falsify those claims under fault. Handles both change-scoped plans (a specific commit / PR / feature) and project-wide plans (a holistic plan for the whole system with existing-test inventory and gap analysis). Also use when asked to write a stability plan, fault matrix, release-validation plan, durability test plan, partition test plan, upgrade test plan, crash-recovery test plan, linearizability test plan, deterministic-simulation plan, "test plan to enough coverage", "what should we be testing", or "make a holistic test plan". Trigger even if the user just says "what should we test for this change" or "design the test plan for this project". Produces a structured Markdown plan file with hypothesis-driven scenarios drawn from a curated technique catalog (Jepsen+Elle, deterministic simulation, chaos/fault injection, fuzzing, formal methods, property+metamorphic, performance, crash-recovery+upgrade).
---

# Designing Distributed-System Tests

The default for testing distributed and stateful systems — write a few
integration tests and call it done — finds a small fraction of the bugs
that actually break these systems in production. This skill enforces an
opinionated workflow: scope the change, generate failure-mode hypotheses
that cover the categories the literature says matter most, pick
techniques from a curated catalog, and emit a structured plan file that
the executing-distributed-system-tests skill (or a human) can run.

## Plan modes

This skill produces two shapes of plan. Decide which one applies before
you start; the steps below branch on it.

- **Change-scoped** — the default. Use when the caller names a commit,
  PR, branch-diff, or feature. The plan covers what *this change* could
  regress, scoped by its blast radius.
- **Project-wide** — use when the caller asks for a "release-validation
  plan", "stability plan for the whole system", "test plan to enough
  coverage", "what should we be testing", or otherwise frames the
  request without a specific change. The plan covers what *the system*
  should be tested for, with an explicit inventory of existing tests
  and a gap analysis driving the new-scenario list.

If the framing is ambiguous, ask once before starting — the modes
diverge enough that retrofitting one into the other wastes work.

## Process

Follow these steps in order. Do not skip; the order matters because
later steps depend on artifacts the earlier steps produce.

### 1. Scope the system

Read the project's entry points: `README`, `AGENTS.md` or `CLAUDE.md`,
top-level `docs/`, any existing test-plan or runbook files. Note:
- Tenancy / isolation model
- Persistence model (what is durable, fsync contract)
- Replication / consensus protocol, quorum, leadership
- Ordering guarantee exposed to clients
- Network boundaries (which RPCs / streams)
- Retry / idempotency contract
- Observability (logs, metrics, traces) available to an oracle

Write this as a one-paragraph SUT model. If anything is ambiguous from
the repo, ask the user before proceeding — do not invent guarantees.

### 1b. Extract claims and guarantees

A good test plan exists to falsify what the product *claims*. Before
generating hypotheses, write down what the SUT promises its users.
This is the spine the rest of the plan hangs off — every hypothesis,
every scenario, every oracle should be traceable back to a claim it
either confirms or refutes.

Sources to mine:
- README "guarantees" / "what we offer" sections
- API docs / reference manuals
- ARCHITECTURE / DESIGN docs (claims about consistency, durability,
  replication, fault tolerance)
- Public blog posts, talks, marketing material (if any)
- The code itself: function names, doc-comments on public APIs,
  error types (`IdempotencyConflict`, `StaleRead`, etc.) imply
  guarantees the system claims to enforce
- Existing test names (a test called `linearizable_under_partition`
  implies a linearizability claim under partition)

Categorise each claim:
- **Safety** — "the system never returns a stale read", "no
  acknowledged write is ever lost", "linearizable per key"
- **Liveness** — "every accepted operation eventually commits",
  "leader election completes within N seconds of crash"
- **Durability** — "fsync'd writes survive crash", "replicated
  writes survive single-AZ loss"
- **Performance / SLO** — "p99 append latency ≤ X ms at Y ops/s
  per session"
- **Operational** — "rolling upgrade is non-disruptive",
  "configuration changes are atomic"
- **Idempotency / dedup** — "same idempotency key never produces
  two committed effects"
- **Privacy / isolation** — "tenant A's reads never observe tenant
  B's writes"

If the project does NOT explicitly document a claim that appears
in the code, write it as an *inferred* claim and mark it as such —
inferred claims are still testable, and surfacing them often
catches places where the docs lie or are silent about real
guarantees the implementation depends on.

When done, you should have a numbered claims list (C1, C2, …). The
hypothesis-generation step (3) will reference these by number, the
coverage matrix (5) tracks claim × fault, and scenarios (7) state
which claim(s) each is trying to falsify. If a hypothesis cannot
be tied back to a claim, either name the missing claim explicitly
or drop the hypothesis — untethered hypotheses produce ceremonial
scenarios.

**Missing claims are a first-class finding.** During hypothesis
generation (step 3) you will encounter behaviors the implementation
relies on that no claim covers — Unicode normalisation policy,
specific timeout windows, edge-case error semantics. List these
in the plan's "Missing claims discovered" section (template §1c).
Surfacing them is one of the highest-value outputs of the whole
exercise: it tells the maintainer where docs and implementation
have drifted apart.

### 2. Scope the change OR the project

**Change-scoped:** Identify the commit, PR, or feature under test.
List every file touched and the surfaces (RPCs, on-disk formats,
replication messages, public APIs) affected. Build a one-paragraph
blast-radius statement.

**Project-wide:** No specific change. Instead, enumerate the system's
externally observable surfaces (public APIs, on-disk formats, wire
protocols, replication/consensus, background jobs, operational
controls) and the invariants each must preserve. Declare what is
in-scope and what is explicitly out-of-scope (adapters, ancillary
tools, demo apps) — a project-wide plan that tries to cover
everything covers nothing well.

### 2b. Inventory existing tests (project-wide only)

Walk the SUT's test surface: unit tests, integration tests, fault-
injection / stability harnesses, smoke scripts, CI workflows, and
any test-plan / runbook docs. For each notable test or harness,
capture: what subsystem, what invariant it pins, and what failure
modes it would catch. This becomes the left-hand column of the
coverage matrix in step 4b.

Do not re-test what is already covered well. The point of the gap
analysis is to surface what is NOT covered.

### 3. Generate failure-mode hypotheses

For each claim from step 1b, ask: under what conditions could the
SUT fail to honor this claim? Each hypothesis must be tied to one
or more claims by number ("could falsify C3 and C7"). Tests exist
to refute claims, not to "check that things work" — a passing test
should mean "this claim survived this fault", and a failing test
should name the claim it falsified.

**Walk the pitfall catalog.** Before generating hypotheses from
intuition, open `references/common-distributed-systems-pitfalls.md`.
It lists 16 failure modes that recur across the Jepsen analyses
corpus, each with a hypothesis template ready to paste-adapt. For
every pitfall, decide if it applies to this SUT: y / n / maybe.
Every `y` and most `maybe`s become hypothesis rows. This shortcut
prevents the common failure mode of plans that only test what the
agent already thought of.

Generate hypotheses for each touched surface (change-scoped) or
in-scope surface (project-wide) across these categories:
correctness, durability, liveness, partial failure, idempotency /
replay, upgrade / rollback, configuration, performance / fairness.

If a category is genuinely not applicable, say so explicitly. The act
of writing "N/A because…" surfaces wrong assumptions more often than
it sounds like it would.

In project-wide mode the list is typically larger (the system has
more surfaces than any single change). Group hypotheses by subsystem
so the gap-analysis table stays readable.

### 4. Select techniques

Open `references/catalog-index.md` and find the techniques that match
your hypotheses. For each technique you pick, open its reference file
and write down in the plan: which hypotheses it addresses, what it
would catch that other techniques would miss, the typical cost.

A change usually warrants 2–4 techniques in combination. One technique
is suspicious — re-check whether you've collapsed multiple distinct
hypotheses into one. A project-wide plan typically reaches further
across the catalog (5–7 techniques) because the surface is larger.

### 4b. Map coverage and identify gaps (project-wide only)

Build a table indexed by claim, not just by hypothesis. Each row:
the claim (C-number), the hypothesis that would falsify it, the
existing test(s) (from step 2b) that exercise it, the verdict
(covered / partial / not covered), and the gap kind (no test /
shallow test / oracle too weak / no fault-injection variant).
Sort by claim severity × gap so the highest-leverage gaps end up
at the top.

This table is the heart of the project-wide plan. It tells the
maintainer where the product's claims are unverified. Without it
the plan is just a wishlist.

**For very large systems (50+ claims, 100+ hypotheses), split the
matrix.** A per-claim summary table (one row per claim with rolled-
up verdict) gives the maintainer the at-a-glance view; a per-
hypothesis detail table keeps the granular gap-kind information.
Without the split, a single matrix where load-bearing claims appear
in many rows becomes unreadable.

### 4c. Declare environment requirements

For each technique you picked, list the runtime dependencies the
executing skill will need on the test box: container runtime
(docker / podman + compose), language toolchains (Rust, Go, Node,
Python at specific minima), database / object-store backends
(Postgres N+, MinIO / S3-compatible), fault-injection facilities
(iptables, tc/netem, libfaketime, dm-flakey, Toxiproxy), kernel
features (network namespaces for asymmetric partitions, cgroups
for IO throttling), observability tooling (Prometheus, OTLP
collector), and any project-specific binaries.

Put this in the plan's "Environment requirements" section as a
checklist with version floors where they matter. The executing
skill consults this list at its environment-capability probe step
and uses it to either guide the operator through install or mark
dependent scenarios INCONCLUSIVE.

### 5. Design scenarios

For each technique, write concrete scenarios. Each scenario must
specify: workload (what generator, rate, distribution, duration);
faults (schedule of what is injected when); oracle (the property
checked and how); observability required; exit criteria (pass / fail /
inconclusive thresholds).

Resist "logs look fine" as an oracle. The oracle must be a
machine-checkable property or a metric SLO with a defined threshold.

In project-wide mode, prioritise scenarios that fill the highest-
leverage gaps from step 4b, and tag each scenario with both the
hypothesis-rows it closes AND the claim(s) it tries to falsify.
Long-tail "nice to have" scenarios go into section 7 (open
questions / followups), not section 5 — the plan should be
actionable, not aspirational.

Every scenario name should encode the claim it targets:
`linearizable_per_session_under_partition`,
`durability_survives_fsync_loss`, `idempotent_replay_across_restart`.
A test named after its claim is harder to weaken; a test named
after its setup ("3-node cluster with chaos") tells you nothing
about what it actually verifies.

**Each scenario is an executable spec.** Beyond the prose
fields (Workload, Faults, Oracle, Observability, Exit criteria),
emit two more:

- **Target test file** — the relative SUT path where this test
  will live if/when it becomes a permanent regression. Follow
  the SUT's test conventions: for Rust crates `crates/<crate>/tests/auto/<S_id>_<slug>.rs`,
  for Go modules `<module>/<pkg>_test.go`, for Python pytest
  `tests/auto/test_<slug>.py`. The `auto/` subdirectory makes
  generated tests easy to find and review separately from
  hand-authored ones.
- **Skeleton** — a language-specific code block with imports,
  the test function signature, and TODO regions for the
  workload / faults / oracle bodies. The skeleton MUST include
  an `AUTO-GENERATED` header comment with the plan path and
  scenario id so a reviewer can trace any committed test back
  to its spec.

The skeleton is what the executing skill (in author mode) writes
to the target path and then fills the TODOs from. The plan +
the generated test are traceable back to each other; if the
plan's prose changes, the test should be regenerated.

### 5b. Argue coverage adequacy

A test plan that lists scenarios without arguing they are *enough*
is not a test plan — it's a wishlist. Before writing the plan file,
build the argument for adequacy. Cover three things:

**1. Architectural summary.** A one-page (≤ 30 lines) summary of the
system's actual architecture: the major components, how data flows
between them, where state is durable, where consensus runs, where
trust boundaries live. This is not the catalog (catalog is reference
material) — this is the system as it actually exists, written so a
reviewer who has never seen the codebase can follow the test plan.
The architectural summary makes it possible for the reviewer to
spot a missing test ("you have nothing exercising the
storage→index handoff") that a flat scenario list would hide.

**2. Coverage adequacy argument.** For each claim, demonstrate that
the chosen scenarios — *taken together* — would falsify the claim
if it were violated. The form is: "claim Cn could be violated under
threats T1, T2, …; scenarios Sa, Sb, Sc exercise those threats
under conditions X, Y, Z; therefore if Cn is wrong, at least one
of Sa/Sb/Sc would catch it." A reviewer should be able to read
this and either accept the argument or point at a specific gap
("scenario Sa doesn't actually inject T2 — it only injects T1").

**3. Residual uncertainty.** Honestly list what the plan does NOT
falsify and why that is acceptable. "Claim Cn is not exercised
under multi-AZ failure because the harness cannot inject AZ-level
faults today; we accept this risk because production deploys are
single-AZ for now." This section is what turns a plan from "tests"
into "an argument for shipping."

These three sections together are the "confidence" the reader
needs. Without them, the plan answers "what would we test" but
not "is testing this enough to ship."

### 6. Write the plan file

Copy `assets/plan-template.md` to the plan destination and fill it in.
Default destination is `docs/testing-plans/<short-slug>.md` in the SUT
repo; the user may override (e.g. when they don't want the agent
writing into their repo, fall back to whatever path they specify, or
to `./testing-plans/<short-slug>.md` in the current working directory
if no path was given).

If the parent directory does not exist, create it before writing. Many
repos won't have a `docs/testing-plans/` directory the first time this
skill runs; `mkdir -p` it without ceremony.

The plan slug is the only handoff to the executing skill. Pick a
descriptive slug — `durable-idempotent-append-replay`, not
`plan-1`.

### 7. Self-check

Read the plan back. Every hypothesis has at least one scenario.
Every scenario has an oracle that is not "logs look fine". Every
chosen technique cites its reference file. If anything fails the
check, fix it in the plan; do not move on with known gaps.

**The adequacy test.** Imagine a reviewer who has never seen the
codebase reading the plan cover to cover. Then they're asked: "if
all of these scenarios pass, would you be comfortable shipping
this code?" If the plan does not contain enough material for them
to answer yes/no with confidence — specifically: the architectural
summary, the coverage-adequacy argument per claim, the residual
uncertainty list — the plan is not done. A list of scenarios is
not a confidence argument.

## Early exit

If the change genuinely does not warrant a distributed test plan —
for example, a docs-only change, a typo fix, a refactor with no
behavior change covered by existing unit tests — say so explicitly
and recommend the appropriate lighter-weight testing. Do not produce
a ceremonial plan for changes that don't need one.

## What this skill does not do

- It does not execute the plan. That's the
  `executing-distributed-system-tests` skill.
- It does not author Jepsen tests, TLA+ specs, or fuzz harnesses.
  It tells the engineer which to reach for; building them stays the
  engineer's job.
- It does not replace project-specific stability plans. It produces
  change-scoped plans that complement them.

## Reference files

- `references/catalog-index.md` — start here; selector page
- `references/jepsen-and-elle.md`
- `references/deterministic-simulation.md`
- `references/chaos-and-fault-injection.md`
- `references/fuzzing.md`
- `references/formal-methods-tla.md`
- `references/property-and-metamorphic.md`
- `references/performance-and-benchmarking.md`
- `references/crash-recovery-and-upgrade.md`

Each reference file follows the same shape: when to reach for it,
what it detects well, what it misses, concrete tools, papers,
cost / wall-clock signal, plan checklist.

## Asset

- `assets/plan-template.md` — the structure to fill in.
