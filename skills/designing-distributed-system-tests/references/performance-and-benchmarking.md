# Performance and Benchmarking

**Last verified:** 2026-05-16

## When to reach for it
Use performance testing and benchmarking when you care about latency tail, throughput, fairness, or any "system slowed down" rather than "system gave wrong answer." Essential for workloads where p99 latency matters as much as correctness.

## What it detects well
- Coordinated-omission lies in load-generator measurements.
- Regressions under load, latency amplification, and GC-induced tail latency.
- Head-of-line blocking, queue buildup, and cascading slowdown under contention.
- Fairness violations across tenants or request classes.
- Resource exhaustion: file-descriptor limits, connection pooling, memory growth under load.

## What it misses
- Correctness bugs that do not change wall-clock timing.
- Bugs only visible at scales or load patterns you do not test.
- Subtle issues in rarely-hit code paths (fuzzing and simulation are better).

## Concrete tools
- `wrk2` — constant-throughput HTTP load generator, avoids coordinated omission — https://github.com/giltene/wrk2
- `k6` — modern, scriptable load testing platform — https://k6.io
- `fortio` — HTTP/gRPC load testing and latency analysis, from Istio — https://github.com/fortio/fortio
- `vegeta` — HTTP load testing and results analysis CLI — https://github.com/tsenart/vegeta
- `HdrHistogram` — Gil Tene's high-dynamic-range latency histogram library — http://hdrhistogram.org
- `YCSB` — Yahoo Cloud Serving Benchmark for key-value stores — https://github.com/brianfrankcooper/YCSB

## Papers to cite
- "How NOT to Measure Latency" (Gil Tene) — coordinated omission, percentile fallacies, and why averages lie — https://www.youtube.com/watch?v=lJ8ydIuPFeU
- "Your Load Generator Is Probably Lying To You" — overview of common load-test mistakes — https://highscalability.com/blog/2015/10/5/your-load-generator-is-probably-lying-to-you-take-the-red-pi.html
- "Performance Analysis Methodology" (Brendan Gregg) — USE/RED method and systematic perf analysis — https://www.brendangregg.com/methodology.html

## Cost / wall-clock signal
Minutes to days per scenario; real bottleneck is environment realism—cloud-hosted tests may not reproduce on-prem behavior.

## How a plan typically uses it
1. Measure latency as a full distribution (p50, p99, p99.9, max)—never report average alone.
2. Confirm the load generator itself avoids coordinated omission (use open-loop, not closed-loop).
3. Drive the open-loop arrival rate as constant; closed-loop load hides queue buildup.
4. Measure under realistic contention, not just single-tenant; include mixed workloads.
5. Capture system metrics alongside (CPU, GC, network, disk I/O) so you can attribute slowdowns to root causes.
