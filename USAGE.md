# Usage — copy/paste prompts

Once the skills are installed (one-line install in [`README.md`](README.md)),
paste any of the prompts below at your AI agent. Each is self-contained:
fill in the angle-bracket placeholders, paste, run.

Auto-trigger note: Claude Code with skills under `~/.claude/skills/`
will pick the right skill from a casual natural-language ask too —
"design a test plan for this project" usually works without
copying any prompt below. The explicit prompts here are for when
you want a specific mode, output path, or scope.

**Before you start: start your agent inside the SUT source
directory.** Once the agent is in the SUT, the prompts below stop
needing to spell out paths — the agent reads files, runs `git`,
and walks the tree all from the current directory.

---

## Design a test plan

### Project-wide (holistic plan with claims, coverage matrix, confidence statement)

```
Design a project-wide test plan for this codebase using the
designing-distributed-system-tests skill.

In scope: <list subsystems / crates / services to cover>
Out of scope: <list with one-line reason each>

Save the plan to ./testing-plans/<short-slug>.md.

Required sections per the skill template:
§0 architectural summary, §1b claims, §1c missing-claims-discovered,
§3 existing-test inventory, §5 coverage matrix (claim × hypothesis),
§6b environment requirements, §7 scenarios named after the claim each
falsifies (including Target test file + Skeleton for executable-spec
mode), §7b coverage adequacy argument, §7c residual uncertainty,
§7d confidence statement.

Walk the common-distributed-systems-pitfalls.md reference before
generating hypotheses from intuition. Use conservative claims in §7d.
```

### Change-scoped (a specific commit / PR / feature)

```
Design a change-scoped test plan using the designing-distributed-system-tests
skill.

Change under test: <commit hash | PR #N | feature description>

Save the plan to ./testing-plans/<short-slug>.md.

Required sections per the skill template:
§0 architectural summary, §1b claims (limited to those the change
touches), §3 existing tests for the touched surfaces, §4 hypotheses
tied to claims, §6b environment requirements, §7 scenarios named
after the claim each falsifies, §7b coverage adequacy, §7c residual
uncertainty, §7d confidence statement using conservative claims.

Walk the common-distributed-systems-pitfalls.md reference before
generating hypotheses from intuition.
```

---

## Execute a test plan

### Default mode (read-only on SUT, ephemeral harness under session dir)

```
Execute the test plan at <path/to/plan.md> against this codebase
using the executing-distributed-system-tests skill. Produce a session
directory and findings report under ./test-sessions/<slug>/.

Before probing my environment, ASK ME what tooling is available
(Docker / podman? sudo for iptables? Toxiproxy? Go toolchain?
libfaketime? etc.). Then reconcile with quick probes. For missing
deps, surface install commands rather than running sudo. For
genuinely unavailable deps, mark dependent scenarios INCONCLUSIVE
with a one-line reason — never silently no-op.

For any command expected to run > 5 minutes (cargo builds, compose
up, multi-node smoke runs), use a background process with periodic
status pulls every 60-120 seconds. Write per-scenario findings as
you go, not only at the end.

The findings report MUST include "Adequacy assessment vs plan" and
"Confidence delta" sections per the template — a list of verdicts is
not a confidence verdict.
```

### Author mode (writes scenario skeletons into the SUT for review)

```
Execute the test plan at <path/to/plan.md> in AUTHOR MODE using the
executing-distributed-system-tests skill. Produce a session directory
and findings report under ./test-sessions/<slug>/.

The plan's §7 scenarios declare a Target test file and Skeleton for
each one. In author mode, write the skeletons to those paths, fill
the workload / faults / oracle TODOs from the prose, compile, run.

Author-mode-specific requirements:
- BEFORE writing any test file, do a pre-flight diff scan: if a
  Target test file already exists, ASK ME whether to overwrite, skip,
  or write a sibling (<path>.new.rs etc.). Never silently overwrite.
- Read at least one sibling test file in the same target directory
  first, so the generated code matches the codebase's idioms.
- Run the SUT's formatter (cargo fmt / gofmt / black / ...) on every
  written file.
- STAGE all changes for my review; do NOT git commit.
- For any scenario missing Target test file or Skeleton, fall back
  to ephemeral-harness execution and flag the gap in the report.

Environment + checkpoint discipline + reporting requirements are
the same as default mode (see the SKILL.md).
```

---

## Update the skills

Re-paste the install one-liner from [`README.md`](README.md). `INSTALL.md`
is idempotent: existing install → `git pull`; missing install → `git
clone`; symlinks and the `~/AGENTS.md` pointer block are replaced
cleanly.

```
Read https://raw.githubusercontent.com/shenli/distributed-system-testing/main/INSTALL.md
and follow the instructions to install or update
distributed-testing-skills for this agent.
```

---

## Tips

**Pick a scope deliberately.** Project-wide gives you the holistic
plan (good for release validation or "do we have enough tests");
change-scoped is faster and tighter (good for PR review). The skill
asks once if the framing is ambiguous; don't over-think it.

**Default first, author mode second.** Run in default mode to
validate the plan against the SUT without touching its source.
Then re-run in author mode to promote the worthwhile scenarios to
permanent regression tests. Most plans don't need every scenario
authored — pick the ones worth carrying.

**Ask, don't probe (step 2b).** When the execute skill starts, it
should ask you what's in your environment before running any
`which` / `--version` checks. If it skips that, prompt it
explicitly: "before probing, ask me what's in my env per step 2b."

**Long runs need checkpoints.** Compose stacks, multi-node smoke
runs, and big cargo builds easily run past harness watchdog limits
(typically ~10 min of silence). The execute skill has a checkpoint
discipline section; if the agent forgets, remind it: "use background
processes and poll every 60-120s; write to the session log when each
phase starts and ends."

**Conservative claims in §7d.** A confidence statement that says
"the system cannot violate C7" is wrong — no test suite proves
absolute negatives. Use "we have not observed C7 violations under
the tested threats." The skill enforces this in its self-check but
review the wording before shipping.

**Watch for "plan is just a list of scenarios."** If the design
skill emits §7 with no §7b adequacy argument or §7d confidence
statement, the plan isn't done. Re-prompt: "redo §7b and §7d —
these are what turn a list of tests into a confidence verdict."

**Hold the line on project autonomy.** The execute skill in
default mode is supposed to leave the SUT source untouched (only
write under the session dir; only write into the SUT in author
mode at declared Target test paths). If the agent starts editing
SUT files unprompted, stop it: "do not modify any files in this
repo; only write under ./test-sessions/<slug>/."

---

## See also

- [`README.md`](README.md) — what the skills are and how they're
  installed
- `skills/designing-distributed-system-tests/SKILL.md` — the design
  skill body (the canonical workflow)
- `skills/designing-distributed-system-tests/assets/plan-template.md` —
  the exact section structure plans must follow
- `skills/designing-distributed-system-tests/references/common-distributed-systems-pitfalls.md` —
  16 failure modes that recur across the Jepsen analyses corpus,
  with hypothesis templates ready to paste-adapt
- `skills/executing-distributed-system-tests/SKILL.md` — the execute
  skill body
- `skills/executing-distributed-system-tests/assets/findings-report-template.md` —
  the exact section structure findings reports must follow
