# Install or update distributed-testing-skills

You are an AI coding agent (Claude Code, Codex, Copilot CLI, Cursor,
Gemini, or any other agent that can read Markdown and run shell
commands). The user asked you to install or update
`distributed-testing-skills` from this URL. Follow these instructions
exactly.

**This file is idempotent.** Run it again any time to pick up the
latest version of the skills — `git pull` if the clone exists,
`git clone` if not. Symlinks and pointers are re-applied without
duplication.

## Step 1 — Clone or update the repo

```bash
SKILLS_DIR=~/.local/share/distributed-testing-skills

if [ -d "$SKILLS_DIR/.git" ]; then
    # Already installed — update to latest.
    git -C "$SKILLS_DIR" fetch -q origin
    if ! git -C "$SKILLS_DIR" diff --quiet origin/main; then
        echo "Updating skills from $(git -C "$SKILLS_DIR" rev-parse --short HEAD) to $(git -C "$SKILLS_DIR" rev-parse --short origin/main)"
    fi
    git -C "$SKILLS_DIR" pull --ff-only -q origin main
else
    # Fresh install.
    git clone -q https://github.com/shenli/distributed-system-testing.git "$SKILLS_DIR"
fi

echo "Skills at: $SKILLS_DIR"
echo "Version: $(git -C "$SKILLS_DIR" rev-parse --short HEAD)"
```

If `git pull --ff-only` fails because the user has local changes,
STOP and ask the user how to resolve before continuing — never
discard their edits.

## Step 2 — Hook the skills into the user's agent

Detect which agent you are running as, then take the matching path
below. If unsure, ask the user.

### Claude Code

Symlink **every** skill folder under `~/.claude/skills/`. The loop
picks up new skills automatically when the repo adds them:

```bash
mkdir -p ~/.claude/skills
for skill_path in "$SKILLS_DIR"/skills/*/; do
    skill_name=$(basename "$skill_path")
    ln -snf "$skill_path" "$HOME/.claude/skills/$skill_name"
done
ls ~/.claude/skills/ | grep -E "^(designing|executing)-distributed-system-tests$"
```

Symlinks always point at the latest cloned content — no further
action needed when the user re-runs this guide later to update.

### Codex CLI, Copilot CLI, Cursor, Gemini, or any other agent

These agents don't have a Claude-style skill auto-load directory.
Drop a pointer block into the user's home `AGENTS.md`. The block
references the skill paths by symbolic name, so re-running this
guide is idempotent — replace the existing block if present.

```bash
AGENTS_FILE=~/AGENTS.md
MARKER_START="<!-- distributed-testing-skills:start -->"
MARKER_END="<!-- distributed-testing-skills:end -->"

# Strip any existing block (idempotent reinstall / update).
if [ -f "$AGENTS_FILE" ] && grep -q "$MARKER_START" "$AGENTS_FILE"; then
    /usr/bin/sed -i.bak "/$MARKER_START/,/$MARKER_END/d" "$AGENTS_FILE"
    rm -f "$AGENTS_FILE.bak"
fi

# Append the current block, pointing at the same install path.
cat >> "$AGENTS_FILE" <<EOF
$MARKER_START
## distributed-testing-skills

This machine has distributed-systems testing skills installed at
$SKILLS_DIR/skills/. When the user asks for a distributed-systems
test plan or wants to execute one, follow the workflow in the
matching SKILL.md:

- Design plans:  $SKILLS_DIR/skills/designing-distributed-system-tests/SKILL.md
- Execute plans: $SKILLS_DIR/skills/executing-distributed-system-tests/SKILL.md

Both skills' bodies are self-contained imperative workflows.
Common-pitfalls catalog: $SKILLS_DIR/skills/designing-distributed-system-tests/references/common-distributed-systems-pitfalls.md
$MARKER_END
EOF
```

The marker comments make the block self-identifying — re-running
this guide replaces it cleanly without duplication.

### Per-project alternative

If the user prefers per-project setup over machine-wide, use the
project's `AGENTS.md` instead of `~/AGENTS.md`. Same marker pattern.

## Step 3 — Verify

```bash
ls "$SKILLS_DIR/skills/"
# expected: designing-distributed-system-tests  executing-distributed-system-tests

git -C "$SKILLS_DIR" log -1 --format='Installed version: %h %s (%ci)'
```

Tell the user:

> "Skills are at `$SKILLS_DIR`, version $(git -C "$SKILLS_DIR" rev-parse --short HEAD).
> To use them, ask me (in this agent) something like 'design a test
> plan for this codebase' or 'execute the plan at X against this
> codebase'. I'll follow the SKILL.md workflow."
>
> "To update later, just paste the install one-liner again."

## What the user gets

Two coupled skills:

- **designing-distributed-system-tests** — produces a claim-driven,
  Jepsen-style test plan with coverage adequacy argument and
  confidence statement. Two modes: change-scoped and project-wide.
- **executing-distributed-system-tests** — reads a plan, discovers
  the SUT's toolbox, runs scenarios with checkpoint discipline,
  produces a findings report. Two modes: default (read-only,
  ephemeral harness) and author mode (writes scenario skeletons
  into the SUT for review).

See the cloned `README.md` for full feature list and the
`skills/*/references/` directory for the 8-file technique catalog +
16-item common-pitfalls reference.
