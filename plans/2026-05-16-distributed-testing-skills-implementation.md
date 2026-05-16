# Distributed-Systems Testing Skills Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build two coupled Claude skills — `designing-distributed-system-tests` and `executing-distributed-system-tests` — backed by a curated technique catalog distilled from Andrey Satarin's `testing-distributed-systems` repo, then verify them by running both end-to-end against AgentDB commit `fab7d9d` (Fix durable idempotent append replay).

**Architecture:** A small plugin repo at `~/work/distributed-testing-skills/` containing two SKILL.md files plus references and templates. The skills communicate only via filesystem artifacts (a plan file produced by skill 1, a session directory + findings report produced by skill 2). The technique catalog lives once under skill 1's references and is cited by skill 2 when needed.

**Tech Stack:** Markdown skills (Claude skill format with YAML frontmatter), `skill-creator` evaluation flow for iteration, AgentDB (Rust + multi-language SDKs) as the verification target with existing `tools/agentdb-cluster-smoke`, `tools/workload`, `tools/agentdb-bench`, and `docs/runbooks/fault_injected.md` as the SUT toolbox.

---

## Source Spec

This plan implements: `specs/2026-05-16-distributed-testing-skills-design.md`

## File Structure

```
~/work/distributed-testing-skills/
├── .gitignore                                              (Task 1)
├── README.md                                               (Task 1, expanded Task 14)
├── plugin.json                                             (Task 14)
├── specs/                                                  (already present)
│   └── 2026-05-16-distributed-testing-skills-design.md
├── plans/                                                  (this file)
│   └── 2026-05-16-distributed-testing-skills-implementation.md
├── skills/
│   ├── designing-distributed-system-tests/
│   │   ├── SKILL.md                                        (Task 6)
│   │   ├── assets/
│   │   │   └── plan-template.md                            (Task 2)
│   │   └── references/
│   │       ├── catalog-index.md                            (Task 3)
│   │       ├── jepsen-and-elle.md                          (Task 4)
│   │       ├── deterministic-simulation.md                 (Task 4)
│   │       ├── chaos-and-fault-injection.md                (Task 4)
│   │       ├── fuzzing.md                                  (Task 5)
│   │       ├── formal-methods-tla.md                       (Task 5)
│   │       ├── property-and-metamorphic.md                 (Task 5)
│   │       ├── performance-and-benchmarking.md             (Task 5)
│   │       └── crash-recovery-and-upgrade.md               (Task 5)
│   └── executing-distributed-system-tests/
│       ├── SKILL.md                                        (Task 11)
│       ├── assets/
│       │   ├── session-log-template.md                     (Task 7)
│       │   └── findings-report-template.md                 (Task 7)
│       └── references/
│           ├── oracle-patterns.md                          (Task 8)
│           ├── fault-injection-howto.md                    (Task 9)
│           ├── test-case-reduction.md                      (Task 10)
│           ├── finding-classification.md                   (Task 10)
│           └── green-but-broken-red-flags.md               (Task 10)
└── evals/
    ├── designing/evals.json                                (Task 12)
    └── executing/evals.json                                (Task 12)
```

AgentDB verification touches (Task 13):
- New worktree: `/Users/lishen/work/agentdb-dst-verify/` (created from `main`)
- New file in worktree: `docs/testing-plans/durable-idempotent-append-replay.md` (produced by the design skill)
- New file (outside repo): `~/work/distributed-testing-skills/verification/agentdb-fab7d9d/findings/report.md` (produced by the execute skill)

---

## Task 1: Plugin scaffolding

**Files:**
- Create: `/Users/lishen/work/distributed-testing-skills/.gitignore`
- Create: `/Users/lishen/work/distributed-testing-skills/README.md`

- [ ] **Step 1: Write the failing precondition check**

This "test" is a directory layout assertion — the cheapest way to fail-fast.

Run:
```bash
test -d /Users/lishen/work/distributed-testing-skills/skills/designing-distributed-system-tests && echo OK || echo MISSING
test -d /Users/lishen/work/distributed-testing-skills/skills/executing-distributed-system-tests && echo OK || echo MISSING
```
Expected: `MISSING` twice (directories not created yet).

- [ ] **Step 2: Create skill directory skeleton**

```bash
mkdir -p /Users/lishen/work/distributed-testing-skills/skills/designing-distributed-system-tests/{references,assets}
mkdir -p /Users/lishen/work/distributed-testing-skills/skills/executing-distributed-system-tests/{references,assets}
mkdir -p /Users/lishen/work/distributed-testing-skills/evals/{designing,executing}
mkdir -p /Users/lishen/work/distributed-testing-skills/verification
```

- [ ] **Step 3: Write `.gitignore`**

```
.DS_Store
*.swp
verification/*/test-sessions/
verification/*/work/
**/.claude-cache/
```

- [ ] **Step 4: Write minimal `README.md`**

```markdown
# distributed-testing-skills

Two coupled Claude skills for testing distributed and stateful systems:

- **designing-distributed-system-tests** — given a system and a change,
  produce a structured test plan that picks techniques from a curated
  catalog (Jepsen, deterministic simulation, chaos, fuzzing, formal methods,
  property/metamorphic, performance, crash-recovery+upgrade).
- **executing-distributed-system-tests** — given a plan, drive it against
  the system using the project's existing test toolbox and produce a
  findings report.

See `specs/` for the design and `plans/` for the implementation plan.

## Install

Skills are loaded from `skills/`. For Claude Code, symlink or copy each
skill folder into `~/.claude/skills/`. A plugin manifest will be added
once the skills are verified end-to-end.

## References

Technique catalog distilled from
<https://github.com/asatarin/testing-distributed-systems>.
```

- [ ] **Step 5: Verify layout**

Run:
```bash
test -d /Users/lishen/work/distributed-testing-skills/skills/designing-distributed-system-tests/references && echo OK || echo FAIL
test -d /Users/lishen/work/distributed-testing-skills/skills/executing-distributed-system-tests/references && echo OK || echo FAIL
test -f /Users/lishen/work/distributed-testing-skills/README.md && echo OK || echo FAIL
```
Expected: `OK` three times.

- [ ] **Step 6: Commit**

```bash
cd /Users/lishen/work/distributed-testing-skills
git add .gitignore README.md skills/ evals/ verification/ 2>/dev/null
# verification/, skills/, evals/ are empty dirs — git won't track them yet; that's OK
git add .gitignore README.md
git commit -m "Scaffold plugin layout and minimal README"
```

---

## Task 2: Plan template for the design skill

**Files:**
- Create: `/Users/lishen/work/distributed-testing-skills/skills/designing-distributed-system-tests/assets/plan-template.md`

- [ ] **Step 1: Write the template**

```markdown
# Test Plan: {{change_slug}}

**Date:** {{YYYY-MM-DD}}
**Change under test:** {{commit_or_pr_link}}
**SUT:** {{system_name}}
**Plan author:** {{agent or human}}
**Status:** draft | reviewed | executed

## 1. Change summary

One paragraph: what does the change add / modify / remove, and which
subsystems does it touch?

Files touched:
- `path/to/file` — what changed
- `path/to/file` — what changed

## 2. SUT model (relevant slice)

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

## 3. Failure-mode hypotheses

Number each so scenarios can link back.

H1. **{{title}}** — {{one-sentence statement of what could go wrong}}
   - Suspected because: {{reason — code path, prior bug, paper, intuition}}
H2. ...

Cover at minimum: correctness, durability, liveness, partial failure,
idempotency/replay, upgrade/rollback, configuration, performance/fairness.
If a category is genuinely N/A for this change, say so explicitly.

## 4. Technique selection

For each chosen technique, state:
- **Hypotheses it addresses:** H1, H3, ...
- **What it would catch that other techniques miss**
- **Reference:** `references/{{technique}}.md`
- **Cost / wall-clock estimate**

## 5. Scenarios

### Scenario S1: {{name}}
- **Addresses:** H1, H2
- **Technique:** {{from §4}}
- **Workload:** {{generator, rate, key/op distribution, duration}}
- **Faults:** {{schedule — what is injected, when, for how long}}
- **Oracle:** {{the property checked and how}}
- **Observability required:** {{logs, metrics, traces, dumps}}
- **Exit criteria:** {{pass / fail / inconclusive conditions, run duration,
  statistical confidence if applicable}}

### Scenario S2: ...

## 6. What this plan does NOT cover

Bullet list of explicit non-goals so reviewers know where the holes are.

## 7. Open questions / followups

- {{question}} — owner, by when
```

- [ ] **Step 2: Verify**

Run:
```bash
test -f /Users/lishen/work/distributed-testing-skills/skills/designing-distributed-system-tests/assets/plan-template.md && wc -l /Users/lishen/work/distributed-testing-skills/skills/designing-distributed-system-tests/assets/plan-template.md
```
Expected: line count between 50 and 120.

- [ ] **Step 3: Commit**

```bash
cd /Users/lishen/work/distributed-testing-skills
git add skills/designing-distributed-system-tests/assets/plan-template.md
git commit -m "Add plan template for designing skill"
```

---

## Task 3: Catalog index (selector page)

**Files:**
- Create: `/Users/lishen/work/distributed-testing-skills/skills/designing-distributed-system-tests/references/catalog-index.md`

- [ ] **Step 1: Write the selector page**

```markdown
# Catalog Index — Pick a Technique by Symptom

This page is for the agent to skim BEFORE opening any specific technique
reference. Match the suspicion on the left to the technique on the right;
read that file for "when / what it catches / what it misses / tools / papers
/ cost". Most plans use 2–4 techniques in combination, not one.

| Suspect this could break... | Reach for... |
|---|---|
| Linearizability / serializability under concurrent ops + faults | `jepsen-and-elle.md` |
| Crash mid-operation, replay, or fsync loss correctness | `crash-recovery-and-upgrade.md` |
| Non-deterministic interleavings that "passed in CI" | `deterministic-simulation.md` or `fuzzing.md` |
| Partial network partitions, asymmetric loss, clock skew | `chaos-and-fault-injection.md` |
| Input parser / state-machine bugs from unexpected bytes or call sequences | `fuzzing.md` |
| Algorithm-level correctness of a protocol you wrote | `formal-methods-tla.md` |
| Whole-system invariants that should hold across many inputs | `property-and-metamorphic.md` |
| Tail latency, head-of-line blocking, throughput collapse under load | `performance-and-benchmarking.md` |
| Mixed-version cluster, rolling upgrade, downgrade, schema migration | `crash-recovery-and-upgrade.md` |
| Limping (degraded but not dead) node | `chaos-and-fault-injection.md` + `performance-and-benchmarking.md` |
| Config typos / defaults silently changing behavior | `property-and-metamorphic.md` (config space) |

## How to combine

- **Jepsen + chaos:** Jepsen drives a workload + oracle; the chaos layer
  injects realistic faults. Use both together for distributed correctness.
- **DST + property:** deterministic simulation gives reproducibility;
  property assertions give the oracle. Both layered is how FoundationDB
  and TigerBeetle catch most regressions before merge.
- **Formal + tests:** TLA+ proves the design; tests catch the
  implementation's drift from the design. One does not replace the other.
- **Fuzzing + sanitizers:** fuzz inputs are only as useful as the bug
  detectors running underneath (ASan, MSan, UBSan, race detectors).

## Quick "do not skip" rules from the literature

- Most production failures come from a small number of underlying causes
  that simple testing can catch — see "Simple Testing Can Prevent Most
  Critical Failures" (Yuan et al., OSDI'14). Always test error-handling
  paths, not just happy paths.
- Random testing is effective at finding partition-tolerance bugs even
  without deep symbolic reasoning — see "Why Is Random Testing Effective
  for Partition Tolerance Bugs?" (Majumdar et al., POPL'18). When in
  doubt, run more random scenarios.
- Redundancy does not imply fault tolerance — replicas can all mishandle
  the same corrupted byte. Always test the fault, not just the failover.
```

- [ ] **Step 2: Commit**

```bash
cd /Users/lishen/work/distributed-testing-skills
git add skills/designing-distributed-system-tests/references/catalog-index.md
git commit -m "Add technique selector catalog index"
```

---

## Task 4: First three catalog references (Jepsen, DST, Chaos)

Each reference file follows the same shape. Keep each ≤ 150 lines.

**Files:**
- Create: `skills/designing-distributed-system-tests/references/jepsen-and-elle.md`
- Create: `skills/designing-distributed-system-tests/references/deterministic-simulation.md`
- Create: `skills/designing-distributed-system-tests/references/chaos-and-fault-injection.md`

- [ ] **Step 1: Common shape (apply to all three)**

Each file:
```markdown
# {{Technique name}}

**Last verified:** 2026-05-16

## When to reach for it
Two short sentences.

## What it detects well
- Bullet list of failure modes this technique surfaces.

## What it misses
- Bullet list of blind spots; pair with these other techniques.

## Concrete tools
- `Tool` — one-line description — <link>

## Papers to cite
- "{{paper title}}" — one-line takeaway — <link>

## Cost / wall-clock signal
One sentence on typical runtime, ranging from cheap (minutes) to
expensive (multi-day), and what dominates that cost.

## How a plan typically uses it
3–6 bullet checklist a plan author should hit.
```

- [ ] **Step 2: Write `jepsen-and-elle.md`**

Key content (paraphrase, do not copy verbatim from sources):

- *When:* distributed correctness — consistency models, isolation,
  linearizability under client concurrency + injected faults.
- *Detects well:* lost updates, dirty reads, stale reads, lost writes
  under partition, lost acks, divided-brain after partition, isolation
  anomalies (G0, G1a/b/c, G-single, G2-item, G2), reconfiguration races.
- *Misses:* performance regressions, single-node memory leaks, UI bugs,
  bugs that only manifest under workloads the test generators don't
  emit, slow leaks of correctness over very long runs (Elle catches more
  here but still bounded by ops history length).
- *Tools:* Jepsen (Clojure, ships with nemesis/fault patterns and a set
  of generators), Elle (transactional anomaly checker, language-agnostic
  via op history), Porcupine (Go linearizability checker), Maelstrom
  (Jepsen toy-protocol workbench).
- *Papers:* "Elle: Inferring Isolation Anomalies from Experimental
  Observations" (Kingsbury & Alvaro, VLDB'20).
- *Cost:* per-scenario 30 minutes to multiple hours; Elle history
  analysis itself is sub-second to minutes depending on op count.
- *Plan checklist:* (1) name the consistency model claimed; (2) pick
  generators that exercise the operations whose anomalies you care
  about; (3) pick a nemesis schedule with realistic partition shapes,
  not just symmetric ones; (4) state which Elle anomaly classes count
  as failures; (5) record ops history for offline re-analysis;
  (6) define minimum ops-per-run for statistical relevance.

- [ ] **Step 3: Write `deterministic-simulation.md`**

Key content:

- *When:* concurrency-heavy or async-IO-heavy code where you can plumb
  all IO and clock through a controllable runtime; you want bug
  reproducibility from seed alone.
- *Detects well:* race conditions, ordering bugs, retry storms, deadlocks,
  hot-loop livelocks, bugs that only surface under specific message
  interleavings. Found bugs are minimally reproducible from a seed.
- *Misses:* anything outside the simulated boundary — kernel bugs, NIC
  driver bugs, disk firmware bugs, real-world timing weirdness, perf
  pathologies that depend on real CPU cache behavior.
- *Tools:* FoundationDB DST harness (the original), TigerBeetle VOPR,
  RisingWave's madsim, Antithesis (commercial, autonomous on top of
  DST), shuttle (Rust), turmoil (Rust).
- *Papers:* "Testing Distributed Systems w/ Deterministic Simulation"
  (FoundationDB, talks/blog posts); Antithesis blog "Is something
  bugging you?".
- *Cost:* infrastructure cost upfront (plumbing IO/clock) is large;
  per-scenario runs are cheap (seconds to minutes); total wall-clock
  scales with seeds you choose to explore.
- *Plan checklist:* (1) confirm the SUT uses a simulated runtime or
  state the work needed to introduce one — DST without it is fiction;
  (2) define the property assertions / oracles inside the simulation,
  not just outside; (3) set a seed budget per scenario; (4) record
  seeds of any failures for replay; (5) include time-warp scenarios
  (slow node, fast node) not just partition.

- [ ] **Step 4: Write `chaos-and-fault-injection.md`**

Key content:

- *When:* you have a real cluster (or close enough) and want to know
  whether it survives the faults that actually happen in production.
- *Detects well:* partial / asymmetric partitions, slow nodes (limping),
  packet loss, latency injection, disk-full, fsync failure, process
  kill, container restart, clock skew, GC pause analogues. Especially
  useful for catching limplock (Hwang et al.) and partial-failure
  detection (Lou et al.).
- *Misses:* deterministic reproducibility (re-running may not hit the
  same interleaving), root-cause without good observability, anything
  that requires whole-history analysis.
- *Tools:* Toxiproxy, Chaos Mesh, LitmusChaos, AWS Fault Injection
  Service, jepsen's nemesis (the fault layer alone), tc/netem,
  iptables, manual `kill -9` / `SIGSTOP` (the latter simulates GC
  pause well).
- *Papers:* "Toward a Generic Fault Tolerance Technique for Partial
  Network Partitioning" (Alfatafta et al., OSDI'20); "Understanding,
  Detecting and Localizing Partial Failures" (Lou et al., NSDI'20);
  "Limping-Hardware Tolerant Clouds" (Do et al.).
- *Cost:* runs typically minutes to hours per scenario; total budget
  is usually bounded by the cluster availability, not compute.
- *Plan checklist:* (1) catalog the realistic faults — power loss,
  network partition (full + partial), slow disk, slow node, clock skew;
  (2) for each, define the property the SUT should preserve;
  (3) require an oracle that actually runs after the fault clears;
  (4) demand evidence the fault was injected (log line, packet capture,
  proxy stat) — silent no-op is the most common false pass;
  (5) include recovery time as an exit criterion, not just correctness.

- [ ] **Step 5: Verify**

```bash
for f in jepsen-and-elle.md deterministic-simulation.md chaos-and-fault-injection.md; do
  path="/Users/lishen/work/distributed-testing-skills/skills/designing-distributed-system-tests/references/$f"
  test -f "$path" || { echo "MISSING $f"; exit 1; }
  lines=$(wc -l < "$path")
  echo "$f: $lines lines"
  test "$lines" -le 200 || { echo "TOO LONG"; exit 1; }
done
```
Expected: each file present, each ≤ 200 lines.

- [ ] **Step 6: Commit**

```bash
cd /Users/lishen/work/distributed-testing-skills
git add skills/designing-distributed-system-tests/references/jepsen-and-elle.md \
        skills/designing-distributed-system-tests/references/deterministic-simulation.md \
        skills/designing-distributed-system-tests/references/chaos-and-fault-injection.md
git commit -m "Add Jepsen/Elle, DST, and chaos catalog references"
```

---

## Task 5: Remaining catalog references

**Files:**
- Create: `skills/designing-distributed-system-tests/references/fuzzing.md`
- Create: `skills/designing-distributed-system-tests/references/formal-methods-tla.md`
- Create: `skills/designing-distributed-system-tests/references/property-and-metamorphic.md`
- Create: `skills/designing-distributed-system-tests/references/performance-and-benchmarking.md`
- Create: `skills/designing-distributed-system-tests/references/crash-recovery-and-upgrade.md`

Each file follows the Task 4 Step 1 common shape. Key content per file:

- [ ] **Step 1: `fuzzing.md`**

- *When:* parsers, state machines, RPC servers, anywhere untrusted or
  arbitrary bytes/sequences enter. Also: randomized concurrency fuzzing
  for scheduling/message-order bugs.
- *Detects well:* memory unsafety (with ASan/UBSan), parser crashes,
  state-machine traps, panic-on-bad-input, integer overflow, infinite
  loops on malformed input, message-order bugs (FlyMC-style).
- *Misses:* high-level correctness (you need an oracle beyond
  not-crashed), bugs that require multi-input setup, perf regressions.
- *Tools:* libFuzzer, AFL/AFL++, cargo-fuzz, go-fuzz (now in stdlib),
  honggfuzz, FlyMC (research, for distributed interleavings),
  randomized scheduler frameworks.
- *Papers:* "Combining AFL and QuickCheck for Directed Fuzzing"
  (Dan Luu); "FlyMC: Highly Scalable Testing of Complex Interleavings"
  (Lukman et al., EuroSys'19).
- *Cost:* CPU-hours to CPU-weeks; well suited to running continuously.
- *Plan checklist:* (1) identify each input boundary; (2) write a
  fuzz target that hits the smallest meaningful entry point;
  (3) run under sanitizers; (4) seed the corpus from real samples or
  existing tests; (5) set a corpus size or coverage exit criterion,
  not just wall-clock.

- [ ] **Step 2: `formal-methods-tla.md`**

- *When:* you are designing a protocol or invariant where getting it
  wrong is expensive (consensus, replication, leasing, transaction
  commit, crash-recovery). The design is small enough to model.
- *Detects well:* protocol-level bugs (livelock, ordering violations,
  invariant violations) at the design stage, mixed-version protocols,
  reconfiguration edge cases.
- *Misses:* implementation bugs — the model is not the code; perf;
  anything outside the modelled state space.
- *Tools:* TLA+ / PlusCal (TLC model checker, TLAPS prover), Alloy,
  P language (state-machine focused), Coq/Verdi (heavy).
- *Papers:* "How Amazon Web Services Uses Formal Methods" (CACM'15);
  "Using Lightweight Formal Methods to Validate a Key-Value Storage
  Node in Amazon S3" (Bornholt et al., SOSP'21).
- *Cost:* days to weeks of human time to model; model-check runs
  minutes to hours depending on state-space size.
- *Plan checklist:* (1) state the invariants you want preserved;
  (2) model only the relevant slice — abstract everything else;
  (3) check both safety and liveness, with fairness assumptions
  documented; (4) keep the model in the repo and re-check on
  protocol-level changes; (5) write a doc that maps model state
  back to implementation state so the gap is visible.

- [ ] **Step 3: `property-and-metamorphic.md`**

- *When:* you can state an invariant or relation that should hold
  across many inputs, even if computing the "right answer" for any
  one input is hard. Especially useful for query engines,
  serializers, encoders, schema systems.
- *Detects well:* algebraic-law violations (round-trip,
  commutativity, idempotency), behavior divergence between two
  implementations, edge cases hidden by example-based testing.
- *Misses:* bugs that don't violate the stated invariant; needs
  good shrinking to be debuggable.
- *Tools:* Hypothesis (Python), QuickCheck (Haskell), PropEr
  (Erlang), proptest / quickcheck (Rust), fast-check (TypeScript),
  ScalaCheck.
- *Papers:* "Metamorphic Testing: A Review of Challenges and
  Opportunities" (Chen et al., 2017).
- *Cost:* per-property runtimes are seconds to minutes; the design
  cost is mostly in stating good properties.
- *Plan checklist:* (1) list the algebraic laws and metamorphic
  relations the SUT should obey; (2) put them next to the code
  they test; (3) keep generators tight enough to actually hit
  edge cases (boundary integers, empty/huge inputs); (4) require
  shrinking on failure; (5) for config systems, treat the config
  space itself as the input domain.

- [ ] **Step 4: `performance-and-benchmarking.md`**

- *When:* you care about latency tail, throughput, fairness, or
  any "system slowed down" rather than "system gave wrong answer".
- *Detects well:* coordinated-omission lies, regressions under load,
  head-of-line blocking, fairness across tenants, GC-induced tail
  latency, limping-induced cascading slowdown.
- *Misses:* correctness bugs that don't change wall-clock,
  bugs only visible at scales you don't test.
- *Tools:* wrk2, k6, fortio, Gil Tene's HdrHistogram, vegeta,
  YCSB, custom workload generators (often the right choice). For
  analysis: Brendan Gregg's USE/RED methodology.
- *Papers:* "How NOT to Measure Latency" (Gil Tene); "Your Load
  Generator Is Probably Lying To You"; "Performance Analysis
  Methodology" (Brendan Gregg).
- *Cost:* minutes to days; bottleneck is environment realism.
- *Plan checklist:* (1) measure latency as a distribution
  (p50/p99/p99.9 + max), never as an average; (2) confirm the load
  generator does not suffer from coordinated omission; (3) hold the
  open-loop arrival rate constant — closed-loop hides queue buildup;
  (4) measure under contention, not just under one tenant;
  (5) capture system metrics alongside (CPU, GC, network) so you
  can attribute the slowdown.

- [ ] **Step 5: `crash-recovery-and-upgrade.md`**

- *When:* anything touching durability, replay, idempotency,
  or version-to-version state. This category is the leading source
  of distributed-systems incidents in studies (Gunawi et al.,
  "What Bugs Live in the Cloud").
- *Detects well:* lost writes after crash, double-apply on replay,
  duplicate side effects, mixed-version protocol bugs, schema-
  migration corruption, downgrade-incompatible state, fsync gaps.
- *Misses:* steady-state correctness (use Jepsen for that), perf
  during recovery (use perf), invariants requiring whole-history
  analysis (use Elle).
- *Tools:* power-loss simulators (ALICE, Torturing Databases
  framework, custom fsync-loss injection), rolling-upgrade test
  harnesses, replay-from-WAL tooling the SUT typically already has.
- *Papers:* "Torturing Databases for Fun and Profit" (Zheng et al.,
  OSDI'14); "An Empirical Study on Crash Recovery Bugs in Large-Scale
  Distributed Systems" (Gao et al., FSE'18); "Understanding and
  Detecting Software Upgrade Failures in Distributed Systems"
  (Zhang et al., SOSP'21).
- *Cost:* per-scenario minutes to hours; planning cost is high
  because the fault matrix (crash at every IO boundary × at every
  upgrade step) is large.
- *Plan checklist:* (1) enumerate the operation's IO boundary
  points and inject a crash at each; (2) assert post-recovery
  equivalence with a known-good baseline (replay-equivalence);
  (3) for idempotent operations, drive the retry storm explicitly;
  (4) for upgrades, test mixed-version state at every intermediate
  step, not only N→N+1; (5) treat fsync as a contract — if the
  filesystem doesn't fsync, neither does the SUT for the purposes
  of the test.

- [ ] **Step 6: Verify and commit**

```bash
cd /Users/lishen/work/distributed-testing-skills
for f in fuzzing.md formal-methods-tla.md property-and-metamorphic.md performance-and-benchmarking.md crash-recovery-and-upgrade.md; do
  path="skills/designing-distributed-system-tests/references/$f"
  test -f "$path" || { echo "MISSING $f"; exit 1; }
done
git add skills/designing-distributed-system-tests/references/
git commit -m "Add remaining technique catalog references"
```

---

## Task 6: SKILL.md for the design skill

**Files:**
- Create: `/Users/lishen/work/distributed-testing-skills/skills/designing-distributed-system-tests/SKILL.md`

- [ ] **Step 1: Write SKILL.md**

```markdown
---
name: designing-distributed-system-tests
description: Use when designing a test plan for a change to a distributed or stateful system — anything with persistence, replication, consensus, retries, idempotency, async messaging, multi-tenancy, or partial failure. Also use when asked to write a stability plan, fault matrix, release-validation plan, Jepsen plan, durability test plan, partition test plan, upgrade test plan, crash-recovery test plan, linearizability test plan, or deterministic-simulation plan. Trigger even if the user just says "what should we test for this change" and the change is in a service with the above properties. Produces a structured Markdown plan file with hypothesis-driven scenarios drawn from a curated technique catalog (Jepsen+Elle, deterministic simulation, chaos/fault injection, fuzzing, formal methods, property+metamorphic, performance, crash-recovery+upgrade).
---

# Designing Distributed-System Tests

The default for testing distributed and stateful systems — write a few
integration tests and call it done — finds a small fraction of the bugs
that actually break these systems in production. This skill enforces an
opinionated workflow: scope the change, generate failure-mode hypotheses
that cover the categories the literature says matter most, pick
techniques from a curated catalog, and emit a structured plan file that
the executing-distributed-system-tests skill (or a human) can run.

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

### 2. Scope the change

Identify the commit, PR, or feature under test. List every file
touched and the surfaces (RPCs, on-disk formats, replication
messages, public APIs) affected. Build a one-paragraph blast-radius
statement.

### 3. Generate failure-mode hypotheses

For each touched surface, generate hypotheses across these categories:
correctness, durability, liveness, partial failure, idempotency /
replay, upgrade / rollback, configuration, performance / fairness.

If a category is genuinely not applicable, say so explicitly. The act
of writing "N/A because…" surfaces wrong assumptions more often than
it sounds like it would.

### 4. Select techniques

Open `references/catalog-index.md` and find the techniques that match
your hypotheses. For each technique you pick, open its reference file
and write down in the plan: which hypotheses it addresses, what it
would catch that other techniques would miss, the typical cost.

A change usually warrants 2–4 techniques in combination. One technique
is suspicious — re-check whether you've collapsed multiple distinct
hypotheses into one.

### 5. Design scenarios

For each technique, write concrete scenarios. Each scenario must
specify: workload (what generator, rate, distribution, duration);
faults (schedule of what is injected when); oracle (the property
checked and how); observability required; exit criteria (pass / fail /
inconclusive thresholds).

Resist "logs look fine" as an oracle. The oracle must be a
machine-checkable property or a metric SLO with a defined threshold.

### 6. Write the plan file

Copy `assets/plan-template.md` to the plan destination and fill it in.
Default destination is `docs/testing-plans/<short-slug>.md` in the SUT
repo; the user may override (e.g. when they don't want the agent
writing into their repo, default to
`~/.claude/testing-plans/<short-slug>.md`).

The plan slug is the only handoff to the executing skill. Pick a
descriptive slug — `durable-idempotent-append-replay`, not
`plan-1`.

### 7. Self-check

Read the plan back. Every hypothesis has at least one scenario.
Every scenario has an oracle that is not "logs look fine". Every
chosen technique cites its reference file. If anything fails the
check, fix it in the plan; do not move on with known gaps.

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
```

- [ ] **Step 2: Verify**

```bash
path=/Users/lishen/work/distributed-testing-skills/skills/designing-distributed-system-tests/SKILL.md
test -f "$path" && head -1 "$path" | grep -q '^---' && echo "OK frontmatter" || echo "FAIL frontmatter"
wc -l "$path"
```
Expected: `OK frontmatter`, line count under 250.

- [ ] **Step 3: Commit**

```bash
cd /Users/lishen/work/distributed-testing-skills
git add skills/designing-distributed-system-tests/SKILL.md
git commit -m "Add SKILL.md for designing-distributed-system-tests"
```

---

## Task 7: Session log + findings report templates

**Files:**
- Create: `skills/executing-distributed-system-tests/assets/session-log-template.md`
- Create: `skills/executing-distributed-system-tests/assets/findings-report-template.md`

- [ ] **Step 1: Write `session-log-template.md`**

```markdown
# Session log: {{plan_slug}} @ {{UTC_timestamp}}

**SUT:** {{system_name}}
**Plan:** {{path/to/plan.md}}
**Commit under test:** {{git_sha}}
**Session dir:** {{absolute_path}}
**Operator:** {{agent / human}}

## Toolbox discovered

What test drivers, fault injectors, runbooks, and observability are
present in the SUT repo. List each with the path and what it does.
This must be filled before any scenario runs.

- `{{path}}` — {{role}}

## Preconditions check

- [ ] Cluster brought up cleanly
- [ ] Observability live (link to dashboard / metrics endpoint)
- [ ] Baseline metric captured (link to artifact)
- [ ] Fault-injection plane verified (proxy / chaos agent responsive)

## Scenario timeline

| Time (UTC) | Scenario | Event | Notes |
|---|---|---|---|
| | S1 | start | |
| | S1 | fault injected: {{type}} | |
| | S1 | oracle: {{result}} | |

(Append rows as the session runs. This is the raw timeline; the
findings report cites entries here.)

## Artifacts

- `logs/` — SUT logs per node
- `metrics/` — scraped metrics, baseline + during run
- `artifacts/` — anything else: heap dumps, op histories, packet
  captures, screenshots
- `findings/` — per-scenario verdict files and the final report
```

- [ ] **Step 2: Write `findings-report-template.md`**

```markdown
# Findings: {{plan_slug}} @ {{UTC_timestamp}}

**Plan:** {{path/to/plan.md}}
**Commit under test:** {{git_sha}}
**Session dir:** {{absolute_path}}

## Summary

One paragraph: what was run, what was found, what was inconclusive.
State the headline result first — "1 oracle violation found", or
"all 5 scenarios passed with stated coverage", or "1 scenario could
not run because of {{reason}}".

## Scenario results

| ID | Scenario | Result | Oracle | Evidence |
|---|---|---|---|---|
| S1 | {{name}} | PASS / FAIL / INCONCLUSIVE | {{property}} | {{link to session log entry + artifacts}} |

## Findings

### Finding F1: {{short title}}

- **Scenario:** S{{n}}
- **Hypothesis addressed:** H{{n}} from the plan
- **What happened:** one paragraph, factual.
- **Reproducer:** the minimal fault sequence that reproduces. If you
  did not minimize, say so and why.
- **Classification:** timing / ordering / partition / crash-recovery
  / config / upgrade / fault-handling / performance (pick the closest
  from `references/finding-classification.md`).
- **Evidence:** log excerpts, trace IDs, metric snapshots, op history
  files. Link to artifacts under the session dir.
- **Hypothesised root cause:** one paragraph; which subsystem owns it.
- **Suggested next action:** one sentence.

(Repeat per finding.)

## Coverage summary

For each hypothesis in the plan, state which scenarios exercised it
and the result. Hypotheses that were not exercised must be listed
with the reason (out of scope, blocked by missing tooling, deferred).

## Green-but-broken check

State the result of each red-flag check from
`references/green-but-broken-red-flags.md`. A scenario marked PASS
without these checks completing is not actually PASS.

- [ ] Workload generator produced the expected ops/sec throughout
- [ ] Oracle ran on every intended cycle
- [ ] Faults verifiably injected (cite evidence)
- [ ] Clock skew did not silently mask the timing assertion
- [ ] Run duration met the plan's exit criterion
```

- [ ] **Step 3: Commit**

```bash
cd /Users/lishen/work/distributed-testing-skills
git add skills/executing-distributed-system-tests/assets/
git commit -m "Add session-log and findings-report templates"
```

---

## Task 8: Oracle patterns reference

**Files:**
- Create: `skills/executing-distributed-system-tests/references/oracle-patterns.md`

- [ ] **Step 1: Write the file**

```markdown
# Oracle Patterns

An oracle is the thing that says "this run passed or failed". Without
a real oracle, you have a load generator and nothing more. Most
"green but broken" results come from oracles that did not actually
run.

## Patterns, with when to use each

### 1. Linearizability / serializability check on an op history
- **Tools:** Elle (general), Porcupine (Go), Knossos (Clojure).
- **When:** anything you claim is linearizable or serializable.
- **Inputs:** complete op history with submit + complete timestamps
  plus the value returned.
- **Output:** anomaly classification (Elle: G0, G1a, G1b, G1c,
  G-single, G2-item, G2) or a counter-example interleaving.
- **Failure mode:** truncated history is silently incomplete — the
  checker can't see what it doesn't have. Always cross-check op
  counts before trusting a PASS.

### 2. Property assertion
- **When:** an invariant holds at every step or at quiescence.
- **How:** assert in code during the run, or assert offline against
  recorded state.
- **Failure mode:** the assertion didn't run because the path
  wasn't taken. Cite execution evidence ("assertion fired N times,
  passed N").

### 3. Replay equivalence
- **When:** crash recovery, idempotent retries, fork operations.
- **How:** snapshot state A, perform op, snapshot B, replay from A,
  compare to B. They must match (allowing for documented
  non-determinism).
- **Failure mode:** the comparison is too lenient — e.g. ignores
  timestamps but should not.

### 4. Metric SLO threshold
- **When:** performance / fairness / availability scenarios.
- **How:** define p99 latency, error rate, throughput floor before
  the run. PASS = within threshold across the measurement window.
- **Failure mode:** measurement window too short, or threshold set
  from the same run that measured the baseline.

### 5. Statistical comparison against baseline
- **When:** "did this regress?" rather than "is it correct?".
- **How:** capture baseline run, capture candidate run, apply a
  test that controls false-positive rate (e.g. bootstrap CI on
  percentile of interest).
- **Failure mode:** comparing a single run against a single run.
  Always replicate.

### 6. Cross-implementation differential
- **When:** you have a reference implementation (or a model) to
  compare against.
- **How:** send the same ops to both, diff outputs.
- **Failure mode:** the reference has the same bug.

## Required for every oracle

- **Cite execution evidence in the report.** Echo the actual fact
  that the oracle ran — not just the verdict. ("Elle processed
  4,217 ops, 0 anomalies." "Property `monotonic_offset` fired
  on every append: 12,401 checks, 0 failures.")
- **Declare the oracle's scope.** Say what it cannot detect.
```

- [ ] **Step 2: Commit**

```bash
cd /Users/lishen/work/distributed-testing-skills
git add skills/executing-distributed-system-tests/references/oracle-patterns.md
git commit -m "Add oracle patterns reference for executing skill"
```

---

## Task 9: Fault-injection how-to reference

**Files:**
- Create: `skills/executing-distributed-system-tests/references/fault-injection-howto.md`

- [ ] **Step 1: Write the file**

```markdown
# Fault Injection: Practical How-To

Faults must (a) actually fire, (b) produce evidence they fired, and
(c) be reversible so the next scenario starts clean.

## Process / lifecycle faults

| Fault | Mechanism | Evidence of injection | Cleanup |
|---|---|---|---|
| Hard kill | `kill -9 <pid>` | exit log line, restart timestamp | none (process gone) |
| Graceful stop | `kill -TERM` | shutdown log lines | none |
| GC pause / freeze | `kill -STOP` … `kill -CONT` | gap in heartbeat log | `-CONT` |
| Container restart | `docker restart` / k8s pod kill | container event | wait for ready |

## Network faults

| Fault | Mechanism | Evidence | Cleanup |
|---|---|---|---|
| Full partition (A↔B) | iptables drop, `tc` qdisc, Toxiproxy disable | counters on dropper, RPC timeouts on both sides | reverse iptables / re-enable proxy |
| Asymmetric partition (A→B only) | iptables drop on one direction | one-side timeouts only | reverse |
| Packet loss | `tc qdisc add … netem loss 30%` | netem stats, RPC retries | `tc qdisc del` |
| Latency injection | `tc … netem delay 200ms` | RPC latency histogram shift | `tc qdisc del` |
| Bandwidth cap | `tc … htb rate 1mbit` | throughput drop | `tc qdisc del` |

**Always verify the fault landed.** A common failure: `iptables`
rule lives in the wrong chain, or `tc` qdisc is on the wrong
interface. Capture stats from the dropper itself and from one
victim and put both in the session log.

## Storage faults

| Fault | Mechanism | Evidence | Cleanup |
|---|---|---|---|
| Disk full | fill the FS with a junk file | write errors in SUT log | `rm` the junk |
| fsync loss / power loss | torturing-databases-style block tracer, or filesystem with `nobarrier`, or simulated via process kill between write and fsync | crash, mismatch on recovery | reset disk image |
| Slow disk | `cgroup` IO throttling | IO latency histogram shift | remove throttle |
| Bit flip on read / write | dm-flakey, dm-error, or in-app injection | checksum failures | remove dm target |

## Time faults

| Fault | Mechanism | Evidence | Cleanup |
|---|---|---|---|
| Clock skew | `date -s` inside container, or libfaketime | timestamp drift in logs | reset clock / drop libfaketime |
| Slow clock | libfaketime with rate | derived metrics drift | drop |

## Cluster-level faults

- **Rolling restart, wrong order:** state-machine bugs that hide
  when restart order matches leadership order.
- **Split-brain:** combine partition with leadership timeout pressure.
- **Slow follower:** latency injection on one node only — exposes
  back-pressure and head-of-line blocking.

## Anti-patterns

- "Inject a fault and wait 5 seconds." The fault may take longer
  to propagate; use an event (RPC timeout observed, replica marked
  down) rather than wall-clock.
- "Reverse the fault and immediately check correctness." Recovery
  takes time; gate the oracle on quiescence, not on the moment of
  un-injection.
- "Trust that the chaos agent did the thing." Always cite proof.
```

- [ ] **Step 2: Commit**

```bash
cd /Users/lishen/work/distributed-testing-skills
git add skills/executing-distributed-system-tests/references/fault-injection-howto.md
git commit -m "Add fault-injection how-to reference"
```

---

## Task 10: Test-case reduction, finding classification, and red flags

**Files:**
- Create: `skills/executing-distributed-system-tests/references/test-case-reduction.md`
- Create: `skills/executing-distributed-system-tests/references/finding-classification.md`
- Create: `skills/executing-distributed-system-tests/references/green-but-broken-red-flags.md`

- [ ] **Step 1: `test-case-reduction.md`**

```markdown
# Test-case Reduction

A 3-hour distributed run that hits a bug is hard to act on; reduce
it to the smallest fault sequence that still reproduces.

## When to reduce
- Always for FAIL findings before filing.
- Not for INCONCLUSIVE — fix the inconclusiveness first.

## Approach (after Scott et al., MCS, NSDI'16)

1. **Capture the full event log:** all faults, all client ops, with
   timestamps, in order.
2. **Bisect by halves:** drop the first half of events. Still
   reproduces? Recurse on the second half. Doesn't reproduce?
   Keep first half and recurse.
3. **Drop one event at a time** from the reduced set; if the bug
   still reproduces without that event, leave it out.
4. **Stop when:** removing any single remaining event makes the
   bug stop reproducing.
5. **Record the budget:** "reduced from N events to k events over
   M re-runs" goes in the finding.

## When to give up
- Bug requires a rare interleaving that does not reproduce on
  every run. Record what you tried, mark "non-deterministic" in
  the finding, and either change tactic (DST / deterministic
  scheduler) or accept the larger reproducer.

## Avoid over-reducing
- Reduction can drop events that are causally required but happen
  to not change the outcome on a given seed. Keep at least one
  event from each fault category in the reproducer if the bug
  depends on the category.
```

- [ ] **Step 2: `finding-classification.md`**

```markdown
# Finding Classification

Use this single-page taxonomy (after TaxDC, Leesatapornwongsa et al.,
ASPLOS'16) to label every finding. Consistent labels make findings
searchable across runs and across projects.

## Top-level category (pick one)

- **Timing** — depends on relative timing of events (slow node,
  GC pause, scheduling delay). Often non-deterministic.
- **Ordering** — incorrect handling of message / event order.
- **Partition** — partial / asymmetric / full network split.
- **Crash-recovery** — incorrect state after process death + restart.
- **Upgrade** — mixed-version cluster or migration bug.
- **Config** — bad / default / changed configuration silently
  changes behavior.
- **Fault-handling** — bug *in* the error-handling code path itself
  (most common per Yuan et al. OSDI'14).
- **Performance** — correct but slow; tail / fairness / throughput.

## Secondary tags (any that apply)

- **Replication** — propagated to / between replicas.
- **Leadership / election** — election timing or split-brain.
- **Idempotency / dedup** — duplicate-application bug.
- **Durability** — lost data after acknowledged write.
- **Liveness** — system never recovers.
- **Safety** — system returns wrong answer (treat as P0 by default).

## Severity (engineering judgement, not part of TaxDC)

- **P0** — data loss, data corruption, or safety violation.
- **P1** — availability loss; recovery requires operator action.
- **P2** — observable degradation; auto-recovers.
- **P3** — no user impact; minor anomaly worth noting.
```

- [ ] **Step 3: `green-but-broken-red-flags.md`**

```markdown
# Green-but-Broken Red Flags

A scenario that "passed" without these checks completing did not
actually pass. Run this list before declaring a scenario PASS.

1. **Workload actually generated the intended load.** Throughput
   chart shows the expected ops/sec for the expected duration.
   If the generator silently stopped or rate-collapsed, the test
   tells you nothing.
2. **Oracle actually ran on every intended cycle.** Cite the
   number of times the property fired or the number of ops the
   linearizability checker consumed.
3. **Faults actually injected.** Cite evidence from the injector
   side AND from at least one victim side. "Toxiproxy was enabled"
   is not evidence; "RPC timeout count went from 0/s to 142/s
   over the injection window" is.
4. **Fault did not silently no-op.** Common: iptables rule in
   the wrong chain, tc qdisc on the wrong interface, container
   restart that the orchestrator immediately reversed.
5. **Clock skew did not mask a timing assertion.** If the
   assertion depends on monotonic time, the timing source must
   be monotonic — not wall clock.
6. **Run duration met the plan's exit criterion.** Short runs
   often miss bugs that only manifest after sustained pressure.
7. **No silent error suppression.** Grep the SUT log for
   exception / panic / "Internal error" lines that the oracle
   did not see because they were swallowed by a higher layer.
8. **Recovery completed.** Post-fault, the SUT returned to
   nominal — not "stayed up but in degraded mode that the oracle
   didn't recognize as degraded".
9. **Baseline comparison is fair.** Comparing today's run to
   last week's baseline is fine only if nothing else has changed
   in the harness. Re-baseline when the harness changes.
10. **One PASS != PASS.** If the test is statistical, replicate.
```

- [ ] **Step 4: Commit**

```bash
cd /Users/lishen/work/distributed-testing-skills
git add skills/executing-distributed-system-tests/references/
git commit -m "Add test-case reduction, finding classification, and red-flags references"
```

---

## Task 11: SKILL.md for the execute skill

**Files:**
- Create: `/Users/lishen/work/distributed-testing-skills/skills/executing-distributed-system-tests/SKILL.md`

- [ ] **Step 1: Write SKILL.md**

```markdown
---
name: executing-distributed-system-tests
description: Use when running a previously designed distributed-systems test plan against a real or simulated cluster — driving fault injection, workload, chaos scenarios, Jepsen-style consistency runs, durability tests, partition tests, crash-recovery tests, upgrade tests, performance/SLO runs, or release validation. Also use when asked to "execute the plan", "reproduce a distributed bug", "run stability tests", "drive chaos", "validate a release end-to-end", or when a plan file exists at docs/testing-plans/ or ~/.claude/testing-plans/ and needs to be run. Discovers the SUT's existing test toolbox (tools/, scripts/, runbooks) and uses it rather than reinventing, produces a session directory of raw artifacts and a structured findings report classified per TaxDC.
---

# Executing Distributed-System Tests

Pairs with `designing-distributed-system-tests`. That skill produces
a plan; this skill runs it. The two communicate only through
filesystem artifacts: the plan file in and a session directory
plus findings report out.

The most common failure mode this skill is built to avoid: a run
that produces a green checkmark without anyone having checked
that the workload, the fault, and the oracle each did their job.
The "green-but-broken" checks are not optional.

## Process

### 1. Load the plan

If a plan file path was supplied, read it. If the user described
a plan in conversation, extract the scenario list. If the plan
is missing oracles or exit criteria, halt — hand back to the
design skill rather than improvise. Improvising an oracle in the
moment is how green-but-broken results get produced.

### 2. Discover the SUT toolbox

Search the repo before writing any new code. Look for:
- `tools/`, `scripts/`, `bin/` — drivers, workload generators,
  cluster bring-up scripts
- `tests/integration/`, `tests/stability/`, `tests/chaos/`
- `docs/runbooks/`, `docs/testing/`, `docs/stability-test-plan.md`
- Makefile / justfile / `cargo xtask` targets that look like
  cluster commands
- existing CI definitions that already wire fault-injection

Record what you found in the session log under "Toolbox
discovered". This is required before any scenario runs — it
prevents the skill from re-inventing tools that already exist.

### 3. Establish a session directory

Create:
```
{{session_root}}/{{plan_slug}}/{{UTC_timestamp}}/
├── logs/
├── metrics/
├── artifacts/
└── findings/
```
Default `session_root` is `./test-sessions/` in the SUT repo or
`~/.claude/test-sessions/` if the user prefers not to write into
the repo. Place a copy of `assets/session-log-template.md` at
`session-log.md` inside this directory and fill in the header.

### 4. Run scenarios in plan order

For each scenario:

1. **Preconditions check.** Cluster up cleanly, observability
   live, baseline metric captured, fault plane responsive.
2. **Start workload** using the discovered driver.
3. **Inject fault** per the plan schedule.
4. **Capture evidence the fault landed** (counter, RPC timeout
   pattern, log line). If you cannot prove it landed, the
   scenario is INCONCLUSIVE, not PASS.
5. **Stop / quiesce** and collect.
6. **Apply oracle** — read `references/oracle-patterns.md` for
   the right one if the plan didn't fully specify.
7. **Record the verdict** with the actual oracle execution evidence,
   not just the verdict.

### 5. On failure: capture before moving on

Do not run the next scenario before recording:
- Reproducer (apply `references/test-case-reduction.md`).
- Classification (`references/finding-classification.md`).
- Evidence (log excerpts, op history files, metric snapshots).
- Hypothesised root cause and owning subsystem.
- Suggested next action.

### 6. Apply green-but-broken checks

Before declaring any scenario PASS, run the list in
`references/green-but-broken-red-flags.md` and record the result
of each check in the findings report.

### 7. Write the findings report

Copy `assets/findings-report-template.md` to
`{{session_dir}}/findings/report.md` and fill it in. Lead with
the headline result. Cover every plan hypothesis in the coverage
section, even the ones not exercised — those are gaps worth
naming.

## Project autonomy

This skill does not modify SUT source files. It may create
scripts inside the session directory and may add a single new
file `docs/testing-plans/<slug>.md` only if the user explicitly
asks the design skill to write there. Everything else lives
under `{{session_root}}`.

## Reference files

- `references/oracle-patterns.md` — start here when an oracle
  needs picking
- `references/fault-injection-howto.md` — concrete mechanisms per
  fault type
- `references/test-case-reduction.md` — minimize a failing
  reproducer
- `references/finding-classification.md` — TaxDC-derived labels
- `references/green-but-broken-red-flags.md` — non-optional
  pre-PASS checklist

## Assets

- `assets/session-log-template.md`
- `assets/findings-report-template.md`
```

- [ ] **Step 2: Verify**

```bash
path=/Users/lishen/work/distributed-testing-skills/skills/executing-distributed-system-tests/SKILL.md
test -f "$path" && head -1 "$path" | grep -q '^---' && echo "OK frontmatter" || echo "FAIL frontmatter"
wc -l "$path"
```
Expected: `OK frontmatter`, line count under 250.

- [ ] **Step 3: Commit**

```bash
cd /Users/lishen/work/distributed-testing-skills
git add skills/executing-distributed-system-tests/SKILL.md
git commit -m "Add SKILL.md for executing-distributed-system-tests"
```

---

## Task 12: Eval prompts (skill-creator format)

**Files:**
- Create: `/Users/lishen/work/distributed-testing-skills/evals/designing/evals.json`
- Create: `/Users/lishen/work/distributed-testing-skills/evals/executing/evals.json`

- [ ] **Step 1: Write `evals/designing/evals.json`**

```json
{
  "skill_name": "designing-distributed-system-tests",
  "evals": [
    {
      "id": 1,
      "prompt": "I just merged a fix titled 'Fix durable idempotent append replay' on a Rust distributed agent runtime (AgentDB) at /Users/lishen/work/agentdb-dst-verify. The fix touches the append + replay paths. Design a stability test plan for this change. Save it to docs/testing-plans/durable-idempotent-append-replay.md in that repo. Use the project's own runbooks and tooling where possible.",
      "expected_output": "A plan file at the named path covering at minimum: append idempotency under client retry storm, replay equivalence after crash mid-append, duplicate suppression under asymmetric partition, durability after simulated fsync loss, concurrent forks observing the same append. Plan must cite catalog references and the project's own docs/runbooks/fault_injected.md and tools/agentdb-cluster-smoke.",
      "files": []
    },
    {
      "id": 2,
      "prompt": "I have a small Raft-based key-value store written in Go (~3k lines, no replication beyond the Raft library itself). I'm about to ship a new feature that lets clients change a key's TTL with the same idempotency guarantee the existing PUT has. Design a focused test plan for this feature. I do not have Jepsen integration yet — propose what to use without assuming Jepsen.",
      "expected_output": "A plan that uses property-based testing for the TTL semantics, deterministic-simulation-style scheduling (or a clear acknowledgement that DST requires plumbing the library doesn't have yet), crash-recovery scenarios across the new IO boundary, and a note that linearizability checking with Elle/Porcupine over op histories would be the natural follow-up if/when Jepsen integration exists. Should NOT propose 'just run Jepsen' as if the user can.",
      "files": []
    },
    {
      "id": 3,
      "prompt": "Quick design check — I'm changing a typo in a log line and removing two unused imports in a stateless HTTP handler. What's the right test plan?",
      "expected_output": "Early-exit. The skill should explicitly say 'this change does not warrant a distributed test plan' and recommend the lightest appropriate testing (e.g. existing unit tests, lint). Should NOT produce a ceremonial multi-scenario plan.",
      "files": []
    }
  ]
}
```

- [ ] **Step 2: Write `evals/executing/evals.json`**

```json
{
  "skill_name": "executing-distributed-system-tests",
  "evals": [
    {
      "id": 1,
      "prompt": "Execute the test plan at /Users/lishen/work/agentdb-dst-verify/docs/testing-plans/durable-idempotent-append-replay.md against AgentDB at the same path. Produce a session directory and findings report under ~/work/distributed-testing-skills/verification/agentdb-fab7d9d/.",
      "expected_output": "Session directory with logs/metrics/artifacts/findings subdirs. Session log shows the SUT toolbox was discovered (tools/agentdb-cluster-smoke, tools/workload, docs/runbooks/fault_injected.md) before any scenario ran. Findings report follows the template, has a headline result, has the green-but-broken checks completed and cited per scenario, and either reports at least one violation with a reproducer + classification OR explicitly states what coverage was achieved with PASSes.",
      "files": []
    },
    {
      "id": 2,
      "prompt": "I have a plan file at /tmp/web-queue-plan.md that defines two scenarios: a partition between web tier and queue, and a queue node crash mid-publish. Execute it against a Docker Compose stack at /Users/lishen/work/web-queue-demo. There's no fault-injection runbook in that repo.",
      "expected_output": "Skill discovers the toolbox is sparse and either (a) uses Toxiproxy / docker network commands and cites that decision in the session log, or (b) halts and asks for permission to install fault tooling rather than silently inventing. Either way, evidence of injection is captured for each scenario before declaring PASS.",
      "files": []
    },
    {
      "id": 3,
      "prompt": "Run the plan at /tmp/plan-without-oracles.md (this plan has scenarios but the oracle column says 'logs look fine' for every scenario).",
      "expected_output": "Skill halts at the load-the-plan step and hands back to the design skill, citing that oracles are missing or non-machine-checkable. Should NOT improvise oracles and run the plan anyway.",
      "files": []
    }
  ]
}
```

- [ ] **Step 3: Commit**

```bash
cd /Users/lishen/work/distributed-testing-skills
git add evals/
git commit -m "Add eval prompts for both skills"
```

---

## Task 13: AgentDB verification — design skill pass

This task and Task 14 are the integration test for both skills.
They run against a fresh worktree of AgentDB.

**Files:**
- Create worktree: `/Users/lishen/work/agentdb-dst-verify/`
- Output expected: `/Users/lishen/work/agentdb-dst-verify/docs/testing-plans/durable-idempotent-append-replay.md`

- [ ] **Step 1: Create the worktree**

```bash
cd /Users/lishen/work/agentdb
git worktree add /Users/lishen/work/agentdb-dst-verify main
cd /Users/lishen/work/agentdb-dst-verify
git checkout -b dst-verify-fab7d9d
```
Expected: worktree present, on a fresh branch off `main`.

- [ ] **Step 2: Cherry-pick the change under test**

The fix commit is `fab7d9d` on `stability-test`. Cherry-pick it
onto the verify branch so the worktree contains the change to
plan against.

```bash
cd /Users/lishen/work/agentdb-dst-verify
git cherry-pick fab7d9d
```
Expected: clean cherry-pick. If conflicts, resolve minimally and
note in the verification log.

- [ ] **Step 3: Install both skills locally**

```bash
mkdir -p ~/.claude/skills
ln -snf /Users/lishen/work/distributed-testing-skills/skills/designing-distributed-system-tests ~/.claude/skills/designing-distributed-system-tests
ln -snf /Users/lishen/work/distributed-testing-skills/skills/executing-distributed-system-tests ~/.claude/skills/executing-distributed-system-tests
ls -l ~/.claude/skills/ | grep -E 'designing|executing'
```
Expected: two symlinks present.

- [ ] **Step 4: Invoke the design skill via a subagent**

Spawn a fresh Claude Code subagent (no prior context) with the
eval-1 prompt for the designing skill, against the worktree. The
subagent must invoke `designing-distributed-system-tests` and
produce the plan file.

Prompt verbatim:

> I just merged a fix titled "Fix durable idempotent append replay"
> on a Rust distributed agent runtime (AgentDB) at
> `/Users/lishen/work/agentdb-dst-verify`. The fix is the
> latest commit on this branch. Design a stability test plan for
> this change. Save it to
> `docs/testing-plans/durable-idempotent-append-replay.md` in that
> repo. Use the project's own runbooks and tooling where possible.

- [ ] **Step 5: Verify the plan exists and is non-trivial**

```bash
path=/Users/lishen/work/agentdb-dst-verify/docs/testing-plans/durable-idempotent-append-replay.md
test -f "$path" || { echo "FAIL: plan not produced"; exit 1; }
lines=$(wc -l < "$path")
echo "Plan lines: $lines"
test "$lines" -ge 80 || { echo "FAIL: plan too short ($lines lines)"; exit 1; }
grep -q "idempotency" "$path" && echo "OK: covers idempotency" || echo "GAP: no idempotency mention"
grep -q -i "fsync\|durability" "$path" && echo "OK: covers durability" || echo "GAP: no durability mention"
grep -q -i "replay" "$path" && echo "OK: covers replay" || echo "GAP: no replay mention"
grep -q -i "partition" "$path" && echo "OK: covers partition" || echo "GAP: no partition mention"
grep -q -i "fault_injected\|cluster-smoke\|workload" "$path" && echo "OK: references project tooling" || echo "GAP: missed project tooling"
```
Expected: all `OK`s. Any `GAP` is a finding to feed back into a
skill iteration (Task 15).

- [ ] **Step 6: Commit the plan inside the worktree**

```bash
cd /Users/lishen/work/agentdb-dst-verify
git add docs/testing-plans/durable-idempotent-append-replay.md
git commit -m "Verification: stability plan for durable idempotent append replay"
```

- [ ] **Step 7: Record the verification result**

```bash
mkdir -p /Users/lishen/work/distributed-testing-skills/verification/agentdb-fab7d9d
cp /Users/lishen/work/agentdb-dst-verify/docs/testing-plans/durable-idempotent-append-replay.md \
   /Users/lishen/work/distributed-testing-skills/verification/agentdb-fab7d9d/design-output.md
echo "Design-skill pass recorded. See design-output.md." > \
   /Users/lishen/work/distributed-testing-skills/verification/agentdb-fab7d9d/README.md
cd /Users/lishen/work/distributed-testing-skills
git add verification/agentdb-fab7d9d/
git commit -m "Verification: capture design-skill output for AgentDB fab7d9d"
```

---

## Task 14: AgentDB verification — execute skill pass

**Files:**
- Output expected: `/Users/lishen/work/distributed-testing-skills/verification/agentdb-fab7d9d/findings/report.md`

- [ ] **Step 1: Confirm preconditions**

```bash
test -f /Users/lishen/work/agentdb-dst-verify/docs/testing-plans/durable-idempotent-append-replay.md && echo OK || { echo "FAIL: plan missing from Task 13"; exit 1; }
test -L ~/.claude/skills/executing-distributed-system-tests && echo OK || { echo "FAIL: execute skill not installed"; exit 1; }
```
Expected: both OK.

- [ ] **Step 2: Invoke the execute skill via a subagent**

Spawn a fresh Claude Code subagent with the eval-1 prompt for
the executing skill. The subagent must:
1. Invoke `executing-distributed-system-tests`.
2. Discover the AgentDB toolbox before running anything.
3. Produce a session directory under
   `/Users/lishen/work/distributed-testing-skills/verification/agentdb-fab7d9d/test-sessions/`.
4. Produce `findings/report.md` at the verification root.

Prompt verbatim:

> Execute the test plan at
> `/Users/lishen/work/agentdb-dst-verify/docs/testing-plans/durable-idempotent-append-replay.md`
> against AgentDB at the same path. Produce a session directory
> and findings report under
> `/Users/lishen/work/distributed-testing-skills/verification/agentdb-fab7d9d/`.

Note: the subagent may not be able to run a full chaos cluster
in this environment. That is acceptable for verification of the
skill's *behavior* — what we need is evidence the skill (a)
discovered the right toolbox, (b) used the right templates,
(c) ran the green-but-broken checks, (d) produced an honest
report (PASS with explicit coverage, FAIL with reproducer, or
INCONCLUSIVE with reason — all three are acceptable verdicts;
silent PASS without evidence is not).

- [ ] **Step 3: Verify the report exists and is honest**

```bash
report=/Users/lishen/work/distributed-testing-skills/verification/agentdb-fab7d9d/findings/report.md
test -f "$report" || { echo "FAIL: report not produced"; exit 1; }
grep -q -i "toolbox discovered\|tools/agentdb-cluster-smoke\|tools/workload" "$report" && echo "OK: toolbox discovery cited" || echo "FAIL: toolbox discovery missing"
grep -q -i "green-but-broken\|red flag\|oracle.*evidence\|injection.*evidence" "$report" && echo "OK: integrity checks cited" || echo "FAIL: integrity checks missing"
grep -q -E "PASS|FAIL|INCONCLUSIVE" "$report" && echo "OK: verdict present" || echo "FAIL: no verdict"
```
Expected: all OK.

- [ ] **Step 4: Commit verification artifacts**

```bash
cd /Users/lishen/work/distributed-testing-skills
git add verification/agentdb-fab7d9d/
git commit -m "Verification: capture execute-skill output for AgentDB fab7d9d"
```

---

## Task 15: Iterate based on verification, then finalize

**Files (potential edits):**
- Modify: any SKILL.md or reference file that the verification surfaced as
  insufficient.
- Create: `/Users/lishen/work/distributed-testing-skills/plugin.json`

- [ ] **Step 1: Read verification outputs with fresh eyes**

Read `verification/agentdb-fab7d9d/design-output.md` and
`verification/agentdb-fab7d9d/findings/report.md` end-to-end.
Compare against the eval `expected_output` fields. List concrete
gaps.

- [ ] **Step 2: Apply skill-creator's iteration loop if gaps exist**

For each gap:
- Locate the cause in the relevant SKILL.md or reference file.
- Edit the file with a targeted change explaining the *why*, not
  just the *what*.
- Re-run only the affected eval (Task 13 or Task 14) for that
  iteration. Repeat until the verification passes the checks in
  Task 13 Step 5 and Task 14 Step 3 cleanly.

If a single gap requires three iterations without convergence,
stop and surface it in `verification/agentdb-fab7d9d/README.md`
as an open issue. Don't keep grinding.

- [ ] **Step 3: Write `plugin.json`**

```json
{
  "name": "distributed-testing-skills",
  "version": "0.1.0",
  "description": "Two coupled skills for designing and executing distributed-systems test plans, backed by a curated technique catalog.",
  "skills": [
    "skills/designing-distributed-system-tests",
    "skills/executing-distributed-system-tests"
  ],
  "author": "Li Shen",
  "license": "MIT"
}
```

- [ ] **Step 4: Expand `README.md` with verification and usage**

Add sections to `README.md`:
- "Usage" — how to symlink/install the skills into `~/.claude/skills/`.
- "Verification" — link to `verification/agentdb-fab7d9d/` with the
  result summary.
- "Catalog" — list the eight technique references with one-line
  summaries.
- "Acknowledgements" — Andrey Satarin's
  `testing-distributed-systems` repo as the source for the catalog
  curation; cite the seminal papers used.

- [ ] **Step 5: Final verification and commit**

```bash
cd /Users/lishen/work/distributed-testing-skills
test -f plugin.json && python3 -c "import json; json.load(open('plugin.json'))" && echo "OK plugin.json valid"
test -f README.md && grep -q "Verification" README.md && echo "OK README has verification section"
git status
git add plugin.json README.md
git commit -m "Finalize plugin manifest and verification-aware README"
git log --oneline | head -20
```
Expected: both OK, clean working tree, commit history shows the
build-up across all tasks.

---

## Self-Review Notes

Ran the checklist:

1. **Spec coverage:** every spec section maps to tasks above —
   plugin layout (Task 1), plan template (2), catalog
   (3–5), design SKILL (6), execute templates (7), execute
   references (8–10), execute SKILL (11), evals (12), AgentDB
   verification both passes (13–14), iteration and packaging (15).
2. **No placeholders:** no TBDs, no "add error handling", no "similar
   to Task N". Each reference file has its concrete content scaffolded.
3. **Type / name consistency:** skill names, plan slug, file paths,
   and the session-directory layout are identical across every
   reference. `green-but-broken` is one term, used consistently.
   `findings/report.md` is the report path everywhere.
4. **Bite-sized steps:** each step is 2–5 minutes of work; tasks
   are sequenced so each commit is small.

One acknowledged compromise: catalog reference files (Tasks 4, 5) and
template files (Tasks 2, 7) are full content blocks embedded in the
plan rather than three-line stubs. They are long, but inlining them
is the only way to honor the "no placeholders, complete content in
every step" rule for prose deliverables.

---

## Execution Handoff

This plan is ready for execution.
