# Distributed-Systems Testing Skills — Design

**Date:** 2026-05-16
**Status:** Draft — pending implementation
**Owner:** Li Shen

## Problem

Coding agents working on stateful, distributed, or agent-runtime systems
routinely propose test changes that miss the failure modes that actually break
these systems in production: partial network partitions, crash-recovery,
non-deterministic concurrency interleavings, upgrade/rollback, idempotency under
replay, timing-sensitive ordering bugs, configuration errors, limping hardware.
Today's defaults — write a unit test, maybe an integration test — find a small
fraction of these. Agents lack a curated, opinionated workflow for picking the
right testing technique for the change at hand and then driving it through to a
useful report.

The reference catalog by Andrey Satarin
(<https://github.com/asatarin/testing-distributed-systems>) plus a handful of
seminal papers (Yuan et al. "Simple Testing Can Prevent Most Critical Failures",
TaxDC, FlyMC, Torturing Databases, "Redundancy does not imply fault tolerance",
"Why is Random Testing Effective for Partition Tolerance Bugs?") collectively
encode the field's hard-won knowledge — but none of it is currently in a form
an agent can consult mid-task.

## Goals

1. Give coding agents a closed-set, opinionated way to **design a distributed-
   systems test plan** for a specific change, system, or release.
2. Give them a paired skill to **execute that plan** against the system under
   test using whatever fault-injection, workload, and observation tooling the
   project already provides.
3. Make technique selection trace back to a small curated catalog so plans are
   reproducible, reviewable, and don't depend on the agent's recall of
   distributed-systems lore.
4. Be reusable across projects, not coupled to any one codebase.

## Non-Goals

- Building a new testing framework, fault-injector, or oracle library. These
  skills orchestrate *existing* tools the SUT project provides.
- Auto-generating Jepsen tests or TLA+ specs. The skills point an agent at the
  right technique; concrete tool authoring stays out of scope.
- Replacing project-specific test plans (e.g. AgentDB's
  `docs/stability-test-plan.md`). The design skill complements those by
  producing change-scoped plans.
- Production incident response or post-mortem. Adjacent, but a different skill.

## Plugin Layout

```
~/work/distributed-testing-skills/
├── README.md
├── plugin.json                          (optional, for shareability)
├── specs/
│   └── 2026-05-16-distributed-testing-skills-design.md  (this doc)
└── skills/
    ├── designing-distributed-system-tests/
    │   ├── SKILL.md
    │   ├── references/
    │   │   ├── catalog-index.md
    │   │   ├── jepsen-and-elle.md
    │   │   ├── deterministic-simulation.md
    │   │   ├── chaos-and-fault-injection.md
    │   │   ├── fuzzing.md
    │   │   ├── formal-methods-tla.md
    │   │   ├── property-and-metamorphic.md
    │   │   ├── performance-and-benchmarking.md
    │   │   └── crash-recovery-and-upgrade.md
    │   └── assets/
    │       └── plan-template.md
    ├── executing-distributed-system-tests/
    │   ├── SKILL.md
    │   ├── references/
    │   │   ├── oracle-patterns.md
    │   │   ├── fault-injection-howto.md
    │   │   ├── test-case-reduction.md
    │   │   ├── finding-classification.md       (TaxDC-style taxonomy)
    │   │   └── green-but-broken-red-flags.md
    │   └── assets/
    │       ├── findings-report-template.md
    │       └── session-log-template.md
    └── evals/
        ├── designing/evals.json
        └── executing/evals.json
```

The technique catalog lives only under `designing-.../references/`. The execute
skill links across when it needs to recall what a chosen technique implies.

## Skill 1: `designing-distributed-system-tests`

### Trigger

Description (frontmatter) must trigger when an agent is about to:
- Test a change to a system with persistence, replication, consensus, retries,
  idempotency, async messaging, multi-tenancy, or partial failure.
- Produce a "stability plan", "fault matrix", "test design", "release
  validation plan", or "verification plan" for any service with the above
  properties.
- Investigate a regression or near-miss in such a system and decide what
  additional coverage would have caught it.

Description must explicitly include keywords agents commonly use: *distributed
test plan, stability test, fault injection plan, Jepsen, chaos, consistency
testing, durability test, crash recovery test, partition test, upgrade test,
linearizability test, deterministic simulation*.

### Process (rigid skill)

1. **Scope the system.** Read repo entry points, `README`, `AGENTS.md`/CLAUDE.md,
   and any existing test plan docs. Output a one-paragraph SUT description
   covering: tenancy, persistence model, replication/consensus, ordering
   guarantee, network boundaries, retry/idempotency contract, observability.
2. **Scope the change.** Identify what the change adds/modifies/removes and the
   surfaces it touches. Build a blast-radius list.
3. **Generate failure-mode hypotheses.** For each touched surface, ask:
   - Correctness: can this violate the ordering/consistency guarantee?
   - Durability: what happens on crash mid-operation? Under fsync loss?
   - Liveness: can this deadlock, livelock, starve, or limplock?
   - Partial failure: what if one of N replicas/nodes is partitioned or slow?
   - Idempotency/replay: what if the operation runs twice? Three times?
   - Upgrade/rollback: mixed-version state?
   - Configuration: bad value, missing value, default drift?
   - Performance/fairness: tail latency, head-of-line blocking?
4. **Select techniques from the catalog.** For each hypothesis, name the
   technique(s) most likely to surface it, citing the reference file. The
   selection MUST be justified — "what could it catch that other techniques
   miss?".
5. **Design scenarios.** For each technique, produce concrete scenarios:
   workload generator, fault schedule, oracle (the property checked),
   observability required (logs, metrics, traces, dumps), exit criteria
   (pass/fail/inconclusive, run duration, statistical confidence).
6. **Write the plan file.** Use `assets/plan-template.md`. Default location is
   `docs/testing-plans/<short-slug>.md` in the SUT repo; user may override.
7. **Self-check.** Read the plan back and verify every hypothesis has at least
   one scenario, every scenario has an oracle, no oracle is "logs look fine".

### Output

A single Markdown file with this structure (enforced by template):
- Change summary (commit hash or PR link, files touched, surfaces affected)
- SUT model (what was learned in step 1)
- Failure-mode hypotheses (numbered, linked to scenarios)
- Technique selection (with reasoning, linked to catalog refs)
- Scenarios (one per heading, with workload/faults/oracle/observability/exit)
- What this plan does NOT cover (explicit non-goals)
- Open questions / followups

### Reference files (the catalog)

Each file has the same shape — easy for the model to skim:
- **When to reach for it** (1–2 sentences)
- **What it detects well** (bullet list of failure modes)
- **What it misses** (so the agent knows when to pair with another technique)
- **Concrete tools** (with links: e.g. Jepsen, Elle, Maelstrom, Porcupine,
  Antithesis, FoundationDB DST, TLA+, Alloy, libFuzzer, AFL, PropEr/Hypothesis)
- **Papers to cite** (one or two anchors from the Satarin catalog)
- **Cost / wall-clock signal** (so plans don't propose a 30-day Jepsen run
  for a 1-day fix)

`catalog-index.md` is a one-page selector: a table of "if you suspect X, read Y"
so the agent doesn't have to open every catalog file.

## Skill 2: `executing-distributed-system-tests`

### Trigger

Description triggers when an agent has a plan (file or in conversation) and
needs to run it, OR when asked to "run stability tests", "execute chaos
scenarios", "drive fault injection", "reproduce a distributed bug", "validate a
release", "run Jepsen", "verify durability".

### Process (rigid skill)

1. **Load the plan.** If file: read it. If in-conversation: extract the
   scenario list; if missing oracles/exit criteria, halt and hand back to the
   design skill rather than improvising.
2. **Discover the SUT's test toolbox.** Search for `tools/`, `scripts/`,
   `tests/integration/`, `tests/stability/`, runbooks in `docs/`, Makefile
   targets, and existing CI definitions. Catalog what's available before
   writing any new code. (E.g. AgentDB: `tools/agentdb-cluster-smoke`,
   `tools/workload`, `tools/agentdb-bench`, `docs/runbooks/fault_injected.md`.)
3. **Establish a session directory.**
   `test-sessions/<plan-slug>/<UTC-timestamp>/` with `logs/`, `metrics/`,
   `artifacts/`, `findings/`. All output lands here.
4. **Run scenarios in plan order.** For each scenario:
   - Preconditions check (cluster state, instrumentation on, baseline metric).
   - Bring up SUT (using project's own scripts; never reinvent).
   - Start workload.
   - Inject faults per plan schedule.
   - Stop, collect, apply oracle.
5. **On failure: do not move on without recording.** Capture:
   - Reproducer (the smallest fault sequence that reproduces — apply
     test-case reduction patterns from references).
   - Classification (TaxDC-style: timing / ordering / partition /
     crash-recovery / config / upgrade / fault-handling).
   - Evidence (log excerpts, trace IDs, metric snapshots).
   - Hypothesis on root cause and which subsystem owns it.
6. **Watch for "green but broken".** Apply the red-flag checklist before
   declaring a scenario passed — e.g. workload throughput dropped to zero
   silently, oracle never ran, fault didn't actually inject, clock skew
   masked timing assertions.
7. **Produce findings report.** Use `assets/findings-report-template.md`:
   - Plan reference, commit-under-test, session directory.
   - Scenario-by-scenario result table.
   - Each finding written up with reproducer + classification + evidence +
     suggested next action.
   - Coverage summary: which plan hypotheses were exercised, which weren't,
     and why.

### Output

`test-sessions/<plan-slug>/<UTC-timestamp>/findings/report.md` plus the
session directory of raw artifacts. The report must be intelligible to someone
who didn't watch the run.

### Reference files

- `oracle-patterns.md` — linearizability (Porcupine/Elle), property assertion,
  metric SLO, replay-equivalence, invariant check, statistical comparison
  against baseline.
- `fault-injection-howto.md` — process kill, kill -9 vs SIGSTOP, disk full,
  fsync loss / power-loss simulation, partition (full, partial, asymmetric),
  packet loss, latency injection, clock skew, GC pause simulation, container
  restart.
- `test-case-reduction.md` — delta debugging / Minimal Causal Sequences from
  Scott et al.
- `finding-classification.md` — TaxDC-derived taxonomy as a one-page lookup.
- `green-but-broken-red-flags.md` — distilled from "Simple Testing Can Prevent
  Most Critical Failures" + "Why is Random Testing Effective for Partition
  Tolerance Bugs?" + Aysylu Greenberg's benchmarking talk.

## Cross-Skill Conventions

- **Plan slug** is the only identifier shared between skills. Design produces
  it, execute consumes it.
- **No hidden state.** Skills only communicate through artifacts in the
  filesystem (plan file, findings report). This keeps them composable with
  other agents and survivable across sessions.
- **Project autonomy preserved.** Neither skill modifies SUT source. Execute
  may add scripts to a sandbox under the session directory but never edits
  project files.
- **Reuse over reinvention.** Execute MUST prefer the SUT's existing test
  drivers, fault injection runbooks, and observability over rolling new ones.

## Verification on AgentDB

Per current AgentDB state (as of 2026-05-16):

1. Create a worktree from `main` for any AgentDB code changes the verification
   surfaces: `git worktree add ../agentdb-dst-verify main`.
2. **Design-skill run.** Prompt: "Design a stability test plan for commit
   `fab7d9d` (Fix durable idempotent append replay) on AgentDB." Expected
   plan outputs scenarios for at minimum:
   - Append idempotency under client retry storm
   - Replay equivalence after crash mid-append
   - Duplicate suppression under asymmetric partition between client and
     replica
   - Durability after simulated fsync loss
   - Concurrent forks observing the same append
   Plan must reference `docs/stability-test-plan.md`,
   `docs/runbooks/fault_injected.md`, and `tools/agentdb-cluster-smoke`
   without being told they exist.
3. **Execute-skill run.** Prompt: "Execute the plan you just produced."
   Expected: discovers and uses existing tools, produces a session directory
   with a findings report. Either ≥1 oracle violation with reproducer, or a
   passed scenarios summary with explicit "this is what coverage we got"
   statement.
4. **skill-creator eval flow.** 2–3 prompts per skill, with-skill vs baseline
   subagent runs, browser review per skill-creator workflow. The verification
   above is one of those prompts; add prompts for two unrelated systems
   (e.g. a small Raft library, a generic web service with a queue) to test
   that the skills don't overfit to AgentDB.

## Implementation Order

1. Write the design skill's SKILL.md + plan template + catalog-index.md (one
   round, kept tight).
2. Write the technique catalog reference files. Each ≤ 150 lines, common
   shape. Cite Satarin's catalog so the agent has anchors.
3. Write the execute skill's SKILL.md + finding/session templates + oracle
   and fault-injection references.
4. Verify on AgentDB scenario above. Iterate per skill-creator loop.
5. Write `evals/` for both skills (skill-creator schema).
6. Add `README.md` + optional `plugin.json` for shareability.

## Risks and Mitigations

- **Catalog drift.** Distributed-systems testing tools evolve. Each catalog
  ref file should carry a "last verified" date in frontmatter.
- **Over-prescription.** Rigid checklists can produce useless plans for small
  changes. Mitigation: design skill has an early-exit "this change does not
  warrant a distributed test plan" branch that says so explicitly.
- **Project-specific tool discovery.** Execute skill could miss the SUT's
  toolbox and reinvent. Mitigation: discovery step is non-optional and must
  be reported in the findings header before any scenario runs.
- **Agents claiming PASS when the oracle didn't run.** This is the bug that
  motivated the "green but broken" red-flag reference. Execute skill must
  echo the oracle's actual execution evidence in the findings, not just its
  verdict.

## Open Questions

- Does the design skill need a sibling skill for "review an existing test
  plan" (vs. produce a new one)? Defer until we see how the design skill is
  used in practice.
- Should the catalog include resilience/SRE references (game days, DiRT)?
  Yes for completeness but secondary — add after the core eight.
- Plugin packaging format: keep as plain folder for now, add `plugin.json`
  only if we want to distribute via a marketplace.
