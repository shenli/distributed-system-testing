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
- "Mutate the raw backend, bypassing the SUT's wrapping layer." A
  common silent-no-op: deleting a row directly on the raw KV / disk /
  table when the SUT writes through a key-prefix, tenant prefix,
  shard/partition wrapper, encryption layer, or namespace
  indirection. The "fault" lands on bytes the SUT never reads.
  Mirror the SUT's write path when injecting at the storage layer —
  and verify by reading the key back through the *same* wrapper
  before declaring the fault landed.
