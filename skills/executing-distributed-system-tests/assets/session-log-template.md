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

## Environment capability matrix

The full capability probe from step 2b. One row per requirement the
plan's §6b declared (or that the selected techniques imply). Fill
before any scenario runs; the findings report cites it per scenario.

| Requirement | Present? | Version | Source |
|---|---|---|---|
| docker + compose | yes / no | {{version}} | {{how detected / how installed}} |
| iptables (CAP_NET_ADMIN) | yes / no | {{version}} | {{host / sudo available?}} |
| {{...}} | | | |

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

## Scenario verdicts

One row per scenario (and per §7.M.S arm). Record the budget tier the
run actually met alongside the verdict, so the findings report can
confirm each verdict is defensible (a `PASS-hardening` requires the
hardening tier to have been met, not just a clean oracle).

| Scenario / arm | Budget tier met | Verdict | Notes |
|---|---|---|---|
| S1 | smoke / hardening / release | {{one of the 10 states}} | |
| S5/api | hardening | PASS-hardening | |
| S5/export | — | NOT-RUN | export harness not built |

## Artifacts

- `logs/` — SUT logs per node
- `metrics/` — scraped metrics, baseline + during run
- `artifacts/` — anything else: heap dumps, op histories, packet
  captures, screenshots
- `findings/` — per-scenario verdict files and the final report
