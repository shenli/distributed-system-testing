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
