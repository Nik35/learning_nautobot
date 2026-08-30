# GTM Automation — Resilience Hardening Work Packages

**Status:** Ready for implementation
**Audience:** Claude Code (Opus 4.6) working in the GTM automation repository
**Companion document:** `docs/gtm-automation-implementation-plan.md`

---

## How to use this document

This document is deliberately self-contained. It restates the system context, the
architectural reasoning, and the relevant behaviour of Redis, Celery and MSSQL inline,
rather than pointing at external references. Do not search the internet for background
on the concepts described here — everything needed to implement these work packages is
written below.

There are five work packages:

| ID | Title | Type | Blocking? |
|----|-------|------|-----------|
| **WP-R1** | Verify redelivery safety of the task claim | Verification, then code only if needed | Do first |
| **WP-R2** | In-process settings cache with database fallback | Code | After R1 |
| **WP-R3** | Explicit fail-closed behaviour when Redis is unavailable | Code | After R2 |
| **WP-R4** | Celery beat high availability via database lock | Code | Independent |
| **WP-R5** | Test suite and failure drill runbook | Tests + runbook | After R1–R4 |

Work them in that order. WP-R1 is first because its outcome determines whether WP-R2
and WP-R3 need extra guards or not. WP-R4 is independent and can be done in parallel
by a separate agent if you are using subagents.

---

## 1. System context

Read this section fully before writing code. It is the shared vocabulary for everything
that follows.

### 1.1 What the system does

A Python service that provisions global server load balancing (GSLB) configuration in
response to API calls from an OpenShift-based consumer.

For a provisioning request, the service:

1. Runs **pre-validation** — extensive read-only checks against F5 BIG-IP DNS
   (WideIP, pool, pool members, monitor) and against Infoblox (CNAME record).
2. Runs **implementation** — creates, updates or deletes those objects.
3. Runs **post-validation** — reads back to confirm the intended end state.
4. On failure at any point, runs **rollback** (compensation) to return the estate to
   its prior state.

Rollback is comprehensive and already implemented. Specific known behaviours:

- On **create**, if the Infoblox CNAME creation fails, all F5 objects created during
  that request are rolled back.
- On **delete**, if the Infoblox CNAME deletion fails, the F5 objects that were deleted
  are recreated, and the CNAME's existence is re-verified.
- The same compensation approach applies to update (PUT) and delete (DELETE) paths.

The service replaced an earlier Ansible implementation that could not handle the
required concurrency.

### 1.2 Technology stack

- **API layer:** FastAPI
- **Async execution:** Celery workers, co-located with the API in the same pods
- **Scheduling:** Celery beat (currently a single instance — see WP-R4)
- **Coordination and rate limiting:** Redis (self-hosted, single instance — see
  WP-R2 and WP-R3)
- **System of record:** MSSQL, a managed and resilient service with Kerberos auth,
  shared across all pods
- **Platform:** OpenShift, private cloud

**MSSQL is the only managed service.** Redis, Celery, beat and the application itself
are all self-operated by the team. There is no managed Redis, no managed queue, and no
managed workflow engine available in this environment. Any design that assumes a
managed dependency is not implementable here.

### 1.3 Deployment topology

- 2 datacentres
- 4 pods total — 2 in each datacentre
- Each pod runs both FastAPI and Celery workers
- Round-robin load balancing across pods
- **Redis** runs as its own separate pod in one datacentre, with persistent storage
  (PVC), Redis persistence enabled, and Celery late-acknowledgement configured
- **Celery beat** runs in one pod only, in that same datacentre
- MSSQL is reachable from all pods

### 1.4 Multi-tenancy model

The system is **team-based**. Multiple teams onboard onto the platform. Each team has
its own F5 cluster (a set of F5 devices operating as one load-balanced target). Team
configuration is stored in Redis and includes:

- Per-team API call caps for POST, PUT and DELETE operations, expressed per hour
- The F5 cluster that team's requests target
- Circuit breaker configuration for that team's cluster

There is also a **global system capacity**: approximately 5,000 requests per hour,
with each team allocated a weighted share of that total.

### 1.5 Protection mechanisms already implemented

These exist and are tested. Do not rebuild them. Understand them, because the work
packages below interact with them.

**Leaky bucket rate limiting.** Constrains how many requests can be in flight against
a specific F5 cluster at a time, per team, per that team's configured limits. This
exists because F5 devices degrade badly above roughly fifteen concurrent API calls,
and Infoblox has similar constraints. The rate limiter is the primary protection for
the downstream network devices.

**Circuit breaker.** If a specific F5 cluster becomes overwhelmed or stops responding,
and there is no healthy alternative in that cluster, requests for that team against
that cluster are stopped. After a cool-down period the breaker moves to a half-open
state and allows a probe request through. If the probe succeeds, the breaker fully
reopens.

**Global admission control.** Enforces the overall system capacity ceiling and the
per-team weighting.

**Database-backed request state.** Every request is written to MSSQL before being
enqueued, and its state is tracked there through the workflow.

### 1.6 The two failure points this document addresses

1. **Redis is a single point of failure.** It is one instance. If it stops, the
   coordination layer stops.
2. **Celery beat is a single point of failure.** It runs on one pod, because
   scheduled tasks such as rollback verification and reconciliation must not run
   twice concurrently.

---

## 2. Architectural decisions governing this work

These are settled. Do not reopen them, propose alternatives, or design around them.
They are recorded here so that the reasoning is available to anyone reading the code
later.

### D-R1 — Redis is not authoritative for correctness

Redis holds coordination state: rate limit counters, in-flight slot counts, circuit
breaker state, and cached team settings. It does **not** hold anything that must be
correct for the estate to be safe.

The reason is that Redis replication is asynchronous. If a replica is ever promoted,
recent writes can be lost. If mutual exclusion depended on a Redis lock, a failover
could allow two workers to hold the same lock simultaneously, producing two concurrent
writers against the same WideIP — precisely the outcome the whole design exists to
prevent.

**Mutual exclusion is enforced in MSSQL**, through the unique constraint on active
FQDN rows and the atomic state claim described in WP-R1. Redis is the fast path.
The database is the truth.

### D-R2 — On Redis unavailability, the system fails closed

New work is rejected. In-flight work continues. This is a deliberate choice: the rate
limiter is what protects the F5 devices from being overwhelmed, and accepting work
without it would risk damaging the network estate. A period of rejections with a
`Retry-After` header is recoverable. Overwhelming an F5 cluster is not.

### D-R3 — Settings fall back to the database; counters do not

Team settings, thresholds and cluster configuration are read-mostly and change rarely.
They can safely be served from MSSQL, and from an in-process cache, when Redis is
unavailable.

Live counters — requests used this hour, slots currently in flight, breaker state —
change constantly and are shared across four pods. They cannot be served from a
per-pod cache, because four pods each holding their own copy of "requests used this
hour" would produce four independent budgets and therefore no effective limit at all.
They must not be moved into MSSQL either, because that would mean a database write per
operation against the shared system of record.

**Consequence:** during a Redis outage, settings remain readable, counters do not
exist, and therefore new work is rejected per D-R2.

### D-R4 — No additional caching component

Do not introduce a cache pod, a sidecar cache, a file-based cache, or a PVC-backed
cache. A cache pod is Redis with extra steps and an additional failure mode. A
file-based cache introduces write handling, corruption handling and concurrent access
problems that an in-process dictionary does not have.

The cache in WP-R2 is a plain Python dictionary in each pod's process memory. When a
pod restarts, its cache is empty and repopulates on first use. That is correct
behaviour, not a defect.

### D-R5 — Not migrating to a workflow engine

A migration to a durable execution engine (Temporal or similar) is out of scope. The
workflows here are short — a handful of API calls with compensation, completing in
minutes — and the compensation logic is already built and tested. Self-hosting such an
engine in this environment would mean operating a multi-service cluster plus its own
persistence layer, none of which is managed. That relocates the operational risk
rather than removing it.

This decision may be revisited if the organisation stands up an operated platform
service with a defined owner and availability target. It is not a reason to delay this
hardening work.

---

## 3. WP-R1 — Verify redelivery safety of the task claim

**Type:** Verification first. Write code only if the verification fails.
**Do this before WP-R2 and WP-R3.**

### 3.1 The problem

Celery is configured with late acknowledgement. This means a task message is removed
from Redis only after the worker finishes processing it, not when it is received. That
is the correct setting — it means a worker crash mid-task results in the task being
redelivered rather than lost.

It also creates a specific window:

1. A worker completes a task and writes the successful result to MSSQL.
2. Before the acknowledgement reaches Redis, Redis becomes unavailable, or the worker
   process dies.
3. Redis recovers. The task message is still present, because it was never acked.
4. The task is redelivered to a worker.
5. A second worker begins executing work that has already completed.

For a provisioning workflow that mutates F5 and Infoblox state, a duplicate execution
is not harmless. It could attempt to re-create objects, or worse, trigger a rollback
path against an estate that is already in its intended final state.

**This scenario has already occurred in this environment.** During a deployment, Redis
was terminated while tasks were in flight. The in-flight tasks completed, but the
acknowledgement error logs were not examined and the final status of the submitted
requests was not verified afterwards. The behaviour is therefore unproven, not known
to be broken.

### 3.2 What to verify

Locate the point in the worker code where a task transitions a request from its queued
state to its running state.

Determine whether that transition is **atomic** — that is, whether the claim is
performed as a single conditional database operation, or as a read followed by a
separate decision in Python.

An atomic claim looks structurally like this: a single `UPDATE` that sets the state to
running **conditioned on** the state currently being queued, followed by inspecting
the affected row count. If the row count is zero, another worker already owns the task,
and this worker must exit without performing any work.

A non-atomic claim looks like this: a `SELECT` to read current state, a Python `if`
to decide whether to proceed, then an `UPDATE`. This is unsafe. Two workers can both
execute the `SELECT` before either executes the `UPDATE`, both see the queued state,
and both proceed.

### 3.3 Report before changing anything

Produce a short written finding covering:

- The exact file and line where the claim occurs
- Whether it is atomic, as defined above
- Whether the affected row count is actually inspected and acted upon, or discarded
- What the worker does when it loses the claim — does it exit silently, log, or raise?
- Whether the same guard covers all task types, or only the main provisioning path
  (check scheduled tasks, rollback verification tasks, and reconciliation tasks
  separately — these are the ones most likely to have been written without the guard)
- Whether each workflow step is independently idempotent, meaning it reads current
  state, compares to desired state, and performs a no-op when they already match

### 3.4 Acceptance criteria

- **AC-R1.1** — The claim is a single conditional `UPDATE` guarded on the current
  state, not a read-then-decide sequence.
- **AC-R1.2** — The affected row count is inspected. A count of zero causes the worker
  to return without performing any external API calls.
- **AC-R1.3** — Losing the claim is logged at info level with the request identifier,
  and is not treated as an error condition. It is expected behaviour under redelivery.
- **AC-R1.4** — The guard covers every task type that mutates estate state, including
  scheduled and reconciliation tasks.
- **AC-R1.5** — Every workflow step contains an explicit no-op branch for the case
  where current state already matches desired state, and that branch does not raise.

### 3.5 Implementation rules

**Do not add a second guard on top of an existing correct one.** If the atomic claim
is already present and correct, the correct outcome of this work package is a written
confirmation and no code change. Two overlapping half-correct mechanisms are worse than
one correct mechanism.

If the claim is not atomic, make it atomic. Do not introduce a Redis-based lock as a
substitute — see D-R1.

---

## 4. WP-R2 — In-process settings cache with database fallback

**Type:** Code.
**Depends on:** WP-R1 complete.

### 4.1 The problem

Team settings — per-team API caps, cluster mapping, circuit breaker thresholds — are
currently read from Redis on the request path. There is already a fallback that reads
them from MSSQL when Redis is unavailable.

That fallback is correct but expensive. Reading settings from the database on every
request would place significant load on the shared MSSQL instance and could not be
sustained for the duration of a Redis outage.

### 4.2 The design

Add a per-process in-memory cache in front of both Redis and the database.

Behaviour on a settings read:

1. Check the in-process cache. If an entry exists and its age is below the TTL, return
   it. Make no network call.
2. If the entry is missing or stale, read from Redis. On success, store the value and
   the current timestamp in the cache, and return it.
3. If Redis is unavailable, read from MSSQL. On success, store the value and timestamp
   in the cache, and return it.
4. If both are unavailable, raise. The caller handles this per WP-R3.

The cache is a module-level dictionary keyed by whatever identifies a settings record
(team identifier, or team plus cluster, matching the existing data model). Each entry
stores both the value and the timestamp at which it was fetched.

### 4.3 Design constraints

**TTL.** Default 60 seconds, and it must be configurable rather than hardcoded. The
practical effect is that a settings change takes up to 60 seconds to become effective
across all four pods. For rate limit configuration this is acceptable. Document this
staleness window in the code comment, so that whoever changes a team's cap in future
understands why it does not take effect instantly.

**Scope — this is the critical constraint.** The cache is only for values where
staleness is harmless. Cache these:

- Team settings and per-team caps
- Circuit breaker thresholds (the configured limits, not the live state)
- Cluster membership and device mapping
- Global capacity configuration and team weightings

**Never cache these:**

- Requests used this hour, or any live counter
- Current in-flight slot counts
- Circuit breaker current state (open, closed, half-open)
- Anything that four pods must agree on in real time

Caching any item from the second list would produce four independent views of shared
state and silently defeat the rate limiting.

**Thread safety.** Celery workers and the FastAPI application are concurrent. The cache
must be safe under concurrent access. A simple lock around read-modify-write is
sufficient; do not over-engineer this.

**Cache benefit is not limited to outages.** Even when Redis is healthy, this removes a
network call per request for a value that typically changed weeks ago.

### 4.4 Acceptance criteria

- **AC-R2.1** — A settings read with a fresh cache entry performs zero network calls.
  Prove this with a test that asserts the Redis client was not called.
- **AC-R2.2** — A settings read with a stale entry refreshes from Redis and updates
  both the value and the timestamp.
- **AC-R2.3** — When the Redis client raises, the read falls back to MSSQL, and the
  result is cached.
- **AC-R2.4** — When both Redis and MSSQL are unavailable, the read raises a distinct,
  named exception that the admission path can catch.
- **AC-R2.5** — TTL is read from configuration, not hardcoded.
- **AC-R2.6** — Concurrent reads from multiple threads do not corrupt the cache and do
  not produce duplicate fetches of the same key within a single TTL window.
- **AC-R2.7** — No live counter, in-flight count, or breaker state is stored in the
  cache. Verify by inspection and state this explicitly in the implementation report.
- **AC-R2.8** — Cache contents are exposed on the existing status or health endpoint
  for debugging: keys held, and the age of each entry.

---

## 5. WP-R3 — Explicit fail-closed behaviour when Redis is unavailable

**Type:** Code.
**Depends on:** WP-R2 complete.

### 5.1 The problem

The current behaviour when Redis is unavailable is largely emergent rather than
designed. Some paths may raise, some may time out, some may surface as a generic 500.
The system's behaviour during its most likely failure mode should be a deliberate,
documented, tested decision — not whatever happens to fall out of the exception
handling.

### 5.2 Required behaviour

Define and implement three explicit operating modes. Name them in code, log
transitions between them, and expose the current mode on the status endpoint.

**FULL** — Redis reachable. Normal operation. All limits enforced from live counters.

**DEGRADED** — Redis unreachable. Behaviour:

- New provisioning requests are rejected at the admission point with HTTP **503** and a
  `Retry-After` header. The response body must state clearly that the coordination
  layer is unavailable and the request was not accepted, so that the consumer does not
  interpret it as a partial or ambiguous outcome.
- The rejection happens **before** any request row is written to MSSQL and before any
  external API call. A rejected request must leave no trace in the estate or the
  database.
- **In-flight tasks continue to completion.** A worker already executing a workflow
  must be able to finish without any further Redis call. Verify this explicitly —
  if any step in the workflow reaches back to Redis mid-execution, that is a defect
  to fix under this work package.
- Settings reads continue via the WP-R2 cache and database fallback.
- Read-only endpoints — request status lookups, health checks — continue to serve
  normally from MSSQL. Do not fail these; the consumer needs status visibility most
  during an outage.
- Queued but unstarted tasks remain in Redis persistence and resume when Redis
  recovers. No action needed, but state this in the runbook so operators do not assume
  the queue was lost.

**RECOVERING** — Redis has returned. Behaviour:

- Counters have been lost or are stale. They rebuild naturally as new work arrives.
- Do not attempt to reconstruct historical counter values from the database. The
  hourly window will be under-counted for the remainder of that window. This is
  accepted and should be documented rather than engineered around.
- Circuit breakers start in the closed state. If a downstream F5 cluster is still
  unhealthy, the breaker reopens on the first failures, which is the correct behaviour.
- Redelivered tasks are handled by the WP-R1 claim.

### 5.3 Detection and transition

Mode is determined by the outcome of Redis operations, not by a separate health-check
loop that could disagree with reality. A configurable number of consecutive Redis
failures moves the system to DEGRADED; a successful operation moves it back.

Redis client calls must have a short, explicit connect and operation timeout. Without
one, an unreachable Redis produces hanging requests rather than fast rejections, which
is a worse failure than the outage itself. If timeouts are not currently set on the
Redis client, set them as part of this work package and report the previous values.

### 5.4 Acceptance criteria

- **AC-R3.1** — Three named modes exist in code, with logged transitions including the
  triggering cause.
- **AC-R3.2** — In DEGRADED mode, a POST, PUT or DELETE returns 503 with a
  `Retry-After` header and an explicit message.
- **AC-R3.3** — A request rejected in DEGRADED mode creates no MSSQL row and makes no
  F5 or Infoblox call. Assert both.
- **AC-R3.4** — In DEGRADED mode, status lookups and health endpoints still return
  successfully.
- **AC-R3.5** — A workflow already in execution completes successfully with the Redis
  client raising on every call. This is the key static-stability test: the in-flight
  path must not touch Redis.
- **AC-R3.6** — The Redis client has explicit connect and operation timeouts, and an
  unreachable Redis produces a rejection within that timeout rather than a hang.
- **AC-R3.7** — On recovery, the mode returns to FULL, and new requests are accepted
  again without a restart.
- **AC-R3.8** — Current mode is visible on the status endpoint.

---

## 6. WP-R4 — Celery beat high availability via database lock

**Type:** Code. Independent of R1–R3, and safe to run in parallel by a separate agent —
it touches different files.

### 6.1 The problem

Celery beat runs on a single pod. If that pod is lost, all scheduled work stops —
reconciliation, rollback verification, and the reclaim sweeper.

The reason it is single-instance is correct as far as it goes: running two beat
schedulers would put each scheduled tick on the queue twice, and these tasks must not
run concurrently.

However, "must not run twice" is an argument for a **lock**, not for a single instance.

### 6.2 What beat actually does — read this before designing anything

This is worth stating plainly because it determines the entire design, and it is easy
to get wrong.

**Beat is a clock, not an executor.** When a schedule fires, beat places a single
message on a queue and does nothing else. It does not notify workers, it does not
broadcast, and it does not execute the task.

Consequently there are three separate guarantees, provided by three separate
mechanisms:

| Guarantee | Provided by | Layer |
|-----------|-------------|-------|
| The tick is **dispatched** once | The MSSQL leadership lease (this work package) | Scheduling |
| The message is **delivered** to one consumer | Celery queue semantics — a queue is not pub-sub; each message goes to exactly one worker | Transport |
| The task **executes** at most once even under redelivery | The atomic claim (WP-R1) | Execution |

Because delivery is already single-consumer, it does not matter which pod's worker
executes a scheduled task. Only dispatch needs protecting. Design accordingly, and do
not add mechanisms that duplicate a guarantee already held by another layer.

### 6.3 The design

Run beat on multiple pods — all of them, or at minimum one per datacentre. Before
dispatching any scheduled tick, the beat instance must hold a short-lived **leadership
lease** in MSSQL. Only the lease holder dispatches. The others idle and retry.

MSSQL is the arbiter because it is the managed, resilient service already trusted as
the system of record. Using Redis for this lease would violate D-R1 and would make beat
availability depend on the very component this work exists to harden.

#### 6.3.1 The lease table

A single row in a dedicated table holding, at minimum:

- A fixed lock name or identifier (so the table can hold other leases later)
- The current holder's identity — pod name or a generated instance identifier
- The lease expiry timestamp
- Optionally, an acquisition timestamp for observability

#### 6.3.2 Acquisition — one statement, both cases

Acquisition is a **single conditional `UPDATE`** that matches both the unheld case and
the expired case in the same statement. Take the lease if there is no current holder
**or** if the recorded expiry is already in the past.

Handling both cases in one statement means a dead leader's stale lock is taken over by
the ordinary acquisition path. There is no separate cleanup step, no reaper process,
and no window in which the row is neither held nor claimable.

Inspect the affected row count. One means this instance is now the leader. Zero means
another instance holds a valid lease — sleep and retry. This is the same atomicity
requirement as WP-R1, and it works for the same reason: the decision is made inside a
single database statement, not across a read and a subsequent write in Python.

#### 6.3.3 All pods race freely — no priority flag

Every beat instance attempts acquisition on its own retry interval. There is no
designated first-attempter, no priority pod, and no configuration flag electing a
preferred leader.

Contention is not a problem to be avoided here. MSSQL serialises the conditional update
internally: exactly one attempt matches the condition and affects one row, and every
other concurrent attempt affects zero rows and backs off. There is no window in which
two instances both believe they won.

**If the existing implementation has a "beat enabled" flag or a designated pod that
tries first, remove it.** That pattern encodes an ordering assumption, and an ordering
assumption is a failure mode. Free contention against an atomic statement is both
simpler and safer.

#### 6.3.4 Use the database clock, never the pod clock

Compute the expiry inside the statement, as database-now plus the lease duration. Do
not compute an expiry timestamp in Python and write it in.

Four pods will have slightly different system times. If each writes its own idea of
"now", one pod can consider a lease still valid while another considers it expired,
which reintroduces exactly the ambiguity the lease is meant to remove. A single clock —
the database's — eliminates that entire class of bug. Apply this to both acquisition
and renewal.

#### 6.3.5 Jitter on the retry interval

Add a small random jitter to each instance's retry interval, so the non-leaders do not
poll in lockstep.

This is not a correctness requirement — the atomic update is correct regardless. It
avoids a pointless synchronised burst of identical queries against MSSQL every retry
cycle. A jitter of roughly ten to twenty percent of the interval is sufficient.

#### 6.3.6 Renewal is a separate, differently-conditioned statement

Once an instance holds the lease it **renews**, it does not reacquire. Renewal is its
own statement, and its condition is different from acquisition's: extend the expiry
**only if this instance is still the recorded holder**.

That condition matters. If the instance was displaced while it was paused, blocked, or
descheduled, it must not silently take the lease back part-way through a tick. Losing
the renewal is the signal that it is no longer leader.

**If renewal fails — for any reason, including a database error, a timeout, or a zero
row count — dispatch stops immediately.** Do not continue dispatching on the assumption
that leadership held a moment ago. An instance that cannot prove it holds a valid lease
is not the leader.

#### 6.3.7 Timing

Starting values: **lease duration 30 seconds, renewal interval 10 seconds.** Both must
be configurable, not hardcoded.

The governing rule is that **lease duration should be roughly three times the renewal
interval.** That margin means two or three consecutive renewals must fail before the
lease lapses, so a brief database slowdown does not cost leadership. Setting them close
together produces leadership flapping, which is worse than a slightly longer handover
gap.

Two things to account for when confirming these values:

- **Database calls here are Kerberos-authenticated and are not instantaneous.** Do not
  assume a renewal completes in milliseconds. Measure the actual round-trip time in the
  dev environment and confirm the renewal interval leaves comfortable headroom above
  it. Report the measured value.
- **The scheduled task interval sets the tolerance for the handover gap.** With
  scheduled tasks running about once per minute, a worst-case 30-second gap is well
  inside a single tick interval and costs nothing. If any schedule runs at a much
  shorter interval, revisit the lease duration — but note the trade: a shorter lease
  means more frequent renewal traffic against MSSQL.

#### 6.3.8 The accepted cost

After a leader dies, no scheduled dispatch occurs until its lease expires — up to the
full lease duration. That gap is the price of guaranteeing no double dispatch, and it
is the correct trade for tasks like rollback verification and reconciliation, which
must never run concurrently. Document the gap duration in the runbook so operators
recognise it as designed behaviour rather than a fault.

### 6.4 Bulkheading the sweeper and reconciler tasks

This is part of WP-R4 because it is what makes multi-pod beat worth having.

#### 6.4.1 The problem

Sweeper and reconciler tasks currently share a worker pool with provisioning work. They
therefore compete for the same concurrency slots. Sweepers are cheap and frequent;
provisioning workflows are slow and heavy. Mixing them means a burst of provisioning
can delay reconciliation, and a long reconciliation run consumes slots that
provisioning needs.

#### 6.4.2 The design

Route scheduled maintenance tasks — sweeper, reconciler, rollback verification — to a
**dedicated queue**, consumed by a small dedicated worker with low concurrency (five is
ample) **running on every pod**.

Provisioning workers then consume only the provisioning queues, uninterrupted.

The general pattern is called **bulkheading**: isolating resource pools so that one
class of work cannot exhaust the capacity another class depends on. It is the same
family of idea as the circuit breaker already in the system — both come from Michael
Nygard's *Release It!*

#### 6.4.3 Run the sweeper workers everywhere, always

The sweeper workers run on all pods, continuously. They are not started, stopped, or
scaled in response to leadership.

**Beat leadership controls dispatch only. It must never control worker lifecycle.**

The reasoning follows directly from §6.2. The lease already guarantees the tick is
dispatched once, so exactly one message reaches the queue. Queue semantics already
guarantee that message goes to exactly one consumer. The atomic claim already
guarantees a redelivered message cannot execute twice. Tying worker startup to
leadership would add a fourth mechanism enforcing a property three layers already
enforce — and would introduce a genuinely new failure mode, in which worker processes
are being torn down and started during a leadership handover. Dynamic process
management under failure conditions is exactly the kind of thing that breaks at three
in the morning.

Idle sweeper workers cost almost nothing; they sit blocked on an empty queue. Running
them everywhere is what gives the design its resilience: if the pod that would have
executed a task is gone, another pod's worker takes the message with no coordination
at all.

The clean separation to hold in mind:

> **Leadership decides *when*. The queue decides *who*. The claim guarantees *once*.**

### 6.5 Acceptance criteria

- **AC-R4.1** — Multiple beat instances run simultaneously and only the lease holder
  dispatches. Prove with a test asserting exactly one dispatch across two concurrent
  instances over several tick intervals.
- **AC-R4.2** — Acquisition is a single conditional statement covering both the unheld
  and expired cases, with the affected row count inspected and acted upon.
- **AC-R4.3** — Expiry is computed from the database clock in both acquisition and
  renewal. No expiry timestamp is generated in application code.
- **AC-R4.4** — No priority flag, preferred pod, or ordering exists between beat
  instances. All instances attempt acquisition on the same footing. If such a flag
  existed previously, it has been removed.
- **AC-R4.5** — The retry interval carries random jitter.
- **AC-R4.6** — Renewal is a separate statement conditional on the instance still being
  the recorded holder.
- **AC-R4.7** — A failed renewal — error, timeout, or zero rows affected — stops
  dispatch immediately. Assert that no dispatch occurs after a renewal failure without
  a fresh successful acquisition.
- **AC-R4.8** — Killing the leader results in another instance acquiring leadership
  within the lease expiry, with no manual intervention.
- **AC-R4.9** — No scheduled task is dispatched twice during a leadership handover.
  **This is the criterion that matters most; test it explicitly and repeatedly, since
  the failure is timing-dependent and a single passing run proves little.**
- **AC-R4.10** — Loss of MSSQL connectivity causes the current leader to stop
  dispatching rather than continue on an unverifiable lease.
- **AC-R4.11** — Lease duration and renewal interval are configuration values, with the
  three-to-one ratio documented. The measured Kerberos-authenticated round-trip time is
  reported alongside the chosen renewal interval.
- **AC-R4.12** — Sweeper, reconciler and rollback-verification tasks route to a
  dedicated queue, consumed by low-concurrency workers running on every pod.
- **AC-R4.13** — Sweeper worker processes are independent of leadership. Killing or
  transferring the leader does not start or stop any worker.
- **AC-R4.14** — Current leader identity, lease expiry, and time until expiry are
  visible on the status endpoint.

## 7. WP-R5 — Test suite and failure drill runbook

### 7.1 Test tiering

Tests are split into two tiers, and the split follows one rule:

> Use a fake for your own logic. Use real infrastructure whenever the infrastructure's
> own behaviour is the thing under test.

The reason is that a fake only confirms your mental model of how the infrastructure
behaves — and your mental model is precisely the thing that might be wrong. A fake
database will happily confirm that two concurrent claims behave the way you assumed.
Only a real database tells you what actually happens.

**Tier 1 — Fast, isolated, fakes.** Runs on every commit, no infrastructure required.

**Tier 2 — Integration, real Redis and real MSSQL.** Runs in the isolated dev
environment.

Mark every test with its tier and, in a comment, why it belongs there.

### 7.2 Tier 1 tests — fakes

| ID | Test | Asserts |
|----|------|---------|
| T1-01 | Cache hit within TTL | Zero calls to the Redis client |
| T1-02 | Cache miss after TTL expiry | Refetches and updates timestamp |
| T1-03 | Redis client raises | Falls back to database, caches result |
| T1-04 | Both sources raise | Raises the named exception |
| T1-05 | TTL from configuration | Non-default TTL is honoured |
| T1-06 | Cache scope inspection | No counter or breaker-state key present |
| T1-07 | Mode transition to DEGRADED | Occurs after configured consecutive failures |
| T1-08 | Mode transition back to FULL | Occurs on first success |
| T1-09 | DEGRADED rejects writes | 503 with `Retry-After` on POST, PUT, DELETE |
| T1-10 | DEGRADED allows reads | Status endpoint returns 200 |
| T1-11 | Rejection leaves no trace | No MSSQL write, no external call |
| T1-12 | Step no-op branch | Step with matching current state does not raise, makes no write call |
| T1-13 | Lost claim path | Worker with zero affected rows performs no external call |
| T1-14 | Rollback on CNAME create failure | All F5 objects created in that request are removed |
| T1-15 | Rollback on CNAME delete failure | F5 objects recreated, CNAME existence re-verified |

### 7.3 Tier 2 tests — real infrastructure

| ID | Test | Asserts | Why real |
|----|------|---------|----------|
| T2-01 | Two concurrent workers claim the same request | Exactly one succeeds; the other sees zero affected rows | Testing MSSQL's actual concurrency semantics |
| T2-02 | Concurrent claim under load, repeated many times | No double claim across all iterations | Races are probabilistic; a single run proves little |
| T2-03 | Unique constraint on active FQDN | Second insert for the same active FQDN is rejected by the database | Testing the constraint itself |
| T2-04 | Rate limiter Lua under concurrency | Limit is never exceeded across concurrent callers | Fake Redis does not reproduce real atomicity |
| T2-05 | Circuit breaker state under concurrency | State transitions are consistent across pods | Shared state semantics |
| T2-06 | Two beat instances, one lock | Exactly one dispatch per tick | Testing MSSQL lock semantics |
| T2-07 | Beat leadership handover, repeated | New leader acquires within lease expiry; no double dispatch across many iterations | Timing-dependent; one run proves little |
| T2-10 | Renewal fails mid-tenure | Dispatch stops immediately; resumes only after fresh acquisition | Real database failure behaviour |
| T2-11 | Sweeper workers on all pods, one dispatch | Exactly one worker executes; others receive nothing | Real queue delivery semantics |
| T2-08 | Full workflow with Redis raising mid-execution | Workflow completes; no Redis call in the execution path | The static stability claim |
| T2-09 | Settings read with Redis stopped | Serves from cache, then from database | Real client failure modes differ from fakes |

### 7.4 Failure drills

These are semi-manual. Scripted setup and scripted verification, but a human reads and
records the outcome. **These drills are the evidence.** "Redis has never failed" is a
weak claim in a design review. "Here is what happens in the first sixty seconds when it
does, with timestamps" is not.

For each drill record: start time, exact action taken, observed behaviour, consumer-
visible impact, recovery time, and anything unexpected.

---

**Drill 1 — Redis killed with tasks in flight**

- Setup: submit enough concurrent requests that several workflows are mid-execution and
  several are queued but unstarted.
- Action: terminate the Redis pod.
- Observe: do in-flight workflows complete? What is the final MSSQL status of each?
  Do new requests receive 503 with `Retry-After`? Do status lookups still work? What
  appears in the acknowledgement error logs?
- Then: restart Redis. Do queued tasks resume? Are any tasks redelivered? If so, does
  the claim prevent duplicate execution? **Verify the final status of every submitted
  request against the actual estate state on F5 and Infoblox** — not just against the
  database.
- Pass: no duplicate execution, no orphaned estate objects, no request in an ambiguous
  state, all rejections clean.

**Drill 2 — Redis killed mid-rollback**

- Setup: induce a failure that triggers rollback — a deliberately failing CNAME
  operation is the cleanest trigger.
- Action: terminate Redis while the compensation is executing.
- Observe: does rollback complete? Is the estate returned to its prior state, or left
  partially compensated?
- Pass: rollback completes; estate matches pre-request state; any incomplete
  compensation is recorded in a state that requires attention rather than silently
  marked complete.

**Drill 3 — Beat pod killed during a scheduled run**

- Setup: run with multiple beat instances (post WP-R4), with a scheduled task executing.
- Action: kill the current leader.
- Observe: time until a new leader acquires. Is the interrupted task redispatched? Is
  anything dispatched twice?
- Pass: leadership transfers within the lease window; no double dispatch.

**Drill 4 — Datacentre partition**

- Setup: normal load across both datacentres.
- Action: block network between the two datacentres.
- Observe: pods in the datacentre without Redis — what do they do? Do they fail closed
  cleanly, or hang? Are there hanging requests indicating a missing timeout? What
  happens to MSSQL connectivity from each side?
- Pass: pods without Redis access degrade per WP-R3 within the configured timeout; no
  hangs; no duplicate processing when the partition heals.

**Drill 5 — F5 cluster unresponsive**

- Setup: normal load against a team's cluster.
- Action: make the target F5 cluster unreachable or unresponsive.
- Observe: does the breaker open within expectation? Are other teams unaffected? Does
  the half-open probe behave correctly on recovery?
- Pass: breaker opens, isolation to the affected team holds, recovery is automatic.

**Drill 6 — MSSQL brief unavailability**

- Setup: normal load.
- Action: interrupt MSSQL connectivity briefly.
- Observe: what happens to in-flight workflows, to the admission path, and to the beat
  lease?
- Pass: no estate mutation occurs that cannot be recorded; beat stops dispatching
  without a valid lease; recovery is clean.

### 7.5 Observation exercise — Redis logical database split

**This is an observation exercise to run during the drills, not a required change and
not a resilience measure.**

While running the drills, try splitting Redis logical databases by concern on the same
instance: database zero for application settings and cached configuration, database one
for the Celery broker and result backend. This requires changing only the database
index in the existing connection URLs — same host, port and authentication.

The purpose is operational visibility during an incident. With everything on one
database, inspecting configuration keys means scanning past thousands of Celery broker
entries. Separating them makes the configuration keyspace readable while a drill is in
progress.

Record whether it materially improved inspection during the drills. If it did not, drop
it.

**Numbered databases provide namespacing, not isolation.** They share one process, one
memory pool and one persistence file, so memory pressure or a slow command in one
affects the other identically, and they are unsupported under Redis Cluster. This split
must not be presented as improving resilience, and it does not change any failure mode
described in this document. Genuine isolation would require separate Redis instances.

Two points that apply regardless of whether the split is adopted:

- **Key prefixes grouped by concern are the primary mechanism** — settings, counters,
  breaker state, each under its own prefix. Apply these whether or not the databases
  are split.
- **Tooling and scripts must use `SCAN` with a match pattern, never `KEYS`.** `KEYS`
  blocks the entire server while it scans, which on a production keyspace under load is
  dangerous. This matters more than the database split does.

### 7.6 Deliverable — the failure modes document

Produce `docs/failure-modes.md` containing, for each failure: what fails, what the
consumer sees, what happens to in-flight work, recovery behaviour, expected recovery
time, and the date and result of the drill that verified it.

This is a one-page operational document, not an essay. Its purpose is to be readable
during an incident and defensible in a design review.

---

## 8. Rules for the implementing agent

1. **Do not search the internet for background on Redis, Celery, MSSQL or resilience
   patterns.** Everything required is in this document. If something here is genuinely
   ambiguous, ask rather than researching around it.

2. **Verify before you build.** WP-R1 may correctly result in zero code changes. That
   is a successful outcome, not a wasted work package.

3. **Never introduce a second guard on top of a correct one.** Overlapping partial
   mechanisms are harder to reason about than one correct mechanism.

4. **Never make Redis authoritative for mutual exclusion.** Per D-R1.

5. **Never cache live counters or breaker state.** Per D-R3. This is the single most
   dangerous mistake available in this work.

6. **Feature-flag every behavioural change.** This system is live and in UAT. Existing
   behaviour must continue working until each new component is explicitly enabled.

7. **Every external operation stays idempotent.** Read current state, compare, act only
   on difference, no-op when identical, never error on a second run.

8. **Rollback never deletes pre-existing objects.** Objects created during a request are
   removed on rollback. Objects that existed beforehand are restored, never deleted.

9. **A timeout is an unknown outcome, not a failure.** Never blind-retry after a
   timeout. Read back actual state first, then converge.

10. **Ask rather than assume** about existing table shapes, function signatures, or
    configuration structure. Do not guess and build on the guess.

11. **Do not invent numeric values** for TTLs, timeouts, lease durations or failure
    thresholds without stating the reasoning. Where a value should come from
    measurement, leave a named configuration constant with a clear TODO rather than a
    plausible-looking number.

---

## 9. Suggested execution order

1. WP-R1 verification and report — no code until reviewed
2. WP-R4 in parallel, if using a second agent (it touches different files)
3. WP-R2
4. WP-R3
5. Tier 1 tests alongside each work package, not afterwards
6. Tier 2 tests once R1–R4 are merged
7. Drills in the isolated dev environment
8. `docs/failure-modes.md` written from the drill results

If using subagents: WP-R4 is safely parallel. WP-R2 and WP-R3 touch the same admission
and settings paths and must run sequentially.
