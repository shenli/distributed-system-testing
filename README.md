# distributed-testing-skills

**Two coupled skills for AI coding agents that test distributed
systems. Works with your agent stack — Claude Code, Codex, Copilot CLI,
Cursor, Gemini, anything that can read Markdown and run shell.**

One skill designs a test plan; one executes it. The plan reads like a
Jepsen-style analysis: claims first, then hypotheses, then scenarios
each named after the claim it tries to falsify, then a coverage
adequacy argument that tells the reviewer why the test set is *enough*.

The skills are plain Markdown SKILL.md files. They were authored
against Claude Code's skill format but the bodies are self-contained
imperative workflows; any agent that can follow Markdown
instructions can use them.

## Why

The default for testing distributed and stateful systems — write a
few integration tests and call it done — finds a small fraction of
the bugs that actually break these systems in production: partial
network partitions, non-deterministic concurrency, crash-recovery,
upgrade/rollback, idempotency under replay, timing-sensitive ordering.

These skills enforce an opinionated workflow that pulls from the
field's hard-won knowledge:

- **Claim-driven, not test-driven.** Start from what the product
  *promises* its users. Every scenario exists to falsify a specific
  claim under a specific fault. A test named after its claim is
  harder to weaken than one named after its setup.
- **Coverage adequacy is a deliverable.** The plan ends with an
  explicit argument that the chosen scenarios are *enough* to ship,
  plus an honest residual-uncertainty list of what stays unverified.
- **Reuse the SUT's own toolbox.** The execute skill discovers
  existing tests, runbooks, and fault-injection scaffolding before
  inventing anything new.
- **No silent passes.** Every PASS cites oracle execution evidence;
  every INCONCLUSIVE cites the missing capability that made it so.

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
7b. Coverage adequacy argument — why these tests are enough
7c. Residual uncertainty       — what stays unverified, and why ok
7d. Confidence statement       — the reviewer's verdict
8. What this plan does NOT cover
9. Open questions / followups
```

## Install (one line, any agent)

Paste this at any AI coding agent — Claude Code, Codex, Copilot CLI,
Cursor, Gemini, or anything else that can read Markdown and run shell
commands:

```
Read https://raw.githubusercontent.com/shenli/distributed-system-testing/main/INSTALL.md
and follow the instructions to install and configure
distributed-testing-skills for this agent.
```

The agent fetches [`INSTALL.md`](INSTALL.md), clones the repo to
`~/.local/share/distributed-testing-skills/`, and wires the skills
into the agent — symlinks under `~/.claude/skills/` for Claude
Code, a pointer block in `~/AGENTS.md` for other agents.

After that, the skills trigger automatically: ask any agent on the
machine to "design a test plan for this system" or "execute the
plan at X", and it'll follow the SKILL.md workflow.

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

**Copy/paste-ready prompts** (when you want a specific mode, output
path, or want to drive a non-auto-trigger agent): see
[`USAGE.md`](USAGE.md) for the canonical prompts for every
workflow — design (project-wide / change-scoped), execute
(default / author mode), update, plus tips on scope, env probing,
and long-run checkpointing.

## The two skills

### `designing-distributed-system-tests`

Given a system and a change (or "project-wide" for a holistic plan),
walks the repo, extracts the claims the product makes, generates
hypotheses tied to those claims, picks techniques from the catalog,
and writes a structured Markdown plan with a coverage adequacy
argument and a confidence statement.

Two modes: **change-scoped** (a specific commit / PR / feature) and
**project-wide** (a holistic plan with existing-test inventory and
gap analysis).

### `executing-distributed-system-tests`

Given a plan file, discovers the SUT's toolbox, probes the
environment (asking the operator first), runs scenarios with
checkpoint discipline, captures findings as the run proceeds, and
produces a findings report with an explicit adequacy-vs-plan
assessment and confidence delta.

Two modes: **default** (read-only on the SUT, ephemeral harnesses
under the session dir) and **author mode** (writes scenario
skeletons declared in the plan's §7 into the SUT for review).

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
│   │   ├── assets/plan-template.md
│   │   └── references/                         ← technique catalog (8 + index)
│   └── executing-distributed-system-tests/
│       ├── SKILL.md                            ← the execute workflow
│       ├── assets/
│       │   ├── session-log-template.md
│       │   └── findings-report-template.md
│       └── references/                         ← oracle / fault / reduction / classification / red-flags
├── specs/                                      ← original design spec
└── plans/                                      ← original implementation plan
```

## Status

Early but exercised. Both skills have been driven against AgentDB
(a distributed agent runtime in Rust) end-to-end multiple times,
surfacing six findings (one P0-candidate now closed, two P1s shipped
as a PR, two open). The skill bodies evolve as harness experience
accumulates; expect minor updates to the SKILL.mds and templates
over the next few iterations.

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
