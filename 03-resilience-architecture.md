# 3. Resilience architecture

See [03-resilience-state-machine.png](./diagrams/03-resilience-state-machine.png).

**At a glance:** Bound every journey with deadlines and retry budgets. Use application
circuit breakers and ledger-backed idempotency; treat uncertain payments as pending
until checked. Apply backpressure and shed optional work before payment durability or
resolution. Detailed thresholds remain for baseline and owner review.

| Concern | Design choice |
|---|---|
| Admission and capacity | Layered limits; tested multi-AZ headroom; defer optional work first |
| Transient failure | One retry owner, bounded backoff with jitter, end-to-end deadline |
| External dependencies | Application-owned breakers with journey-specific safe fallbacks |
| Payment correctness | Aurora uniqueness and ledger state; unknown provider outcome stays pending |
| Event recovery | MSK offset-aware retry, quarantine, and authorized replay |

## 3.1 Layered controls

| Layer | Control | Boundary |
|---|---|---|
| Edge | AWS WAF managed protections and rate-based rules | Reduces unwanted traffic; does not enforce payment correctness |
| Gateway | API Gateway request validation, throttling, and quotas | Limits admission before EKS |
| Application | Authorization, business limits, deadlines, ledger invariants | Owns business decisions and payment safety |
| EKS capacity | Readiness checks, PodDisruptionBudgets, HPA, Karpenter, priority classes, bounded worker pools | Maintain tested headroom for surviving AZs; autoscaling is not instantaneous |

**Overload priority:** Protect payment submission, durable ledger writes, and payment
resolution. Defer optional reporting, analytics, and non-urgent batch work first. Do not
silently discard durable payment events or shed reconciliation needed for accepted or
pending payments. If the ledger cannot durably record a new payment, do not report it as
accepted or send it for execution.

## 3.2 Deadlines and retry budgets

Each customer journey has an end-to-end deadline. Allocate shorter deadlines to
downstream calls so the caller retains time to return a useful response. Set the actual
durations from measured latency, provider contracts, and Product/Risk expectations;
the assessment does not invent numeric thresholds. Assign one layer to own retries so
independent retry loops do not multiply traffic.

| Journey or operation | Retry treatment |
|---|---|
| Sign-in | Retry only classified transient identity failures within the remaining deadline. Never retry invalid credentials or definitive rejection. |
| Conditional risk check | Use the Product/Security-approved fallback. Block or defer elevated-risk sign-in if no safe decision is available; do not wait indefinitely. |
| Balance read | Retry transient failures only while time remains. If freshness or accuracy is uncertain, return unavailable rather than stale-as-current data. |
| Payment submission | Resubmit with the same idempotency key to resolve the same operation. Reject a reused key with a different request. |
| Provider outcome unknown | Keep pending and query status or reconcile before retry. Retry only if the provider contract guarantees safe idempotency. |

For safe transient failures, use bounded exponential backoff with full jitter, a maximum
attempt count, and a total retry-time budget. Respect provider retry guidance such as
`Retry-After` when applicable. Track retry budget and concurrency by journey and
provider. When exhausted, fail fast or return a safe pending result according to the
operation. Definitive business outcomes, such as a payment decline, are not transient
failures and are not retried.

## 3.3 Circuit breaking and external dependencies

Implement business-aware circuit breakers in application services using a currently
supported, reviewed resilience library. Keep separate breaker and concurrency budgets
for each external provider so one provider cannot consume capacity reserved for another
dependency.

| State | Behavior |
|---|---|
| Closed | Calls proceed within deadlines and retry budgets. Classified timeouts/dependency failures count toward the threshold; definitive business responses do not. |
| Open | Stop calls for a bounded cool-down. Apply the approved fallback: risk-based sign-in decision, clear balance-unavailable response, or pending payment if the provider may have received the request. |
| Half-open | Permit a small number of safe probes. Close on successful recovery checks; reopen on failure. Prefer provider health/status checks; never probe with a synthetic funds transfer. |

**Mesh trade-off:** AWS App Mesh support ended September 30, 2026, so it is excluded.
Istio on EKS is optional and requires consistent operation of its control plane, proxies,
upgrades, policies, and resource overhead. It can enforce connection/request limits and
eject unhealthy upstream hosts, but it cannot determine payment state or retry safety.
Applications retain responsibility for business-aware breakers, idempotency, pending
outcomes, and reconciliation. Avoid mesh retries for payment writes unless provider
semantics and idempotency prove them safe. Compare Istio's operating cost with application
libraries; do not add a mesh solely to replace App Mesh.

## 3.4 Idempotency and deduplication

| Step | Correctness control |
|---|---|
| Request | Require an idempotency key, request-hash validation, and stable business operation ID. Reject key reuse with a different request. |
| Authority | Aurora stores the payment operation/state and enforces durable uniqueness on the idempotency scope. Redis is only a fast lookup or short-lived coordination aid. |
| Cache failure | A Redis miss, outage, eviction, or expiry falls back to Aurora and must never permit a duplicate financial effect. |
| Event publication | Commit payment state and outbox record atomically in Aurora; publish the outbox event to MSK. |
| Consumer | Deduplicate by stable event/payment ID and persist processing state. |

The ledger is proof of acceptance: if Aurora cannot durably record the payment, do not
claim acceptance or execute it externally. Unknown provider outcomes retain the original
operation identity and are resolved by status inquiry or reconciliation before retry.

## 3.5 Backpressure, queue recovery, and capacity

**Backpressure and catch-up:** API Gateway throttling, application concurrency limits,
bounded worker pools, and bounded producer buffers contain overload. Monitor MSK lag and
oldest-event age with consumer concurrency and partition capacity. Increase throughput
gradually after recovery so catch-up does not overwhelm Aurora or external providers.

**MSK retry and quarantine:** MSK uses offsets, not SQS-style visibility timeouts or
automatic DLQ redrive. If processing fails before the source offset commits, the message
can be delivered again to the same or reassigned consumer. Use bounded retries for
transient failures. For repeated failures, publish the event and failure context to a
quarantine topic, then advance the source offset only after that write succeeds.
Deduplicate quarantine and replay by original event ID. Publication may succeed while
offset commit fails, so quarantine consumers must tolerate duplicates.

**Ordering and replay:** If later events depend on a failed event, pause that partition or
use an explicitly validated sequencing and reconciliation design. Replay is an
authorized, audited, selective republish after correcting the cause, not a customer
resubmission. Kafka transactions can coordinate Kafka offsets and output records, but
cannot include Aurora transactions or external provider calls atomically. Keep
ledger-backed idempotency and reconciliation.

**Database and regional capacity:** Use bounded Aurora connection pools and restore
connections gradually after failover. Route reads to replicas only where consistency
allows; payment writes use the authoritative writer. Scale EKS within approved capacity
classes using HPA and Karpenter, maintaining tested multi-AZ headroom. Test the primary
Region under one-AZ loss. Broader regional degradation is governed by the separate DR
plan, not normal autoscaling.

## 3.6 Validation and trade-offs

The following are planned validation scenarios; they are not claimed as completed
tests:

| Scenario | Acceptable evidence |
|---|---|
| Provider latency/errors; retry amplification | Agreed retry budget holds, breaker limits calls, unrelated journeys retain capacity. |
| Provider timeout after possible payment receipt | Customer sees pending; one ledger operation remains authoritative; inquiry/reconciliation resolves without duplicate effect. |
| Redis unavailable or entry expired | Aurora idempotency prevents duplicates; latency may rise, correctness holds. |
| MSK lag, crash, repeated event failure | Lag/oldest age alert; retry/quarantine/replay respects ordering and avoids duplicate financial effects. |
| One AZ lost under load | Remaining AZs meet tested capacity/headroom with measured customer impact. |
| Demand exceeds safe capacity | Admission control activates; optional work is deferred; payment durability/resolution remains protected. |

Set traffic levels, timeout values, retry attempts, breaker thresholds, queue-age
limits, and headroom targets after baselining and confirming provider contracts. Record
customer impact, retries per original request, breaker state, queue age, ledger outcomes,
duplicate rate, and pending-payment resolution evidence.

Managed services reduce undifferentiated operations but create AWS coupling and quota
dependencies. Redis improves latency but cannot be the only correctness store. Active
cross-Region MSK mirroring reduces recovery work but increases replication cost and
requires topic ownership rules. Istio can add shared traffic controls but also creates
mesh operations and proxy overhead. Adopt it only if its shared controls justify that
cost. These trade-offs require measured tests and accountable owners.

