# Jepsen and Elle

**Last verified:** 2026-05-16

## When to reach for it
Use Jepsen and Elle when testing distributed correctness: consistency models, isolation levels, linearizability under concurrent clients with injected network faults. Builds automated test harnesses that find anomalies no human would manually trigger.

## What it detects well
- Lost updates; dirty reads; stale reads; lost writes under partition.
- Lost acks and divided-brain scenarios after network partitions.
- Isolation anomalies (G0, G1a/b/c, G-single, G2-item, G2).
- Reconfiguration races and leader election failures.
- Concurrency bugs that require precise message ordering.

## What it misses
- Performance regressions and resource leaks (single-node memory leaks, thread exhaustion).
- UI bugs and client-side logic errors.
- Bugs that only manifest under workloads the test generators don't emit.
- Slow leaks of correctness over very long runs (Elle is bounded by ops history length).

## Concrete tools
- `Jepsen` — Clojure test framework with built-in nemesis and generators — https://github.com/jepsen-io/jepsen
- `Elle` — transactional anomaly checker over op histories, language-agnostic — https://github.com/jepsen-io/elle
- `Porcupine` — Go linearizability checker — https://github.com/anishathalye/porcupine
- `Maelstrom` — Jepsen toy-protocol workbench for any language — https://github.com/jepsen-io/maelstrom

## Papers to cite
- "Elle: Inferring Isolation Anomalies from Experimental Observations" (Kingsbury & Alvaro, VLDB'20) — black-box anomaly classifier you can run against any DB — https://github.com/jepsen-io/elle/raw/master/paper/elle.pdf
- "Jepsen Analyses" — case studies of real-world anomalies in production databases — https://jepsen.io/analyses

## Cost / wall-clock signal
Per-scenario: 30 minutes to multiple hours of wall-clock time; Elle history analysis itself sub-second to minutes depending on op count.

## How a plan typically uses it
1. Name the consistency model claimed (linearizable / serializable / SI / RC / causal+ etc.).
2. Pick generators that exercise the operations whose anomalies you care about.
3. Pick a nemesis schedule with realistic partition shapes, not just symmetric ones.
4. State which Elle anomaly classes count as failures vs. acceptable.
5. Record ops history for offline re-analysis.
6. Define minimum ops-per-run for statistical relevance (typically 10k+ ops per scenario).
