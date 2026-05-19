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
| Minority-side partition | iptables drop minority↔majority in both dirs | RPC timeouts on minority; majority continues | reverse iptables |
| Majority-side partition | invert above: drop traffic into majority from minority + outside | majority quorum fails to make progress | reverse iptables |
| Leader isolation | partition just the current leader (identified live from cluster status); both other-sides keep talking | leader sees its peers go silent; quorum re-elects | reverse the rule when re-election completes |
| Packet duplication | `tc qdisc add … netem duplicate 10%` | duplicate-packet counter, double-RPC counters | `tc qdisc del` |
| Packet reordering | `tc qdisc add … netem reorder 25% delay 50ms` | out-of-order arrival counter | `tc qdisc del` |
| Metadata-store outage | drop traffic to etcd / ZK / DNS by IP+port | dependent ops fail with metadata-unreachable errors | reverse |

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
| fsync failure | dm-flakey configured to drop fsync; or eatmydata; or syscall interception | crash-recovery uncovers lost writes | remove dm target / disable interception |
| Storage corruption (typed) | dm-flakey returning EIO; or dm-error; or bitrot via dd if=/dev/urandom into the device | SUT surfaces checksum / integrity errors | remove dm target |
| Backup/restore race | trigger backup mid-workload, then restore mid-workload, then continue | restore-time markers in log; oracle compares pre/post replicas | wait for restore to complete |

## Time faults

| Fault | Mechanism | Evidence | Cleanup |
|---|---|---|---|
| Clock skew | `date -s` inside container, or libfaketime | timestamp drift in logs | reset clock / drop libfaketime |
| Slow clock | libfaketime with rate | derived metrics drift | drop |

## Cluster-level faults

| Fault | Mechanism | Evidence | Cleanup |
|---|---|---|---|
| Rolling restart, wrong order | restart nodes in an order that crosses leadership transitions | leadership changes during restarts; counts in raft / consensus metrics | finish the restart cycle |
| Split-brain attempt | combine partition with leadership timeout pressure | competing-leader log lines on both sides | end the partition |
| Slow follower | latency injection (`tc netem`) on one node only | RPC latency histogram skew per node; back-pressure metrics on others | `tc qdisc del` |
| Mixed-version cluster | bring half the nodes up on version N, half on N+1 | version-mismatch log lines; cluster membership view | finish the upgrade or roll back |
| Rolling upgrade | upgrade nodes one at a time; workload keeps running | per-node version flip in cluster status; workload error rate | finish the upgrade |
| Config divergence | one node has a different config value the SUT does not gossip | reachability of config via SUT introspection; divergent behavior per node | reset the config |
| Credential / auth divergence | rotate auth secret on N-1 nodes, leave one stale | auth-failure counter on the stale node | rotate the last node |
| Compaction / GC during reads | trigger compaction (storage-engine specific) while a high-read workload is mid-flight | compaction-active log line; read latency histogram skew | wait for compaction to complete |
| Rebalancing / reconfiguration during writes | trigger a shard rebalance or membership change during a sustained write workload | rebalance-in-progress metric; per-shard ownership transitions | wait for rebalance to complete |

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
