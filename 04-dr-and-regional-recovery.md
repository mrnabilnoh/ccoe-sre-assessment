# 4. Disaster recovery and regional recovery

See [04-dr-failover-sequence.png](./diagrams/04-dr-failover-sequence.png).

**At a glance:** Singapore is primary and Malaysia is warm standby. Humans authorize
promotion; SSM Automation runs the approved steps; ARC switches traffic after writer
fencing and readiness checks. Practice failover and failback in staging/UAT first, then
measure production RTO/RPO evidence through approved exercises.

| Decision | Approach |
|---|---|
| Recovery model | Singapore primary; Malaysia warm standby |
| Promotion authority | Human approval after writer fencing and replication/RPO review |
| Orchestration | Separate SSM Automation runbooks for failover and failback |
| Traffic switch | ARC routing controls only after database and application readiness |
| Recovery gates | Customer service restored first; business recovery complete after reconciliation |

## 4.1 Target topology

| Region | Role and readiness |
|---|---|
| Singapore (`ap-southeast-1`) | Primary |
| Malaysia (`ap-southeast-5`) | Warm standby with reduced EKS capacity, replicated configuration, Aurora Global Database secondary storage, MSK Replicator, provider credentials, and tested observability |

Validate recovery capacity after scale-up, service availability, quotas, and cost in
both Regions. Test CloudFront, WAF, API Gateway, and DNS behavior against both
destinations.

**Replication assumptions to prove:** Aurora Global Database replication behavior,
lag, promotion steps, and achievable RPO/RTO depend on the still-undecided engine and
version. MSK Replicator asynchronously copies approved topic data and supported Kafka
metadata, including consumer-group offsets and topic configuration. Measure lag; validate
offset behavior, topic ownership, ACLs, producer/consumer cutover, and partition ordering
in the selected Regions. Replication lag is a safety signal, not proof of zero data loss
or complete application consistency.

## 4.2 Detection, authority, and recovery orchestration

Failover is a serious, human-authorized action. Declare a regional incident when
customer-journey impact is supported by independent regional evidence. A single alarm
starts investigation; it does not start failover. A partial regional impairment is
possible, so do not assume the primary is completely down.

| Decision | Accountable role |
|---|---|
| Declare regional incident and coordinate response | Incident Commander (IC) |
| Verify fencing/replication evidence and approve promotion | DR Recovery Authority, normally the designated database/platform lead |
| Approve exception when five-minute RPO is unconfirmed or exceeded | Designated business and risk authority |
| Execute approved ARC routing change | Authorized operator or automation acting under that authority |
| Decide payment/customer impact; escalate provider issues | Payments journey owner with Product/Risk; provider relationship owner handles external escalation |

Use separate AWS Systems Manager (SSM) Automation runbooks for failover and failback.
SSM orchestrates repeatable checks/actions, records execution history, and pauses at
human approval gates. ARC provides on/off routing controls only; it does not promote
Aurora, fence the old writer, assess transaction correctness, or orchestrate recovery.
Prepare and validate the recovery Region before switching ARC. Never make ARC routing
trigger database promotion. Protect ARC with least-privilege IAM, change controls, safety
rules, and an audited break-glass procedure.

## 4.3 Failover runbook: Singapore to Malaysia

The RTO clock starts at authorized recovery declaration and stops when critical
customer journeys pass validation in Malaysia. Record customer-visible interruption
separately. The RPO applies to the agreed critical data set and is measured from
acknowledged, durable transactions absent from the promoted database. If the five-minute
RPO cannot be confirmed, pause the normal promotion path, estimate potential data loss,
and obtain documented business/risk approval for any exception.

1. **Declare and prepare:** Confirm customer-journey impact using independent regional
   evidence. Open the bridge, record UTC timestamps, freeze risky changes, and assign
   authorities.
2. **Collect recovery evidence:** Start the approved SSM failover runbook. Record Aurora
   lag and last acknowledged durable transaction, MSK replication/offset evidence,
   provider status, recovery readiness, and measured or uncertain RPO.
3. **Authorize:** Obtain DR Recovery Authority approval. Obtain documented business/risk
   approval for any RPO exception.
4. **Fence the writer:** Isolate Singapore's write path and verify it cannot accept
   writes. If fencing is uncertain, stop and escalate; do not promote while split brain
   remains possible.
5. **Promote the database:** Promote the Aurora Global Database secondary. Verify writer
   endpoint and application reconnect behavior for the selected engine/version.
6. **Cut over event ownership:** Move producer/consumer ownership to Malaysia. Verify
   topics, offsets, ACLs, permissions, replay boundaries, and single-writer enforcement.
7. **Prepare applications:** Activate/scale EKS. Verify capacity, IAM, KMS grants,
   secrets, certificates, network/security rules, provider allowlists, observability, and
   alarms.
8. **Validate before traffic:** Test sign-in, read-only balance, payment preflight that
   cannot move funds, provider status/connectivity, event processing, and
   ledger/reconciliation.
9. **Route traffic:** After readiness and authorized approval, switch ARC on/off controls
   to Malaysia. ARC has no percentage-based shifting. Include DNS/client convergence in
   measured recovery time.
10. **Restore customer service:** Observe CUJ SLIs, saturation, ledger writes, payment
    outcomes, pending age, duplicate indicators, MSK lag, and reconciliation. Announce
    service restored only after Gate A passes.
11. **Complete business recovery:** Reconcile pending/ambiguous payments against
    provider and ledger evidence. Declare recovery complete only after Gate B passes;
    otherwise report **service restored, reconciliation in progress** with an owner and
    deadline for outstanding items.

## 4.4 Failback runbook: Malaysia to Singapore

Singapore health is a reason to consider failback, not an automatic trigger. Malaysia
remains the only write-active Region until the separate, approved SSM failback runbook
completes.

1. **Repair:** Verify Singapore capacity, application versions, security controls,
   dependencies, provider allowlists, observability, and alarms.
2. **Synchronize:** Keep Malaysia authoritative while Aurora/MSK sync back. Validate
   replication direction and lag, topics, offsets, ACLs, and consumer ownership.
3. **Reconcile:** Resolve pending/ambiguous payments and delayed, quarantined, or replayed
   events. Confirm ledger/event consistency and retain an auditable discrepancy list.
4. **Validate:** Run all critical-journey readiness checks and verify rollback conditions.
   Never use a real funds transfer as a synthetic probe.
5. **Authorize:** Obtain DR Recovery Authority approval and business/risk approval for
   outstanding exceptions. Start the approved SSM failback workflow.
6. **Route and observe:** Switch ARC to Singapore, monitor journeys/payment outcomes, and
   keep Malaysia ready. If checks fail, pause and follow the approved rollback decision;
   do not switch back automatically.

## 4.5 DR success checklist

### Gate A: Customer service restored

| Check | Required evidence |
|---|---|
| Authority and data safety | Recovery approved; prior writer fenced; Aurora promoted; RPO within five minutes or documented exception approved |
| Event ownership | MSK ownership, offsets, topics, and permissions validated for active Region |
| Dependencies and capacity | EKS, IAM/KMS, secrets, network, provider allowlists, CloudFront/WAF/API routing, telemetry, and alerts ready |
| Customer journeys | Sign-in/balance pass; payment preflight is safe; submission has a safe outcome and durable ledger write |
| Traffic and timing | ARC points to recovery Region; critical journeys pass during observation; customer impact and RTO include DNS/client convergence |

### Gate B: Business recovery complete

| Check | Required evidence |
|---|---|
| Payments | Pending/ambiguous payments reconciled against provider and ledger evidence |
| Events | No unresolved duplicate financial effects; delayed/quarantined events accounted for and replayed only with authorization and idempotency controls |
| Governance | Communications, decisions, exceptions, evidence, owners, and follow-up actions recorded |

Service can be reported restored while owned reconciliation continues, but it must be
reported as **service restored, reconciliation in progress** until Gate B passes.

## 4.6 Staging-first game-day plan

Staging/UAT failover and failback practice is mandatory before any production exercise.
Use a representative environment to validate runbooks, approval roles, SSM Automation,
Aurora promotion, MSK cutover, ARC routing, dependencies, and reconciliation without
risking live funds. Include configuration, IAM/KMS, network, provider, and operational
gaps. Staging results do not prove production RTO/RPO.

| Practice | Scenarios and evidence |
|---|---|
| Cadence | At least quarterly and after material runbook/architecture changes |
| Scenarios | Partial regional degradation/false positive, writer fencing, Aurora promotion, MSK lag/consumer restart, provider timeout, pending-payment reconciliation, AZ capacity loss, failover, and failback |
| Production exercise | Separate controlled activity; requires explicit recovery, Product, Risk, and business approval, customer-impact safeguards, and stop/rollback criteria |
| Success evidence | Detection/declaration, approval/promotion, measured RPO and RTO, time to critical-journey recovery, single-writer proof, duplicate-payment results, reconciliation completion, decision records, and corrective actions retested |

AWS Resilience Hub can assess architecture against configured RTO/RPO policies and help
plan tests. Actual RTO/RPO evidence comes from timed recovery exercises and
transaction-level validation.

## 4.7 Assumptions and limitations

Regional service availability, Aurora Global Database behavior for the selected
engine/version, MSK Replicator topic semantics, provider allowlists, data residency,
and current critical-data volume must be validated in a representative account. Verify
that MSK Replicator supports the selected source/target Regions and measure its lag and
offset behavior. AWS Resilience Hub provides policy-based assessments and test
recommendations, not proof of actual live failover time or data loss. AWS replication
mechanisms do not remove the need for application consistency checks, writer fencing,
promotion authority, or business reconciliation.

