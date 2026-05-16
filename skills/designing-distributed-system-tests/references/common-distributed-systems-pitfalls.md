# Common Distributed-Systems Pitfalls

**Last verified:** 2026-05-18

A catalog of failure modes that show up over and over across the
Jepsen analyses corpus (jepsen.io/analyses) plus adjacent
literature. Use this during step 3 (hypothesis generation) of the
design skill: for each in-scope subsystem, walk this list and ask
"could the SUT exhibit this pitfall?" If yes, that's a hypothesis
worth a scenario.

Each entry includes a hypothesis template you can paste into the
plan's §4 and adapt to the SUT's vocabulary.

## How to use

1. After writing the SUT model (§2), open this file.
2. For each pitfall, decide: applicable to this SUT (`y/n/maybe`)?
3. For every `y` and `maybe`, write a hypothesis row in §4 that
   names the claim it could falsify (link back to §1b).
4. The "Test technique" link tells you which catalog file to read
   for the right tool — Jepsen+Elle for consistency-shaped pitfalls,
   crash-recovery+upgrade for durability-shaped, and so on.

The list below is ordered by frequency of appearance across the
Jepsen corpus; pitfalls 1–6 are present in the majority of
analyses, 7–14 are common, 15+ are subsystem-specific but worth
checking.

---

## 1. Lost updates under concurrent write + partition

**Symptom.** Two clients update the same key; both writes return
success; only one is preserved. Or: a write returns success but
is rolled back when the partition heals.

**Mechanism.** The system accepts writes on both sides of a
partition without quorum, then has no merge policy when the
partition heals (or uses last-write-wins with skewed clocks).

**Where seen.** MongoDB (2013), Redis (2013), NuoDB (2013),
RethinkDB (2016), Aerospike (2015, 2018).

**Test technique.** `jepsen-and-elle.md` — register / counter
operations under nemesis partitions, oracle = Elle.

**Hypothesis template.**
```
H<N>. **Lost update under partition + recovery** — when a partition
splits clients across two replicas and both accept writes to the
same key, the heal-side reconciliation could drop one of them.
Could falsify: C<x> (write durability).
Suspected because: <SUT> claims <consistency level> but the merge
policy for concurrent writes on disjoint partition sides is
<documented? inferred? unknown?>.
```

---

## 2. Stale reads from followers / secondaries

**Symptom.** A client writes, gets ack, then reads from a
follower and sees the old value. Often dressed up as a
"performance feature" but violates monotonic-read or read-your-
write claims.

**Mechanism.** Async replication where follower lag is unbounded;
the client routing layer doesn't track which replica saw the
write.

**Where seen.** Elasticsearch (2014, 2015), etcd (2014),
RabbitMQ (2014), MongoDB (2015), Cassandra (2013), Kafka (2013).

**Test technique.** `jepsen-and-elle.md` with read-write
generators + monotonic-read oracle. For high-throughput cases
also `property-and-metamorphic.md` (write-then-immediate-read
metamorphic relation).

**Hypothesis template.**
```
H<N>. **Stale read from follower after recent write** — a read
routed to a lagging follower returns a value older than the
client's most recent successful write.
Could falsify: C<x> (read-your-write), C<y> (monotonic reads).
Suspected because: replication is <sync? async?>; the read-routing
layer <does? does not?> track applied-index per replica.
```

---

## 3. Replica divergence after partition / restart

**Symptom.** Replicas converge to *different* states for the
same key, with no reconciliation path. Long-lived inconsistency
that survives the original fault.

**Mechanism.** Partition or crash during the apply phase leaves
divergent log tails. Anti-entropy doesn't trigger or has a bug;
the system has no canonical "winner."

**Where seen.** Aerospike (2015, 2018), Crate (2016), RethinkDB
(2016), VoltDB (2016).

**Test technique.** `jepsen-and-elle.md` + nemesis with partition
+ kill scenarios. After heal, compare every replica's state for
every key; oracle = bitwise-equal (or schema-aware equivalent).

**Hypothesis template.**
```
H<N>. **Replica divergence post-fault** — after partition or crash
+ heal, two replicas durably disagree about the value of key K
with no anti-entropy convergence.
Could falsify: C<x> (eventual convergence), C<y> (replicated
durability).
Suspected because: <SUT>'s anti-entropy mechanism is <X>; the
divergence-detection path runs <every N seconds? on demand?>.
```

---

## 4. Linearizability violation under wall-clock skew or timing

**Symptom.** A history of reads + writes that no sequential
order can explain — i.e. operation A appears after B from one
client and before B from another, in a way that no linearizable
schedule allows.

**Mechanism.** Wall-clock used to order events across nodes
without bounded skew. Or: lease expiry races; epoch numbers
roll back.

**Where seen.** CockroachDB (2017), MariaDB Galera Cluster
(2015, 2026), Percona XtraDB Cluster (2015), Hazelcast (2017).

**Test technique.** `jepsen-and-elle.md` with the linearizability
checker (Knossos / Porcupine), or Elle on a register workload.

**Hypothesis template.**
```
H<N>. **Linearizability violation under clock skew** — when wall-
clock drift between nodes exceeds <SUT>'s safety margin, observed
op histories admit no sequential ordering consistent with real-
time precedence.
Could falsify: C<x> (linearizability per <unit>).
Suspected because: <SUT> uses wall-clock for <ordering / leases /
hybrid logical clocks?> with assumed bound <N ms>; the bound is
<verified by NTP? assumed?>.
```

---

## 5. Aborts of "committed" transactions (lost acks)

**Symptom.** The client sees a SUCCESS response for a write or
commit, then later learns the write didn't actually persist (or
was rolled back). Worse: the client sees ERROR but the write
*did* persist.

**Mechanism.** The commit-confirmation message can be lost
between server and client (partition, crash). Without typed
"unknown" errors, the client must guess.

**Where seen.** Across most Jepsen analyses — particularly bad
in MongoDB (2017), RethinkDB (2016), VoltDB (2016). AgentDB had
this as F4 / F_seq_visibility (closed by PR #274 introducing a
typed retryable convergence error).

**Test technique.** `jepsen-and-elle.md` with nemesis kill +
partition during commit. Track client-observed verdict vs.
server-side authoritative log.

**Hypothesis template.**
```
H<N>. **Lost ack vs. lost commit** — when the server commits a
write but the client's connection drops before the ack arrives,
the client cannot distinguish "uncommitted, retry" from "committed,
do not retry" without a typed retryable error class.
Could falsify: C<x> (idempotency under retry), C<y> (durability).
Suspected because: <SUT>'s error responses use generic 5xx with no
typed retry hint; client retry policy assumes "5xx → not committed".
```

---

## 6. Reconfiguration / membership-change races

**Symptom.** Adding or removing a node during traffic loses
committed writes, causes elections to oscillate, or leaves the
cluster in a state with no leader.

**Mechanism.** Joint-consensus implementations have edge cases
when membership changes overlap with leader elections, log
catch-up, or other membership changes.

**Where seen.** RethinkDB (2016), Tendermint (2017), Redis-Raft
(2020). AgentDB had this as F_replacement_wedge (closed by PR #274
realigning rebalance ownership).

**Test technique.** `chaos-and-fault-injection.md` — drive
membership changes concurrent with writes + nemesis faults.
`crash-recovery-and-upgrade.md` for the recovery-state portion.

**Hypothesis template.**
```
H<N>. **Membership change is not idempotent under retry** — when a
re-issued add/remove RPC races the in-flight conf-change, the
system rejects the retry with "pending change" and the original
change never completes; cluster wedges.
Could falsify: C<x> (membership changes are safe under arbitrary
client retry).
Suspected because: <SUT>'s replacement RPC is <documented? inferred?>
to be idempotent; the retry-after-timeout policy assumes the first
call did not take effect.
```

---

## 7. Crash-recovery divergence

**Symptom.** A node crashes, restarts, and its post-recovery state
differs from the rest of the cluster. Often involves
partially-fsynced batches or incompletely-applied writes.

**Mechanism.** The fsync contract between application and OS/disk
is misunderstood; or recovery replays from a checkpoint that has
its own corruption.

**Where seen.** Across many Jepsen analyses; central to the
"Torturing Databases" line of research. AgentDB had this as F1
(register_skill / bind_skill bypass persist on cache miss; closed
by PR #273).

**Test technique.** `crash-recovery-and-upgrade.md` — power-loss
simulators, fsync-loss injection, in-process replay equivalence
checks.

**Hypothesis template.**
```
H<N>. **Crash recovery skips persisted operations** — a node that
crashes between operation persistence and in-memory cache update,
on restart, fails to consult persist for state the cache didn't
carry, and treats the operation as un-done.
Could falsify: C<x> (durability of acknowledged writes), C<y>
(idempotency across process restart).
Suspected because: <SUT>'s rehydration path <hydrates fully? lazily?
on demand?>; the cache-miss-then-persist-fallback contract is
<consistent across all call sites? per-site discretionary?>.
```

---

## 8. Schema migration during traffic

**Symptom.** Adding a column, changing a type, or dropping a
constraint while writes are happening loses data, corrupts
indexes, or breaks live reads.

**Mechanism.** Migrations are not coordinated with the write path;
or coordinated only via best-effort locks that don't extend across
nodes.

**Where seen.** Most relational Jepsen analyses (MySQL 2023,
PostgreSQL 2020/2025 — explicitly verified safe modulo concurrent
DDL); long-standing issues in NoSQL systems offering schema (Mongo,
Cassandra, etc.).

**Test technique.** `crash-recovery-and-upgrade.md` — mix writes,
reads, and DDL concurrently. Oracle: post-migration scan must
recover every write that returned success.

**Hypothesis template.**
```
H<N>. **Online migration loses concurrent writes** — DDL applied
while writes are flowing drops or mangles rows committed between
the migration's logical start and end.
Could falsify: C<x> (online schema change is non-disruptive).
Suspected because: <SUT>'s migration mechanism <takes a global lock?
runs per-shard? uses double-write?>; the verification step after
migration is <byte-for-byte? approximate?>.
```

---

## 9. ID / sequence number collision under partition

**Symptom.** Two clients on disjoint partition sides each get the
"next ID" and the IDs collide. Or: post-recovery, a previously-
generated ID is regenerated.

**Mechanism.** Sequence allocator depends on a centralized
counter that's not partition-tolerant, or on wall-clock without
bounded skew.

**Where seen.** MongoDB (2013), VoltDB (2016), AgentDB had a
related family in PR #272 ("Prevent append seq reuse after stale
runtime state").

**Test technique.** `property-and-metamorphic.md` for the
uniqueness property; `jepsen-and-elle.md` for the partition arm.

**Hypothesis template.**
```
H<N>. **Sequence number reused after partition + restart** — a
restarted owner mints a seq value that a prior incarnation
already minted (and possibly served), violating the gap-free /
unique-per-session contract.
Could falsify: C<x> (per-session gap-free monotone seq).
Suspected because: <SUT>'s seq allocator persists <every write?
periodically?>; the restart path <re-reads max persisted seq?
assumes in-memory state is canonical?>.
```

---

## 10. Watch / change-feed event loss or duplication

**Symptom.** A change-feed consumer either misses events that
happened (loss) or sees the same event twice without an idempotency
token (duplication). Especially bad under leader handoff.

**Mechanism.** Watch cursor management is per-replica; failover
loses or duplicates the cursor position.

**Where seen.** etcd (2014), Zookeeper (2013), MongoDB
(change-streams in various years), Kafka (consumer-group edge
cases).

**Test technique.** `jepsen-and-elle.md` with a watch oracle;
post-run: every committed write must appear exactly once in the
consumer's observed sequence.

**Hypothesis template.**
```
H<N>. **Watch drops or duplicates events across leader handoff** —
when the watch source fails over to a new replica, the consumer's
resume cursor either lands behind (re-deliver) or ahead (skip) of
the actual stream position.
Could falsify: C<x> (exactly-once change-feed delivery), C<y>
(cursor durability across handoff).
Suspected because: <SUT> exposes a cursor token; the token's scope
is <per-replica? cluster-wide?> and the handoff <translates? does
not translate?> tokens.
```

---

## 11. Cross-shard / cross-partition transaction non-atomicity

**Symptom.** A multi-key transaction commits some keys and not
others; reads see a partial transaction.

**Mechanism.** Distributed commit protocol (2PC, percolator) has
edge cases under coordinator failure, prepare-then-die scenarios.

**Where seen.** TiDB (2019), CockroachDB (2017), FaunaDB (2019),
YugaByte DB (2019), Dgraph (2018, 2020).

**Test technique.** `jepsen-and-elle.md` with Elle's anomaly
detection on multi-key transactions; oracle = isolation level
claimed.

**Hypothesis template.**
```
H<N>. **Multi-key txn partially commits under coordinator failure** —
if the transaction coordinator dies between prepare and commit-
all, a participant replica may commit while another does not,
visible to a subsequent read.
Could falsify: C<x> (atomicity across shards), C<y> (claimed
isolation level).
Suspected because: <SUT>'s 2PC recovery on coordinator death
<depends on a recovery leader scanning prepared participants?>;
the recovery path <runs on every restart? on a timer?>.
```

---

## 12. Clock-skew-dependent safety violations (TrueTime-ish)

**Symptom.** Operations that should be ordered by real-time
precedence aren't, when one node's clock jumps forward / back.

**Mechanism.** "Clock skew is bounded by X" assumption built into
safety, without bound enforcement at runtime.

**Where seen.** CockroachDB (2017) — most famous example;
foundational concern across many systems (Spanner avoids by
exposing TrueTime uncertainty).

**Test technique.** `chaos-and-fault-injection.md` — libfaketime
or clock-adjustment nemesis. Oracle: linearizability check that
should survive bounded skew.

**Hypothesis template.**
```
H<N>. **Safety breaks when wall-clock skew exceeds the assumed bound** —
under nemesis-injected clock skew of more than <X ms>, observed
op histories violate the claimed ordering guarantee even though
no message was lost.
Could falsify: C<x> (the safety bound on clock skew).
Suspected because: <SUT> assumes skew <≤ X ms> via NTP or similar;
the runtime <does? does not?> verify skew before relying on it.
```

---

## 13. Auth / quota / metadata state divergence under partition

**Symptom.** Authentication or quota state becomes inconsistent
across nodes after a partition. A user is logged in on one node,
denied on another. A quota is exceeded but the system doesn't
notice.

**Mechanism.** Auth/quota state replicated via a separate
mechanism from data state, with weaker consistency.

**Where seen.** Less common in Jepsen-narrow scope but central to
many real production incidents (Cloudflare, AWS IAM, etc.).

**Test technique.** `property-and-metamorphic.md` — assert auth
state convergence post-partition; `jepsen-and-elle.md` for the
quota-counter version.

**Hypothesis template.**
```
H<N>. **Auth/quota state diverges across replicas under partition** —
when partition isolates the auth-state coordinator, subsequent
auth decisions on each side diverge and post-heal reconciliation
keeps the divergence rather than reconciling.
Could falsify: C<x> (tenant isolation), C<y> (global quota
enforcement).
Suspected because: <SUT>'s auth state lives in <where>; replication
of that state is <sync? async? eventually consistent?>.
```

---

## 14. Lease expiry under contention

**Symptom.** A lease holder's renewal call is slow (because of
unrelated meta-layer contention); the lease expires before
renewal returns; ownership is forfeited even though the holder is
alive.

**Mechanism.** Lease renewal critical path shares a resource
(connection pool, lock, single thread) with other workloads. A
spike in the other workload starves the renewal.

**Where seen.** AgentDB's F_LEUSS (closed structurally by
PR #275 bounding renewal queries with per-query timeout). Common
in Raft / Paxos implementations with shared infrastructure.

**Test technique.** `chaos-and-fault-injection.md` (introduce
load on the contended resource); `performance-and-benchmarking.md`
(latency oracle on the renewal path).

**Hypothesis template.**
```
H<N>. **Lease expires under unrelated workload contention** — when
the meta-layer queue depth exceeds <X>, lease renewals on the
shared path stall longer than the lease window; the reaper clears
ownership of a live node.
Could falsify: C<x> (ownership stability under load), C<y> (lease
budget is sufficient under worst-case meta latency).
Suspected because: <SUT>'s renewal path uses <shared pool? dedicated
connection?>; the lease budget is <N ms> vs. worst-observed query
latency of <M ms>.
```

---

## 15. Idempotency replay bypassed by cold cache

**Symptom.** A retry with the same idempotency key, after the
in-memory dedup cache is cold (post-restart, post-handoff, post-
eviction), commits a SECOND effect instead of replaying the first.

**Mechanism.** Idempotency check goes "cache lookup → if miss,
proceed" without persist fallback.

**Where seen.** AgentDB's F1 — closed by PR #273 (the
`lookup_idempotency_record` helper that falls back to persist
when the cache misses).

**Test technique.** `crash-recovery-and-upgrade.md` — process
restart between original and retry; oracle = `IdempotencyConflict`
or replay of original result.

**Hypothesis template.**
```
H<N>. **Idempotency replay bypasses persist on cold cache** — after
restart / failover / eviction, a retry with the same idempotency
key but different payload commits a duplicate effect instead of
returning IdempotencyConflict.
Could falsify: C<x> (idempotency across process restart).
Suspected because: <SUT>'s idempotency check is <documented? inferred?>
to consult persist on cache miss; the implementation hits this
contract <consistently? per call-site?>.
```

---

## 16. Async outbox queue head-of-line-block on missing referent

**Symptom.** A background queue (outbox, index-apply, replication
log) stalls on the first entry it can't process, blocking every
later entry behind it.

**Mechanism.** Defensive `return Err(applied)` on unreparable
entries instead of skip-and-continue. Common when GC removes data
out from under an in-flight queue.

**Where seen.** AgentDB's F2 — closed by PR #273 with
skip-advance-continue + a new `record_index_outbox_skipped_total`
counter.

**Test technique.** `property-and-metamorphic.md` for the
"no head-of-line block" property; `crash-recovery-and-upgrade.md`
for the GC-purged-out-from-under variant.

**Hypothesis template.**
```
H<N>. **Outbox stalls on one unreparable entry** — when entry N
references state that GC has removed, the outbox apply loop
returns without advancing past N, blocking entries N+1, N+2, …
indefinitely with no operator-visible counter.
Could falsify: C<x> (eventual application of all enqueued
mutations), C<y> (observability of stall via a counter).
Suspected because: <SUT>'s outbox loop has an early-return path
for missing-referent that doesn't increment a counter or advance
the cursor.
```

---

## How this list evolves

Add a row when a new failure family appears across 3+ analyses or
gets a public post-mortem. Drop the per-system "where seen" list
if it grows past 5 entries — link to the analyses index instead.

When using this catalog, the hypotheses you generate should look
like the templates above but with concrete <SUT>-specific verbs,
counts, and component names. A hypothesis full of placeholders is
a hypothesis no one will test.
