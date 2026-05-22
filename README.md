# Distributed Systems Testing Skills

**Two skills for AI coding agents that design and run claim-driven
tests for distributed and stateful systems.** Together they produce a
structured Markdown test plan and a findings report with 10-state
verdicts and an explicit SUT / harness / checker / environment blame
classification. A reviewer reads the two artifacts and decides
whether to ship; nothing else has to be re-run.

Works with Claude Code, Codex, Copilot CLI, Cursor, Gemini, or any
agent that reads Markdown and runs shell. The skills are plain
SKILL.md files. The agent executes them; the plan and findings
report are the output.

One skill designs the plan. The other runs it. A plan starts from
the product's claims, generates hypotheses tied to those claims, and
writes scenarios named after the claim each tries to falsify. For
consistency-critical scenarios, each scenario also binds an abstract
model (`register | queue | log | lock | lease | ledger | …`) to an
operation-history schema, a named checker, and a nemesis with
observable landing evidence. The plan ends with a coverage adequacy
argument and a conservative confidence statement.

## Why

The default for testing distributed and stateful systems — write a
few integration tests and call it done — finds a small fraction of
the bugs that actually break these systems in production: partial
network partitions, non-deterministic concurrency, crash-recovery,
upgrade/rollback, idempotency under replay, timing-sensitive ordering.

These skills enforce an opinionated workflow that pulls from the
field's hard-won knowledge:

- **Claim-driven, not test-driven.** Start from what the product
  promises. Every scenario falsifies one claim under one fault. A
  test named after its claim is harder to weaken than one named
  after its setup.
- **Coverage adequacy is a deliverable.** The plan ends with an
  argument that the chosen scenarios are *enough* to ship, plus an
  honest list of what stays unverified.
- **Reuse the SUT's own toolbox.** The execute skill discovers
  existing tests, runbooks, and fault-injection scaffolding before
  inventing anything new.
- **Model + history + checker, not just chaos.** For safety,
  durability, idempotency, isolation, ordering, or membership
  claims, every scenario declares an abstract model, an
  operation-history schema, a named checker (linearizability,
  serializability, session-consistency, no-lost-ack, exactly-once,
  …), and how it treats ambiguous outcomes (timeouts, unknown
  commits, retries). Chaos plus a model and a checker, not chaos
  alone.
- **No silent passes.** Every PASS cites oracle execution evidence
  *and* the signal proving the fault actually fired. Verdicts come
  from a 10-state set, so "the chaos script ran cleanly" can't be
  read as "the claim survived the fault." Every FAIL carries a
  SUT / harness / checker / environment blame tag so reproducers
  reach the right queue.

## What you get

End-to-end, the two skills produce:

```
testing-plans/<slug>.md             ← plan with §0–§9 (see below)
test-sessions/<UTC>/
  ├── session-log.md                 ← timeline + toolbox + env probe
  ├── logs/                          ← per-scenario stdout/stderr
  ├── metrics/                       ← metric snapshots
  ├── artifacts/                     ← ephemeral harnesses, dumps
  └── findings/
      ├── <scenario>.md              ← per-scenario verdict (written as run proceeds)
      └── report.md                  ← summary + adequacy + confidence delta
```

The plan structure (a reviewer can read this and decide whether to
ship without re-running the tests):

```
0. Architectural summary       — system as it actually exists
1. Scope
1b. Claims under test          — the spine
1c. Missing claims discovered  — docs ↔ code drift
2. SUT model
3. Existing test inventory     — what's already covered
4. Failure-mode hypotheses     — tied to claim IDs
5. Coverage matrix             — claim × hypothesis
6. Technique selection         — from the catalog
6b. Environment requirements
7. Scenarios                   — each named after the claim, with
                                  Target test file + Skeleton
   7.M Model / history /       — mandatory when the scenario falsifies
       checker discipline        a claim in {safety, durability,
                                  idempotency, isolation, ordering,
                                  membership}: model under test,
                                  operation-history schema, named
                                  checker, nemesis + landing evidence,
                                  ambiguous-outcome handling, reduction
                                  plan (SUT/harness/checker/env blame)
7b. Coverage adequacy argument — why these tests are enough
7c. Residual uncertainty       — what stays unverified, and why ok
7d. Confidence statement       — the reviewer's verdict
8. What this plan does NOT cover
9. Open questions / followups
```

### Example §7.M block (excerpt from a plan)

```
### Scenario S3: linearizable_append_under_partition
- Falsifies if it FAILs: C1 (every acknowledged append is durable
  and linearisable), C5 (leader election completes within 5s)
- Workload: 8 clients, 70% append / 30% read, 5min, key-skew zipf
- Faults: asymmetric partition isolating current leader at T+60s
  for 30s
- Oracle: linearizability via Porcupine over per-key histories

§7.M (model / history / checker discipline)
- Model under test:    log
- Operation history:   default 11-field schema (op id, process id,
                       invoke/complete ts, op type, key, input,
                       output, error, timeout marker, node seen,
                       fault epoch). Recorded in-process + server-
                       side audit.
- Checker:             linearizability (Porcupine) per-key, then
                       no-lost-ack against final state
- Nemesis + landing:   asymmetric-partition (iptables drop one
                       direction). Landing evidence = iptables drop
                       counter goes 0 → 14,712 over the 30s window
                       AND raft log emits "leader-lost; starting
                       election" within 2s of injection.
- Ambiguous outcomes:  timeouts → timeout_marker=true, complete_ts
                       =null, treated as could-have-succeeded;
                       retries are separate ops sharing input
- Reduction plan:      if FAIL, bisect fault window + fix seed, then
                       classify SUT / harness / checker / environment
                       per references/test-case-reduction.md
```

### Example findings-report row

| ID | Verdict             | Nemesis landing evidence                              | Reduction class |
|----|---------------------|-------------------------------------------------------|-----------------|
| S3 | `PASS-hardening`    | iptables ctr 0→14,712; raft re-election at T+1.8s     | n/a             |
| S4 | `FAIL-reproducible` | partition landed; Elle: G2-item anomaly on key K17    | SUT             |
| S7 | `INCONCLUSIVE-fault-not-proven` | iptables rule installed but counter stayed 0 — wrong chain | harness |
| S9 | `PARTIAL-model`     | landing ok; checker covered per-key, not cross-key    | n/a             |

(The full findings template carries Oracle, Oracle execution evidence,
artifact links, an adequacy-vs-plan section, and a confidence delta —
see `skills/executing-distributed-system-tests/assets/findings-report-template.md`.)

## Install (one line, any agent)

Paste this at any AI coding agent (Claude Code, Codex, Copilot CLI,
Cursor, Gemini, or anything else that reads Markdown and runs shell):

```
Read https://raw.githubusercontent.com/shenli/distributed-system-testing/main/INSTALL.md
and follow the instructions to install and configure
distributed-testing-skills for this agent.
```

The agent fetches [`INSTALL.md`](INSTALL.md), clones the repo to
`~/.local/share/distributed-testing-skills/`, and wires the skills in
(symlinks under `~/.claude/skills/` for Claude Code, a pointer block
in `~/AGENTS.md` for other agents).

After that, ask any agent on the machine to "design a test plan for
this system" or "execute the plan at X" and it'll follow the
SKILL.md workflow.

### Update

**Paste the same one-liner again.** `INSTALL.md` is idempotent:
if the install path exists, it does `git pull --ff-only`; if not,
it does `git clone`. Symlinks always point at the cloned content
so they pick up the new version automatically. The `~/AGENTS.md`
pointer block uses HTML markers and is replaced cleanly on each
run — no duplication.

If you have local edits to the cloned skills, `git pull --ff-only`
will fail; the agent will stop and ask before discarding them.

### Manual install (if you'd rather see what's happening)

```bash
git clone https://github.com/shenli/distributed-system-testing.git \
    ~/.local/share/distributed-testing-skills

# Claude Code: symlink under ~/.claude/skills/
mkdir -p ~/.claude/skills
ln -snf ~/.local/share/distributed-testing-skills/skills/designing-distributed-system-tests \
    ~/.claude/skills/designing-distributed-system-tests
ln -snf ~/.local/share/distributed-testing-skills/skills/executing-distributed-system-tests \
    ~/.claude/skills/executing-distributed-system-tests

# Codex / Copilot CLI / Cursor / Gemini / others: see INSTALL.md
```

## Usage

Once the skills are installed, you have two ways to drive them:

**Casual ask (Claude Code with auto-trigger):**

```
Design a project-wide test plan for this codebase.
```

```
Execute the plan at ./testing-plans/<slug>.md against this codebase.
```

The skill descriptions pick up natural phrasing like "design a
test plan", "execute the plan", "run stability tests", "design a
release validation plan", etc.

For a specific mode, output path, or a non-auto-trigger agent,
[`USAGE.md`](USAGE.md) has copy/paste prompts for every workflow
(design and execute, in their respective modes) plus tips on scope,
env probing, and long-run checkpointing.

## The two skills

### `designing-distributed-system-tests`

Walks the repo, extracts the claims the product makes, generates
hypotheses tied to those claims, picks techniques from the catalog,
and writes a structured Markdown plan with a coverage adequacy
argument and a confidence statement. For consistency-critical
scenarios, the plan fills a §7.M block per scenario: model under
test, operation-history schema, named checker, nemesis + landing
evidence, ambiguous-outcome handling, reduction plan. Details:
[`history-discipline.md`](skills/designing-distributed-system-tests/references/history-discipline.md).

Two modes: **change-scoped** (a specific commit or PR) and
**project-wide** (a holistic plan with existing-test inventory and
gap analysis).

### `executing-distributed-system-tests`

Reads the plan, discovers the SUT's toolbox, probes the environment,
and runs scenarios with checkpoint discipline. Per scenario: captures
landing evidence for the fault, runs the green-but-broken and
weak-oracle audits, assigns a verdict from the 10-state taxonomy in
[`verdict-taxonomy.md`](skills/executing-distributed-system-tests/references/verdict-taxonomy.md),
and classifies every FAIL into SUT / harness / checker / environment
before filing. Produces a findings report with adequacy-vs-plan
assessment and confidence delta.

Two modes: **default** (read-only on the SUT, ephemeral harnesses
under the session dir) and **author mode** (writes scenario skeletons
declared in the plan's §7 into the SUT for review).

## Technique catalog

Eight reference files distilled from the field's literature:

| File | When to reach for it |
|---|---|
| [`catalog-index.md`](skills/designing-distributed-system-tests/references/catalog-index.md) | Selector page — start here |
| [`jepsen-and-elle.md`](skills/designing-distributed-system-tests/references/jepsen-and-elle.md) | Linearizability / serializability under faults |
| [`deterministic-simulation.md`](skills/designing-distributed-system-tests/references/deterministic-simulation.md) | Reproducible bugs from a seed; async heavy code |
| [`chaos-and-fault-injection.md`](skills/designing-distributed-system-tests/references/chaos-and-fault-injection.md) | Real-cluster partial / asymmetric faults |
| [`fuzzing.md`](skills/designing-distributed-system-tests/references/fuzzing.md) | Input or concurrency fuzzing under sanitizers |
| [`formal-methods-tla.md`](skills/designing-distributed-system-tests/references/formal-methods-tla.md) | Protocol correctness at design time |
| [`property-and-metamorphic.md`](skills/designing-distributed-system-tests/references/property-and-metamorphic.md) | Algebraic-law / metamorphic-relation testing |
| [`performance-and-benchmarking.md`](skills/designing-distributed-system-tests/references/performance-and-benchmarking.md) | Tail latency / throughput / fairness |
| [`crash-recovery-and-upgrade.md`](skills/designing-distributed-system-tests/references/crash-recovery-and-upgrade.md) | Durability, replay, idempotency, mixed-version |

Each follows the same shape: when to reach for it, what it detects
well, what it misses, concrete tools, papers, cost signal, plan
checklist. The catalog index pairs symptoms to references.

## Repo layout

```
.
├── plugin.json                                 ← optional plugin manifest
├── README.md                                   ← this file
├── INSTALL.md                                  ← idempotent install / update (paste-this)
├── USAGE.md                                    ← copy/paste prompts for every workflow
├── LICENSE
├── skills/
│   ├── designing-distributed-system-tests/
│   │   ├── SKILL.md                            ← the design workflow
│   │   ├── assets/plan-template.md             ← §0–§9 incl. gated §7.M
│   │   └── references/                         ← 8-file technique catalog + index,
│   │                                             common-distributed-systems-pitfalls,
│   │                                             history-discipline
│   └── executing-distributed-system-tests/
│       ├── SKILL.md                            ← the execute workflow
│       ├── assets/
│       │   ├── session-log-template.md
│       │   └── findings-report-template.md     ← 10-state verdicts + landing evidence
│       └── references/                         ← oracle-patterns (checker picker + 13
│                                                 patterns), fault-injection-howto
│                                                 (22-row nemesis taxonomy),
│                                                 test-case-reduction (with blame
│                                                 classification), green-but-broken-
│                                                 red-flags (incl. weak-oracle audit),
│                                                 finding-classification (TaxDC),
│                                                 verdict-taxonomy (10-state)
├── evals/                                      ← eval suites for both skills
├── verification/                               ← real runs against AgentDB (concrete output)
└── specs/                                      ← original design spec
```

## Status

Early but exercised. Both skills have been driven against AgentDB
(a distributed agent runtime in Rust) end-to-end multiple times,
surfacing six findings (one P0-candidate now closed, two P1s shipped
as a PR, two open). The skill bodies evolve as harness experience
accumulates; expect minor updates to the SKILL.mds and templates
over the next few iterations.

Real plan outputs, session directories, and findings reports from
those runs live under [`verification/`](verification/), one
subdirectory per run, each with a `README.md` describing what
passed, what failed, and what the skill surfaced about itself in
the process. Notable runs:

- [`verification/agentdb-fab7d9d/`](verification/agentdb-fab7d9d/) —
  change-scoped plan + execution for AgentDB commit `fab7d9d` (durable
  idempotent append replay); 670-line plan with 16 hypotheses across
  all eight failure-mode categories.
- [`verification/agentdb-jepsen/`](verification/agentdb-jepsen/) —
  consistency + crash-recovery run with linearizability checking.
- [`verification/agentdb-projectwide-lidev/`](verification/agentdb-projectwide-lidev/)
  and `-v2` — project-wide plans with full coverage matrix +
  adequacy argument + confidence statement.

There is also an eval suite under [`evals/`](evals/) (separate
`evals.json` for the design and execute skills) — used to validate
behavioural changes to the SKILL.md bodies between iterations.

## Acknowledgements

The technique catalog is distilled from Andrey Satarin's comprehensive
[testing-distributed-systems](https://github.com/asatarin/testing-distributed-systems)
catalog. Seminal papers anchoring the catalog include:

- Yuan et al., "Simple Testing Can Prevent Most Critical Failures" (OSDI'14)
- Gunawi et al., "What Bugs Live in the Cloud?" (SoCC'14)
- Zheng et al., "Torturing Databases for Fun and Profit" (OSDI'14)
- Kingsbury & Alvaro, "Elle: Inferring Isolation Anomalies from
  Experimental Observations" (VLDB'20)
- Alfatafta et al., "Toward a Generic Fault Tolerance Technique for
  Partial Network Partitioning" (OSDI'20)
- Lou et al., "Understanding, Detecting and Localizing Partial Failures
  in Large System Software" (NSDI'20)
- Gao et al., "An Empirical Study on Crash Recovery Bugs in Large-Scale
  Distributed Systems" (FSE'18)
- Zhang et al., "Understanding and Detecting Software Upgrade Failures
  in Distributed Systems" (SOSP'21)
- Bornholt et al., "Using Lightweight Formal Methods to Validate a
  Key-Value Storage Node in Amazon S3" (SOSP'21)
- Newcombe et al., "How Amazon Web Services Uses Formal Methods" (CACM'15)

## License

MIT.
