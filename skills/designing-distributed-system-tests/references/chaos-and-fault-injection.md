# Chaos Engineering and Fault Injection

**Last verified:** 2026-05-16

## When to reach for it
Use chaos and fault injection when you have a real cluster (or close enough) and want to know whether it survives the faults that actually happen in production. Bridges the gap between closed-box testing and real-world resilience.

## What it detects well
- Partial and asymmetric network partitions.
- Slow nodes (limping) and high-latency scenarios.
- Packet loss, latency injection, and bandwidth constraints.
- Disk-full, fsync failure, and storage anomalies.
- Process kill, container restart, and orchestrator churn.
- Clock skew and GC pause analogues.
- Limplock and partial-failure detection gaps.

## What it misses
- Deterministic reproducibility (re-running may not hit the same interleaving).
- Root cause without good observability in place.
- Anything requiring whole-history analysis or post-hoc log replay.

## Concrete tools
- `Toxiproxy` — TCP proxy with programmable network faults — https://github.com/Shopify/toxiproxy
- `Chaos Mesh` — Kubernetes-native chaos platform — https://chaos-mesh.org
- `LitmusChaos` — CNCF chaos engineering platform — https://litmuschaos.io
- `AWS Fault Injection Service` — managed FIS for AWS workloads — https://aws.amazon.com/fis/
- `tc` / `netem` — Linux traffic control for latency/loss/bandwidth — https://man7.org/linux/man-pages/man8/tc-netem.8.html
- `iptables` — kernel-level packet drops for partition emulation — https://netfilter.org/projects/iptables/
- `kill -STOP` / `kill -CONT` — process freeze, simulates GC pause well.

## Papers to cite
- "Toward a Generic Fault Tolerance Technique for Partial Network Partitioning" (Alfatafta et al., OSDI'20) — taxonomy and mitigation for partial partitions — https://www.usenix.org/conference/osdi20/presentation/alfatafta
- "Understanding, Detecting and Localizing Partial Failures in Large System Software" (Lou et al., NSDI'20) — intrinsic watchdogs catch degraded-but-alive failures — https://www.usenix.org/conference/nsdi20/presentation/lou
- "The Case for Limping-Hardware Tolerant Clouds" (Do et al.) — degraded hardware is worse than dead — https://www.usenix.org/node/174577

## Cost / wall-clock signal
Minutes to hours per scenario; total budget usually bounded by cluster availability, not compute.

## How a plan typically uses it
1. Catalog the realistic faults—power loss, network partition (full/partial/asymmetric), slow disk, slow node, clock skew.
2. For each, define the property the SUT should preserve.
3. Require an oracle that actually runs after the fault clears.
4. Demand evidence the fault was injected (log line, packet capture, proxy stat)—silent no-op is the most common false pass.
5. Include recovery time as an exit criterion, not just correctness.
