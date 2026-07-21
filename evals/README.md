# Evals

Manual regression prompts for the two skills — **not an automated
suite.** Each `evals.json` is a small set of `{prompt, expected_output}`
pairs used to sanity-check that a change to a SKILL.md body did not
regress its behaviour.

- `designing/evals.json` — prompts for `designing-distributed-system-tests`
- `executing/evals.json` — prompts for `executing-distributed-system-tests`

## How to run

There is no harness. Paste a prompt into an agent that has the skills
installed and compare the result against `expected_output` by reading.

## Portability caveat

Several prompts reference the author's local machine — absolute paths
like `/Users/lishen/work/agentdb-dst-verify`, `/tmp/web-queue-plan.md`,
and SUT files such as `tools/agentdb-cluster-smoke` or
`docs/runbooks/fault_injected.md`. Those checkouts are not part of this
repo. To reuse a prompt elsewhere, substitute your own SUT path and
tooling. Prompt 3 in each file (early-exit / missing-oracle) is
self-contained and runs anywhere.
