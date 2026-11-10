# Distributed Systems Testing Skills

Two skills that help an AI coding agent **design meaningful tests, run them,
and explain the results** for systems that store data or run across multiple
processes or machines.

Start with a product promise—such as “once saving succeeds, the data is still
there after a restart”—and turn it into tests your agent can run and check.
Use these skills for databases, queues, workflow engines, and services where
retries, simultaneous requests, or failures could lose or duplicate data.
A service running on one machine can benefit too.

You do not need to be a testing expert. Describe the feature and what should
remain true; the design skill helps choose the tests and explain why they matter.

The skills are Markdown instructions, templates, and reference material.
Your coding agent follows them using your project's test tools and runtime.
This repository does not bundle a database test runner or a ready-made cluster.

| Skill | Give it | It produces |
|---|---|---|
| [Design tests](skills/designing-distributed-system-tests/SKILL.md) | A codebase and the change or feature you want to test | A plan explaining what to test, how to check it, and how much testing to do |
| [Execute tests](skills/executing-distributed-system-tests/SKILL.md) | A plan and a test environment | Results, supporting logs and data, steps to reproduce failures, and a report |

You can use the design skill on its own. Execution is a separate step that
reads the saved plan, so you can review the work and its cost before running it.

Start with the [quick start](#quick-start), [Codex setup](#for-codex-users),
or [test environment guide](#choose-and-prepare-a-test-environment).

## Quick start

### 1. Install the skills

Paste this into your coding agent:

```text
Read https://raw.githubusercontent.com/shenli/distributed-system-testing/main/INSTALL.md
and follow the instructions to install or update
distributed-testing-skills for this agent.
```

The [installation guide](INSTALL.md) uses a shared checkout at
`~/.local/share/distributed-testing-skills/` and configures pointers or
symlinks for your agent. Reuse the same prompt to update. The explicit paths
in the prompts below also let you load a skill without automatic discovery.
Substitute your install path if you chose a different location.

For Claude Code, the repository also includes a plugin installation option:

```text
/plugin marketplace add shenli/distributed-system-testing
/plugin install distributed-testing-skills@distributed-testing-skills
```

### 2. Design a plan in your project's directory

Start your agent in the repository you want to test. For a specific change:

```text
Use ~/.local/share/distributed-testing-skills/skills/designing-distributed-system-tests/SKILL.md

Design a test plan for the current branch's changes to adding log entries,
retrying failed requests, and recovering after a crash.
Reuse the existing tests and setup instructions.
Save the plan to docs/testing-plans/append-recovery.md.
```

Replace the example feature with yours. For a broader review, ask for a
**project-wide plan** and name the parts to include. That mode reviews existing
tests and identifies missing coverage across the system rather than one change.

Review the promises being checked, proposed tests, required tools, and run times.
A documentation typo or similarly low-impact change should get a lightweight
testing recommendation instead of a distributed test plan.

### 3. Run the plan against a disposable test environment

```text
Use ~/.local/share/distributed-testing-skills/skills/executing-distributed-system-tests/SKILL.md

Run the small initial checks (the smoke tier) from
docs/testing-plans/append-recovery.md against this project's local test
environment. Keep temporary test scripts and results under
./test-sessions/. Use default mode; do not modify project source files.
Report what ran, what failed, and what remains untested.
```

The execution skill discovers existing tools, asks what is available in your
environment, then verifies the requirements. Missing dependencies or ineffective
simulated failures must be reported rather than silently counted as successful
tests. Start small, then request the longer, more thorough runs defined in the plan.

See [USAGE.md](USAGE.md) for detailed prompts, custom output paths, and author mode.

## For Codex users

Run the installation prompt above in Codex. The guide links both skill folders
from the shared checkout into `~/.agents/skills/`, where Codex discovers them.
For a project-only installation, use that project's `.agents/skills/` instead.
These locations and symlink support are described in the
[official Codex skills documentation](https://developers.openai.com/codex/skills).

Open the project you want to test. In Codex CLI or the IDE extension, use
`/skills` to check that both skills appear, or type `$` to select one. If a newly
installed skill does not appear, restart Codex. You can also use the explicit
file-path prompts in the quick start. See the
[official invocation guidance](https://developers.openai.com/codex/skills#how-codex-uses-skills).

For example, ask Codex to design a plan:

```text
$designing-distributed-system-tests
Design a test plan for this project's retry and recovery behavior.
I have Docker locally and can run disposable containers.
Start with normal operation, then simultaneous requests and restarts.
Save the plan to docs/testing-plans/retry-recovery.md.
```

After reviewing it, run the initial tests:

```text
$executing-distributed-system-tests
Run the smoke tier of docs/testing-plans/retry-recovery.md.
Use default mode and save results under ./test-sessions/.
Docker is available; use the project's documented container setup.
Report any missing tools or permissions and which tests they prevent.
```

Installing a skill supplies instructions, not Docker, remote access, or a test
cluster. Codex must be able to reach the environment where commands will run.
For remote testing, give it the test hosts, your existing connection method,
and the project's setup commands. If access is unavailable, it can still
design the plan, but cannot report that remote tests ran.

## Choose and prepare a test environment

Use the smallest setup that can answer your question. These are starting
points to adapt to your application, not hardware requirements or bundled
deployment recipes.

| Goal | Suggested setup | What it can tell you |
|---|---|---|
| Find correctness bugs quickly | Your laptop, the project's runtime, and disposable local processes or containers | Whether requests, retries, transactions, and restarts behave correctly in that setup |
| Compare performance between versions | A quiet, dedicated test machine; a separate machine to send requests when measuring a service | Whether a change affects speed or resource use under the same conditions |
| Test failures across machines | Separate test hosts or VMs, one server instance per host, plus a separate test controller | How the system handles one server stopping or communication between servers breaking |

### Local correctness tests

Start with the project's existing setup instructions. Install its required
language/runtime and, if it uses containers, Docker or another supported
container runtime. Give each instance its own ports and data directory or
volume. Use a dedicated test database and a repeatable way to reset its data.

For containers that communicate by name, a user-defined Docker network can
provide name resolution between them. See [Docker networking](https://docs.docker.com/engine/network/).
Several containers on one laptop are useful for exercising communication, but
still share the host's resources and failure risks. They do not establish
resilience to losing a separate machine.

### Performance tests

A **load generator** is the program sending test requests. Prefer the project's
existing benchmark; for example, PostgreSQL's
[pgbench](https://www.postgresql.org/docs/current/pgbench.html) supports repeated
transactions and custom SQL scripts. Choose requests and data sizes that resemble
your application, rather than treating a tool's default workload as representative.

1. **Make comparisons fair.** Use the same hardware, data, settings, and test
   script for the old and new versions. Avoid unrelated builds or heavy jobs
   on the test machines. Record CPU, memory, disk, software versions, and settings.
2. **Separate startup from measurement.** Give caches and connections time to
   warm up, then measure a fixed window. Test cold starts separately if they
   matter. Repeat each version's run several times and report the variation.
3. **Increase traffic gradually.** Record requests attempted and completed per
   second, errors, and timeouts. Monitor the load generator too: if it is at
   capacity, the test may be measuring the client rather than the service.
4. **Look beyond averages.** Report p50, p95, and p99 response times—the times
   within which 50%, 95%, and 99% of requests finish. Capture CPU, memory,
   network, and disk use alongside them, and keep data-correctness checks active.
5. **Match how traffic arrives.** If production requests keep arriving while
   the service slows down, use a generator that can maintain that arrival rate.
   Waiting for each response before sending more can hide overload. The
   [k6 traffic-model guide](https://grafana.com/docs/k6/latest/using-k6/scenarios/concepts/open-vs-closed/)
   explains the distinction. Report missed request starts as well as completed ones.

Tell the design skill your target request rate, data size, acceptable response
time, and available machines. If you do not have targets yet, ask for an initial
measurement plan; do not invent a pass threshold from the same run being judged.

### A real distributed setup

Use the application's supported cluster size. For a service that supports three
voting replicas, a starting layout is three server hosts plus a controller that
sends requests and collects results. A three-node majority needs two votes, so
this layout can exercise loss of one voter; other architectures need different
layouts. Separate VMs on one physical host still do not test physical-host loss.

Before running tests:

- Record each host's address, role, software version, and storage location.
  Match production storage behavior where it matters; do not give replicas the
  same data directory unless shared storage is part of the actual design.
- Verify client-to-server and server-to-server connections. If the design uses
  a leader, verify who it is and that replication works before causing failures.
- Arrange remote start/stop commands and log collection. Keep the controller's
  management connection available while interrupting application traffic, so
  the test can restore communication and collect evidence.
- Choose how to stop one process or interrupt one specific network link, and
  check the required permissions. Use the project's existing tools first.
  Restore each interruption and verify recovery before the next test.
- Keep clocks synchronized for comparing logs, but do not assume timestamps on
  different hosts establish an exact event order. Use the recording method
  required by the plan's checker.

Start with normal traffic, then one server restart, then a broken connection
while requests continue. Test both successful recovery and the correctness of
the data. Larger clusters, cross-region layouts, and storage failures should
follow the guarantees you actually need to test.

Give Codex a concrete environment description, for example:

```text
$designing-distributed-system-tests
Design a plan for this project's three-replica test cluster.
Available: Linux test hosts test-a, test-b, test-c, plus test-controller.
The controller has SSH access and can reach the application ports.
Use our existing setup and reset commands in docs/testing.md.
Cover normal operation, one server restart, and one interrupted network link.
Record how each failure is verified and how the environment is restored.
Save the plan to docs/testing-plans/three-replica-recovery.md.
```

Replace those example hosts and file path with your own. For performance work,
use the same pattern with your benchmark command, machine details, workload,
and performance targets. Keep commands, versions, results, and reset instructions
together so another developer can repeat the run.

## How it works

The workflow connects a product promise to an observable result:

1. **Find the promises.** Inspect documentation, code, configuration, and
   existing tests. Identify what the system promises, under which settings,
   and where the guarantee is ambiguous. Record actual builds and effective
   settings rather than assuming a mode name proves its behavior.
2. **Identify ways it could fail.** For example, a retry might repeat an
   operation the server already completed, or a restart might lose data that
   was reported as saved. Test normal operation first, then simultaneous
   requests and failures where relevant.
3. **Define the tests.** Specify which requests to send, how long to run,
   and exactly what makes a result correct. “The server stayed up” is not
   enough to establish that it kept every saved entry.
4. **Run and record.** Reuse the project's tools, create temporary test
   scripts where needed, and record requests and responses. When deliberately
   interrupting a connection or stopping a process, verify that it happened.
5. **Check and explain.** Evaluate the recorded results. When a test fails,
   try to find a smaller example that reproduces the problem. Investigate
   whether the bug is in the application, the test code, or the environment,
   and explain which planned tests remain incomplete.

For tests involving simultaneous requests, the order of events can matter as
much as the final data. The agent records what each client attempted, when it
happened, and what came back. A timeout is an unknown outcome: the server may
have completed the request even though the client never received its response.

### Terms you may see in the detailed guides

| Term | Plain-language meaning |
|---|---|
| SUT (system under test) | Your application, service, or database being tested |
| Claim | A promise the system makes, such as keeping data after a restart |
| Scenario | One test setup: the requests to send, conditions to create, and results to check |
| Checker or oracle | Code that decides whether the observed results are correct |
| Test harness | The scripts and supporting code that set up and run a test |
| Operation history | A record of requests, responses, and their timing |
| Fault injection; nemesis | Deliberately causing a failure; the tool that causes it |
| Budget | How much testing to do: run time, number of requests, repetitions, and configurations |
| Verdict | The recorded result: passed, failed, incomplete, or not run |

### Example: retrying a request must not add the same entry twice

Suppose your service adds entries to a log and never removes them. Each request
includes an **idempotency key**: an identifier reused when retrying that same
request, so the server can recognize it. The service promises:

- Retrying the same request with the same key never creates a second entry.
- Once the server reports success, the entry is still there after a restart.

A focused plan could include these scenarios:

| Scenario | What it tries to expose | Evidence and check |
|---|---|---|
| Send the same request from two clients at once | Both callers create an entry | Give both requests the same key and check that no more than one entry was created |
| Interrupt a response, then retry | The client repeats a request the server already completed | Record the interruption and retry; check for duplicates without assuming the first request failed |
| Restart after successful requests | Saved entries disappear | Record which requests succeeded and check that their entries remain after restart |

The real plan also specifies test files, request counts, run times, cleanup,
and the data needed to check the results. Surviving a power outage needs a
separate test: stopping a process can leave data in operating-system memory
that would disappear if the machine lost power.

A successful health check after restart tells you the service responds. It does not
establish either of the two guarantees above; the recorded operations and
state checks do that work.

## What gets tested

You can describe the behavior you care about without choosing a testing tool.
The agent selects an approach based on your code and existing tests. Advanced
tools such as Jepsen, a framework for testing systems under failures, are
optional; a small test using your existing tools may be enough.

| Concern | Approach and reference |
|---|---|
| Unusual inputs or sequences of operations produce wrong results | [Generate many inputs and check expected relationships](skills/designing-distributed-system-tests/references/property-and-metamorphic.md) |
| SQL queries or transactions return incorrect data | [Compare with a reference database and check each transaction's operations](skills/designing-distributed-system-tests/references/sql-semantic-testing.md) |
| Simultaneous requests lose updates or return inconsistent data | [Check request histories against the promised behavior](skills/designing-distributed-system-tests/references/jepsen-and-elle.md) |
| A bug appears only with particular timing or inputs | [Replay controlled event schedules](skills/designing-distributed-system-tests/references/deterministic-simulation.md) or [generate varied inputs](skills/designing-distributed-system-tests/references/fuzzing.md) |
| A connection breaks or one part of the system stops responding | [Deliberately interrupt connections and processes](skills/designing-distributed-system-tests/references/chaos-and-fault-injection.md) |
| A crash or upgrade loses or corrupts data | [Check recovery, saved data, and upgrades](skills/designing-distributed-system-tests/references/crash-recovery-and-upgrade.md) |
| One customer's data or heavy traffic affects another customer | [Check access boundaries](skills/designing-distributed-system-tests/references/boundary-and-isolation-testing.md) and [performance per customer](skills/designing-distributed-system-tests/references/performance-and-benchmarking.md) |
| A coordination protocol may have a design error | [Model the design and check its rules](skills/designing-distributed-system-tests/references/formal-methods-tla.md) |

The [catalog index](skills/designing-distributed-system-tests/references/catalog-index.md)
helps choose techniques. The [pitfall catalog](skills/designing-distributed-system-tests/references/common-distributed-systems-pitfalls.md)
provides examples of recurring bugs to consider when planning tests.

For example, SQL comparisons must preserve duplicate rows and data types;
otherwise the test could hide incorrect results. Storage tests must actually
reach the disk-writing and recovery code they claim to test, rather than
exercising only data held in memory.

## Outputs and how to read them

By default, a design-and-execute run produces:

```text
docs/testing-plans/<slug>.md
test-sessions/<slug>/<UTC_timestamp>/
├── session-log.md       # tools, versions, settings, timeline, amount of testing
├── logs/                # application and test-script logs
├── metrics/             # measurements collected during the run
├── artifacts/           # request records, temporary scripts, data, failure examples
└── findings/
    ├── <scenario>.md    # findings recorded as scenarios finish
    └── report.md        # results, evidence, and what remains untested
```

**Read the plan before running tests.** It explains what could go wrong, which
tests will look for those problems, and what tools they need. It defines three
levels of testing:

- **Smoke:** a small initial run to check basic behavior and test setup.
- **Hardening:** more thorough testing, using longer runs, more inputs,
  repeated runs, or additional settings as appropriate.
- **Release:** the testing proposed before shipping. If this has not been
  defined, the plan must say why and when to revisit it.

Each level has a budget specifying how much testing to do. This lets you review
the time and resources needed before asking the agent to run it.

**Read the report after running tests.** It links results to logs and recorded
data and explains what remains untested. The result labels mean:

| Verdict | Meaning |
|---|---|
| `PASS-smoke` | The small initial run completed all required checks without finding a problem |
| `PASS-hardening` | The more thorough run completed its required checks without finding a problem; any planned failures were verified to have happened |
| `FAIL-reproducible` / `FAIL-nondeterministic` | A problem was found; rerunning either reproduced it or did not |
| `INCONCLUSIVE-env` | A required tool or environment capability was unavailable |
| `INCONCLUSIVE-oracle-too-weak` | The checks, recorded data, or amount of testing were insufficient to decide |
| `INCONCLUSIVE-fault-not-proven` | The test could not verify that the intended interruption or failure happened |
| `PARTIAL-surface` / `PARTIAL-model` | Only basic checks ran, or only some of the required behavior was checked |
| `NOT-RUN` | A planned scenario was not attempted |

Thorough testing of normal operation can earn `PASS-hardening` without causing
failures. That does not establish how the system behaves during failures.
Likewise, one quick restart test does not count as a thorough run.
See the [verdict rules](skills/executing-distributed-system-tests/references/verdict-taxonomy.md)
for how labels are assigned when several conditions apply.

A passing run means no problem was found in the behavior tested. It is not
a proof of correctness or an automatic release approval. Review the report's
untested areas and evidence before deciding whether to ship. “The vendor says
it is fixed” is kept separate from “we reran the failing test and checked the fix.”

## What the agent may change

In **default execution mode**, the skill leaves project source files alone.
It can create temporary test scripts and result files under the session
directory and operate the test environment. Tests can restart processes,
interrupt traffic, or change test data, so use an environment you can reset.

In **author mode**, explicitly requested by you, it adds tests to your project
so you can rerun them to catch the same bugs later. It uses the file paths and
starter code specified in the plan, completes and runs the tests, then stages
them for review without committing. If a target file already exists, it asks
whether to overwrite it, skip it, or write a separate file.

The design skill writes a plan, not a full test implementation. A useful plan
may identify missing test tools or setup that need engineering work before
the tests can run.

## Repository guide

- [INSTALL.md](INSTALL.md): installation and updates.
- [USAGE.md](USAGE.md): prompts for each workflow and mode.
- [Design skill](skills/designing-distributed-system-tests/SKILL.md) and
  [plan template](skills/designing-distributed-system-tests/assets/plan-template.md):
  the planning workflow and its output structure.
- [Execution skill](skills/executing-distributed-system-tests/SKILL.md) and
  [findings template](skills/executing-distributed-system-tests/assets/findings-report-template.md):
  the execution workflow and its output structure.
- [Result-checking approaches](skills/executing-distributed-system-tests/references/oracle-patterns.md)
  and [request-recording guidance](skills/designing-distributed-system-tests/references/history-discipline.md):
  choosing a checker and recording the data it needs.
- [evals/](evals/README.md): manual regression prompts for changes to the skills.
- [specs/](specs/): historical design notes; the current skill files define the workflow.

## Status and limitations

These skills are evolving. Development has included local tests against
AgentDB and LMCache. Those machine-specific plans and results live under
`verification/`, which is gitignored and is not included in this repository.

The evaluation prompts are **manual cases, not an automated test suite**.
Some need other projects checked out locally; others are self-contained design or result
reviews. They test the agent's use of the skills, separately from the tests
that the skills help you run against your own system.

Results depend on what was tested and how the agent built and checked the tests.
The report must show supporting evidence and what was left untested so you
can judge how much to rely on it.

## Acknowledgements

The technique catalog draws on Andrey Satarin's
[testing-distributed-systems](https://github.com/asatarin/testing-distributed-systems)
catalog and research on Jepsen/Elle, fault injection, crash consistency,
deterministic simulation, and formal methods. Individual references link to
the original tools and papers.

## License

[MIT](LICENSE).
