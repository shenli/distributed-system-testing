# Boundary and Isolation Testing

A claim about a boundary — "tenants are isolated from each other,"
"every shard's traffic stays on its own routing path," "no group
starves another for throughput" — is not a single property of the
system; it is a property held *across every surface where the
boundary could leak*. A test against the API surface alone proves
the API surface is fine. It does not prove the boundary.

This reference is the discipline a plan must follow when a scenario
falsifies a claim in `{boundary, fairness}` (see plan template §1b
legend). For those scenarios, the plan template's §7.M.S block is
mandatory.

## When this discipline is mandatory

When the scenario's `Falsifies if it FAILs` row touches any claim
in:

- `boundary` — tenant isolation, authz, namespace, routing,
  multi-protocol access, compatibility across API surfaces.
- `fairness` — per-group performance, noisy-neighbor isolation,
  queue / partition / region / priority-class fairness.

For consistency-isolation claims (the existing `isolation`
category, meaning G2-item / serializability / Elle-detectable
anomalies) this file is *not* the right reference — use
`oracle-patterns.md` §1 (linearizability) or §3 (serializability)
instead. The legend uses two distinct labels (`isolation` vs
`boundary`) for two distinct concepts.

## Surface catalogs by system kind

A starting set of surface enumerations per archetypal SUT. The
catalog is not exhaustive; the plan author should add SUT-specific
surfaces the catalog misses.

### Database

- SQL / query API
- Backup and restore
- Change-data-capture (CDC) stream
- Admin API (DDL, role management, configuration)
- Driver libraries (per language, with their own connection state)
- Bulk import / export tools

### Queue / message broker

- Produce
- Consume (per consumer group)
- Offset management
- Consumer group rebalancing
- Dead-letter queue (DLQ)
- Replay / time-travel reads

### Object store

- Put / Get / List / Delete
- Multipart upload
- Lifecycle (expiration, transition between storage classes)
- Cross-region replication
- Bucket-level admin operations

### Control plane

- REST or RPC API
- CLI
- Scheduler
- Reconciler / controller loop
- Metadata store (etcd / ZK / equivalent)

### Multi-tenant service

- Public API
- Per-language SDKs
- Background jobs (cron, queued workers)
- Exports / reporting
- Billing and metering
- Metrics / observability stack
- Audit logs

### Multi-region service

- Per-region read / write
- Replication lag
- Region failover
- Data residency boundary

## Boundary claim matrix template

For every boundary-style claim, the plan fills this 13-field matrix
inside the scenario's §7.M.S block. Copy the template; fill each
field; leave none blank.

```
Boundary claim:           <one-sentence statement of the claim>
Boundary keys:            <what scopes the boundary — tenant id,
                          namespace, shard, region, etc.>
Surfaces:                 <list, drawn from the catalog above or
                          SUT-specific>
Operations:               <per surface, which ops the scenario
                          exercises>
Positive controls:        <what legitimate access should still
                          succeed>
Negative controls:        <what illegitimate access should be
                          denied AND not observable in metrics /
                          logs / side channels>
Confusable identifiers:   <crafted inputs designed to provoke leakage>
Delayed / async paths:    <background jobs, retries, GC, compaction,
                          CDC streams, exports>
Background paths:         <jobs that run without per-request
                          authentication context>
Observability paths:      <metrics / traces / audit logs that could
                          themselves leak across the boundary>
Oracle:                   <the per-arm property checked; must
                          include both positive and negative
                          controls>
Minimum smoke budget:     <configuration size, run duration, fault
                          count, seed count required for PASS-smoke>
Minimum hardening budget: <strictly stronger than smoke on every
                          dimension; required for PASS-hardening>
```

The matrix is the input to the §7.M.S block in the plan template.
The plan author lifts the rows into the template's named fields.

## The split-into-arms rule

If a single scenario, after surface decomposition, would span more
than **three surfaces**, more than **three claim categories**, or
require more than **one independent oracle**, split it into
scenario arms.

Each arm gets its own ID (e.g., `S5/api`, `S5/sdk`, `S5/admin`),
its own §7.M block (model / history / checker), its own oracle, and
its own verdict at run end.

The arms' verdicts aggregate to a scenario-level verdict via the
downgrade rule (see `verdict-taxonomy.md`): any `NOT-RUN` or
`PARTIAL-*` arm caps the scenario-level verdict at
`PARTIAL-surface`, regardless of which arms passed.

## Confusable-identifier catalog

Inputs crafted to provoke boundary leakage. Cover at least one from
each row for every boundary claim:

| Class | Example pair |
|---|---|
| Identical name under different scopes  | tenant `acme` user `bob` vs tenant `globex` user `bob` |
| Prefix collision                       | `acme/` vs `acme-corp/` |
| Case-folding ambiguity                 | `Acme` vs `acme` vs `ACME` |
| Embedded separators                    | `a/b/c` vs `a%2Fb%2Fc` vs `a:b:c` |
| Trailing whitespace / control chars    | `acme` vs `acme ` vs `acme​` |
| Recycled ID after deletion             | tenant `acme` ID `1234` deleted, new tenant assigned `1234` |
| Near-quorum / off-by-one range         | shard ranges `[0, 100)` and `[100, 200)`; key exactly `100` |
| Integer overflow at boundary           | tenant ID `2^31 - 1` vs `2^31` vs `-1` |
| Unicode normalisation collision        | `Ã©` (U+00E9) vs `é` (e + combining acute) |

## Negative-control anti-pattern catalog

Common mistakes when designing the negative-control side of a
boundary oracle:

- **Positive control only.** Tests that the legitimate user can
  read but never tests that the illegitimate user cannot. Asserts
  half the boundary.
- **Denial without observability check.** The illegitimate request
  returned 403, so the test passes — but the metric counter still
  incremented, the audit log still recorded the resource id, or the
  cache fingerprint changed. The denial leaked metadata.
- **No-leak conflated with no-error.** "No errors during the
  cross-tenant run" is not the same as "no cross-tenant data
  observed." Check the data, not the error log.
- **Async path skipped.** Real-time API denies the cross-tenant
  request, but the nightly export job runs without auth context and
  emits the data anyway. Cover the async paths in the same scenario.
- **Observability path skipped.** Metrics are scoped per tenant but
  emit a global counter that exposes the existence of other
  tenants' activity. Treat the observability surface as a real
  surface.

## When this discipline is overkill

A scenario falsifying a single per-key safety claim with no
boundary semantic does not need the matrix. Use this reference only
when the claim is in `{boundary, fairness}`. Forcing surface
decomposition on a claim that is genuinely single-surface produces
ceremonial scenarios that say nothing extra.

## In the plan

The plan template's §7.M.S block has eight named fields drawn from
this matrix (Surfaces, Operations, Adversarial inputs, Positive
controls, Negative controls, Delayed / async paths, Observability
paths, Scenario arms). The matrix's `Confusable identifiers` row
maps to the template's `Adversarial inputs` field — the names
differ but the content is the same. The matrix's `Background paths`
row folds into the template's `Delayed / async paths` field — list
background-job paths there explicitly. The matrix's `Boundary
claim`, `Boundary keys`, `Oracle`, and budget rows live as
scenario-level fields elsewhere in the plan template (Falsifies
row, §7 Oracle field, §7 budget tiers). The plan author fills the
matrix here, then lifts the rows into the corresponding template
fields. The findings report's Surface coverage table mirrors the
matrix per arm.
