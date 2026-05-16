# Using these skills with Codex

This repo ships two skills written for the Claude skill format
(YAML frontmatter + Markdown body, auto-loaded by Claude Code from
`~/.claude/skills/`). Codex does not have the same auto-loading
mechanism today, so these skills are used by **explicit reference**
in your prompts. This guide walks through that workflow end to end.

## Prerequisites

1. **Codex CLI** installed and authenticated.
2. **Clone of this repo** somewhere on the box Codex runs on.
3. **`cd` into the SUT source directory before launching Codex.**
   This is the most important step. Once Codex starts inside the
   SUT, the prompts below stop needing to spell out paths — Codex
   reads files, runs `git`, and walks the tree all from the
   current directory.
4. **The environment requirements** the SUT's plan calls for.
   For distributed systems this typically means Docker + a language
   toolchain. The execute skill's step 2b will tell you exactly
   what's needed; for the first run, just bring whatever you have.

```bash
# One-time: clone the skills repo somewhere stable.
git clone https://github.com/shenli/distributed-system-testing.git \
    ~/work/distributed-testing-skills

# Every time: cd into the system you want to test, then start Codex.
cd ~/work/my-distributed-system
export SKILLS=~/work/distributed-testing-skills
codex
```

## How Codex uses Claude-format skills

Codex does not look in `~/.claude/skills/` and does not parse the
YAML frontmatter. The skills' SKILL.md bodies are self-contained
imperative workflows — Codex follows them when you tell it to.
Two ways:

- **Per-prompt reference (recommended).** Give Codex the SKILL.md
  path in each prompt. Simple, explicit, works one-shot.
- **Pin via `AGENTS.md`.** Drop a pointer to the SKILL paths into
  your SUT's `AGENTS.md`; Codex auto-reads it. Good if you're
  iterating in one repo for a long time.

The rest of this guide assumes the per-prompt pattern.

## The two-skill workflow

```
designing-distributed-system-tests   →   <plan>.md
                                            ↓
                                     executing-distributed-system-tests
                                            ↓
                                     session-dir/ + findings/report.md
```

Run them in that order. The execute skill expects the plan file
the design skill produced.

---

## Step 1 — Design a test plan

Pick the mode before you prompt:

- **change-scoped** — you have a specific commit / PR / feature and
  want a plan that covers what it could regress.
- **project-wide** — you want a holistic plan with existing-test
  inventory, gap analysis, and a coverage-adequacy argument for
  the whole system.

Use the matching prompt below. The skill, template, and catalog
are all referenced by path; Codex reads them on demand. You're
already in the SUT directory, so the plan can use relative paths
and `git` naturally.

### Prompt A — change-scoped

```
Use the design skill at $SKILLS/skills/designing-distributed-system-tests/SKILL.md
to produce a change-scoped test plan for the change I name. Save the
plan to ./testing-plans/<short-slug>.md.

Change under test: <commit hash | PR #N | feature description>

Follow the skill exactly. Use the template at
$SKILLS/skills/designing-distributed-system-tests/assets/plan-template.md
verbatim for the section structure. Skim the catalog index at
$SKILLS/skills/designing-distributed-system-tests/references/catalog-index.md
before picking techniques, then read the specific technique
reference files that match the hypotheses.

The plan is not done unless §7b (coverage adequacy argument), §7c
(residual uncertainty), and §7d (confidence statement) are populated.
A list of scenarios without those three sections is a wishlist, not
a test plan. Use conservative claims in §7d.
```

### Prompt B — project-wide

```
Use the design skill at $SKILLS/skills/designing-distributed-system-tests/SKILL.md
to produce a project-wide test plan for this codebase. Save the plan
to ./testing-plans/<project>-project-wide.md.

Scope:
- In: <list of subsystems / crates / services to cover>
- Out of scope: <list of things to declare out of scope, with one-
  line reason each>

Follow the skill exactly, in project-wide mode. Use the template at
$SKILLS/skills/designing-distributed-system-tests/assets/plan-template.md
verbatim for the section structure. Skim the catalog index at
$SKILLS/skills/designing-distributed-system-tests/references/catalog-index.md
before picking techniques.

Project-wide mode requires the existing-test inventory (§3), the
coverage matrix indexed by claim × hypothesis (§5), and the
missing-claims-discovered table (§1c). The plan is not done unless
§7b (coverage adequacy argument), §7c (residual uncertainty), and
§7d (confidence statement) are populated. Use conservative claims
in §7d.
```

### What good output looks like (both modes)

The produced plan should have these top-level sections:

```
0. Architectural summary       (~30 lines)
1. Scope
1b. Claims under test          (table; the spine of the plan)
1c. Missing claims discovered  (table; gaps in docs vs code)
2. SUT model
3. Existing test inventory     (project-wide only)
4. Failure-mode hypotheses
5. Coverage matrix             (project-wide only)
6. Technique selection
6b. Environment requirements
7. Scenarios
7b. Coverage adequacy argument
7c. Residual uncertainty
7d. Confidence statement       (the verdict reviewers read)
8. What this plan does NOT cover
9. Open questions / followups
```

If §7b / §7c / §7d are missing or shallow, the plan is unfinished.
Tell Codex to redo those sections specifically.

---

## Step 2 — Execute the plan

Same Codex session, same SUT directory.

Pick the mode first:

- **Default (read-only on SUT):** runs scenarios via the SUT's
  existing tests + an ephemeral sibling-package harness under
  the session dir. Nothing is written into the SUT repo.
- **Author mode:** writes the scenario skeletons (declared in
  the plan's §7) into the SUT at their `Target test file`
  paths, fills the workload / faults / oracle TODOs from the
  prose, compiles, runs. Stages files for review — does NOT
  git commit. Use when you want the test set to grow as a
  durable artifact in the SUT repo, not just as a one-shot run.

### Prompt (default mode)

```
Use the execute skill at $SKILLS/skills/executing-distributed-system-tests/SKILL.md
to run the plan at <path/to/plan.md> against this codebase. Produce
a session directory and findings report under
./test-sessions/<slug>/.

Honest environment constraints:
- Available: <list — docker, language toolchain, fault-injection tools>
- Not available: <list — Go, sudo for iptables, etc.>

Follow the skill exactly. Use the findings-report template at
$SKILLS/skills/executing-distributed-system-tests/assets/findings-report-template.md
verbatim for structure. Read the five reference files under
$SKILLS/skills/executing-distributed-system-tests/references/
when you need to pick an oracle, look up a fault mechanism,
minimize a reproducer, classify a finding, or apply the green-
but-broken checklist.

Step 2b: BEFORE you probe anything, ASK ME what's in my env
(Docker / podman? sudo for iptables? Toxiproxy? Go toolchain?
libfaketime? etc.). I'll tell you what I have. THEN you verify
with quick probes and reconcile gaps. For missing-but-trivial
deps, surface install commands rather than running sudo. For
missing-non-trivial deps, mark dependent scenarios INCONCLUSIVE
with a one-line reason — never silently no-op.

Step 4 checkpoint discipline: anything expected to run > 5
minutes (cargo builds, compose up, multi-node smoke runs) MUST
run as a background process with periodic status polls. Never
block a foreground command past the harness watchdog. Write per-
scenario findings as you go, not only at the end.

Step 7: the findings report MUST include "Adequacy assessment vs
plan" and "Confidence delta" sections. A list of verdicts is not
a confidence verdict.
```

### Prompt (author mode)

```
Use the execute skill at $SKILLS/skills/executing-distributed-system-tests/SKILL.md
in AUTHOR MODE to run the plan at <path/to/plan.md> against this
codebase. The plan's §7 scenarios declare a `Target test file`
and `Skeleton` for each one; write the skeletons into the SUT
at those paths, fill the workload / faults / oracle TODOs from
the prose, compile, run.

Environment + checkpoint discipline + reporting requirements
are the same as default mode (see step 2b + step 4 + step 7
in the SKILL.md).

Author-mode-specific requirements:
- Pre-flight diff scan: for each Target test file that already
  exists, ASK ME whether to overwrite, skip, or write a sibling
  (`<path>.new.rs` etc.). Never silently overwrite.
- Read at least one sibling test file in the same target directory
  before writing each new test, so the generated code matches the
  codebase's idioms (fixtures, helpers, async runtime, etc.).
- Run the SUT's formatter (cargo fmt / gofmt / black / ...) on
  every written file.
- STAGE all changes for my review; do NOT git commit.
- For any scenario whose plan entry is missing `Target test file`
  or `Skeleton`, fall back to ephemeral-harness execution and
  flag the gap in the findings report.
```

Use author mode when you want the test set to GROW from the plan
— i.e. when this run should leave new committed tests behind for
the next reviewer / next CI run. Use default mode for one-shot
validation that doesn't change the SUT.

### What good output looks like

The session directory will have:

```
test-sessions/<UTC>/
├── session-log.md              (timeline + toolbox discovery)
├── logs/                       (per-scenario stdout/stderr)
├── metrics/                    (metric snapshots)
├── artifacts/                  (sibling-crate harnesses, dumps)
└── findings/
    ├── <scenario>.md           (per-scenario; written as run proceeds)
    └── report.md               (summary with adequacy + confidence)
```

The `findings/report.md` should:
- Lead with the headline verdict
- Have a per-scenario row with **oracle execution evidence**, not
  just verdict
- INCONCLUSIVE scenarios cite a one-line reason
- Include the "Adequacy assessment vs plan" table and the
  "Confidence delta" paragraph

---

## Step 3 — Validate the output

The skills are designed to produce confidence-building reports
specifically so a reviewer can check the output without re-running
the tests.

**Plan check:**
- §1b claims are concrete and specific (not "strong consistency")
- §7b coverage adequacy has ≥ 1 row per claim
- §7c residual uncertainty is non-empty OR explicitly states the
  plan exercises every claim under at least one realistic threat
- §7d confidence statement uses conservative claims ("we have not
  observed X" rather than "X cannot happen")

**Findings check:**
- Headline verdict is at the top
- Per-scenario rows include oracle execution evidence
- INCONCLUSIVE scenarios cite a one-line reason
- Adequacy-vs-plan section shows where the run fell short of what
  the plan argued
- Confidence delta section tells the stakeholder what to believe
  more/less after this run

If any of those are missing, ask Codex to add them — the templates
require them.

---

## Common pitfalls

**Codex runs a single long compose command, watchdog kills it.**
The skill has explicit checkpoint discipline in step 4. Tell Codex
to use background processes + periodic polling rather than blocking
foreground commands for anything > 5 minutes. Each scenario should
write to the session log before / after, not only at the end.

**Codex marks scenarios INCONCLUSIVE without telling you what's
missing.** The env-probe step (2b) is supposed to surface install
commands for missing deps. If Codex skipped that, point at the
step explicitly: "before running any scenario, complete the step
2b environment-capability probe and tell me what you found."

**The plan is just a list of scenarios with no argument.** Re-issue
the design prompt with emphasis on §5b ("Argue coverage adequacy")
and §7d ("Confidence statement"). These are the difference between
"here are some tests" and "here is why these tests are enough."

**Codex tries to modify SUT source.** The execute skill explicitly
forbids this in its "Project autonomy" section. If Codex starts
editing SUT files instead of writing harness code under the session
dir, stop it and re-orient: "do not modify any files in this repo;
only write under ./test-sessions/<slug>/."

---

## End-to-end example

For your first Codex run, point it at a small system you know well,
ask for a project-wide plan, and check that the output's §7d
confidence statement actually tells you what you'd believe after
the tests pass. If it does, the skills are working; if not, the
prompt or environment needs more guidance.

The maintainer of this repo runs the skills against AgentDB
regularly; the verification artifacts live locally (gitignored)
and demonstrate the shape: each `test-sessions/<UTC>/` has a
session-log.md, per-scenario findings written as the run
progressed, raw logs, and a final report with adequacy + confidence.

## Feedback

If a Codex run surfaces something the skills handle poorly, file
the friction back as an issue on this repo. The skills evolve
based on real harness experience; the more environments they're
exercised in, the better the templates get.
