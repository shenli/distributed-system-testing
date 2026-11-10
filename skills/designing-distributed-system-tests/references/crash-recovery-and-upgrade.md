# Crash Recovery and Upgrade Testing

**Last verified:** 2026-05-16

## When to reach for it
Use crash-recovery and upgrade testing on anything touching durability, replay, idempotency, or version-to-version state migration. This category is the leading source of distributed-systems production incidents per Gunawi et al., "What Bugs Live in the Cloud" (SoCC'14).

## What it detects well
- Lost writes after crash and double-apply on replay.
- Duplicate side effects from retried operations.
- Mixed-version protocol bugs and incompatible state during rolling upgrades.
- Schema-migration corruption and downgrade-incompatible state.
- Fsync gaps and partial-checkpoint corruption.
- WAL truncation races and orphaned temp files.

## What it misses
- Steady-state correctness bugs (use Jepsen or Elle for that).
- Performance characteristics during recovery (use performance testing).
- Invariants requiring whole-history analysis across machines (use Elle).

## Concrete tools
- `ALICE` — application-level crash explorer for filesystem-level bugs — https://github.com/madthanu/alice
- `Torturing Databases framework` — power-loss simulator with block-level tracing — https://www.usenix.org/system/files/conference/osdi14/osdi14-paper-zheng_mai.pdf
- `CrashMonkey` — filesystem crash-consistency tester for ext4 — https://github.com/utsaslab/crashmonkey
- Rolling-upgrade harnesses — project-specific; most systems have one (Kafka, Cassandra, Postgres).
- WAL replay tooling — most durability systems already ship replay; surface it in tests.

## Papers to cite
- "Torturing Databases for Fun and Profit" (Zheng et al., OSDI'14) — power loss testing against real and open-source databases — https://www.usenix.org/system/files/conference/osdi14/osdi14-paper-zheng_mai.pdf
- "An Empirical Study on Crash Recovery Bugs in Large-Scale Distributed Systems" (Gao et al., FSE'18) — crash-recovery bug taxonomy and implications — https://dl.acm.org/doi/10.1145/3236024.3236030
- "Understanding and Detecting Software Upgrade Failures in Distributed Systems" (Zhang et al., SOSP'21) — upgrade failure taxonomy with detection tools — https://dl.acm.org/doi/10.1145/3477132.3483577

## Cost / wall-clock signal
Per-scenario: minutes to hours; planning cost is high because the fault matrix (crash at every IO boundary × every upgrade step) is large.

## How a plan typically uses it
1. Enumerate operation IO boundaries and inject a crash at each one.
2. After recovery, assert post-recovery state is equivalent to a known-good baseline via replay equivalence.
3. For idempotent operations, drive explicit retry storms to surface double-apply bugs.
4. For upgrades, test mixed-version state at every intermediate step, not only N → N+1 transitions.
5. Treat fsync as a contract—if the filesystem doesn't fsync, the SUT doesn't fsync for test purposes; make that assumption explicit.

## Separate recovery failure models

A process kill leaves the kernel page cache alive; it does not simulate
power loss. Choose distinct arms for process death, loss of unsynced data
(for example, a configured LazyFS-style filesystem), returned IO/fsync
errors, and silent corruption. Deliberately disabling fsync is a different
contract from a storage device returning an error. Declare which writes
each mode promises to preserve at acknowledgement.

For corruption testing, use disposable copies and record exact files,
offsets, original/mutated bytes or hashes, and restart boundaries. Treat
bit changes, truncation, and block substitution/reordering as separate
cases. A checksum warning alone is not success: verify the documented
recovery policy, including refusal to serve, explicit repair, or recovery
from an intact replica. Do not accept silently serving a partial history
or initializing an empty store as successful recovery.

Prove the storage path under test was reached: table/index files, flushes,
checkpoints, or compactions need their own evidence. Record any reduced
test thresholds. A run that wrote only a WAL cannot establish correctness
of recovery from table files; leave that coverage open.
