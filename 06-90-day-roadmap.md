# 6. 90-day reliability roadmap

See [06-90-day-roadmap-gantt.png](./diagrams/06-90-day-roadmap-gantt.png).

**At a glance:** Sequence the 90 days around risk reduction: establish evidence and
owners, address the highest customer and financial risks, then rehearse recovery and
review residual risk. Product and Platform capacity must be committed; SRE enables the
work without inheriting unowned operations.

| Days 1–30 | Days 31–60 | Days 61–90 |
|---|---|---|
| Baseline ownership; close payment safety gaps; improve journey detection | Implement bounded resilience; prepare regional recovery dependencies | Rehearse failover/failback, measure outcomes, and close critical findings |

## 6.1 Prioritization method

Present this roadmap as an investment and risk-acceptance discussion. For each item,
state the risk if delayed, evidence available, accountable owner, capacity needed, proof
of risk reduction, and residual risk. Prioritize customer and financial safety first,
then consider likelihood, blast radius, detection/recovery gaps, recurring toil,
dependencies, and delivery effort. Do not invent probabilities, savings, or capacity
figures. Where evidence is missing, make baseline collection an early deliverable.

The sequence below is a proposed plan, not a commitment that all work fits without
confirmed team capacity. Before leadership approves it as funded, Product and Platform
leaders must assign named capacity, estimate effort using their planning method, and
agree which feature work moves if reliability work is prioritized. Those estimates are
currently open because team capacity and delivery baselines were not provided in the
assessment scenario.

## 6.2 Backlog

### Days 1–30: Establish baseline and ownership

- **Risk if delayed:** Journey failures, frequency, and ownership remain unclear; priorities rely on assumptions.
- **Investment and owner:** Confirm journey inventory, SLI/SLOs, error budgets, incident baseline, and owner map. Product owns journey definitions; service teams own service data; SRE facilitates.
- **Capacity and dependencies:** Product, service-owner, Operations, and SRE time. Leaders confirm availability; collect incident, toil, and traffic data.
- **Evidence:** Agreed journey/SLO/owner catalog; baseline for customer impact, MTTD, toil, and capacity; prioritized risk register.
- **Residual risk and decision:** Incident probability, cost, and capacity remain uncertain until data quality is assessed. Leadership confirms initial allocation.

### Days 1–30: Close payment and dependency safety gaps

- **Risk if delayed:** Provider timeouts, retry amplification, or weak idempotency may create unresolved or duplicate financial effects.
- **Investment and owner:** Inventory deadlines, retry ownership, provider contracts, idempotency, ledger invariants, and reconciliation gaps. Payments owns service behavior; Platform owns shared controls.
- **Capacity and dependencies:** Payments, Database, Platform, provider relationship, and Risk input. Validate provider status-query/idempotency support.
- **Evidence:** Reviewed dependency contracts and retry matrix; documented ledger-backed idempotency and unknown-outcome handling; owners for each gap.
- **Residual risk and decision:** Thresholds and provider guarantees depend on contract confirmation and production baselines.

### Days 1–30: Improve actionable journey detection

- **Risk if delayed:** Customer-impacting failure may stay hidden among component alerts, extending impact and diagnosis toil.
- **Investment and owner:** Prioritize OTel/ADOT, Application Signals, CloudWatch service metrics, CloudFront request-ID correlation, safe Synthetics, and alert routing. Observability/Platform enables; service teams instrument journeys.
- **Capacity and dependencies:** Observability and service-team time; verify AWS regional support, service limits, and telemetry cost.
- **Evidence:** Trace continuity and CloudFront correlation tests; sign-in, balance, and payment-preflight canaries; sample routed alert.
- **Residual risk and decision:** CloudFront is correlated through edge telemetry, not represented as an OTel span. Settlement relies on real-user and business-outcome signals.

### Days 31–60: Implement bounded resilience

- **Risk if delayed:** Sustained provider or service degradation may amplify into a wider outage or overwhelm critical payment capacity.
- **Investment and owner:** Test deadlines, bounded jittered retries, application breakers, per-provider concurrency limits, backpressure, safe load shedding, and Kafka consumer recovery. Product/service teams implement behavior; Platform provides shared guidance.
- **Capacity and dependencies:** Payments, service teams, Platform, and Streaming. Library choices depend on supported languages and operations. Istio is optional and needs separate platform-cost approval.
- **Evidence:** Staging tests for timeouts, retry amplification, breakers, Redis loss, ledger failure, MSK lag/quarantine, and overload; documented results and fixes.
- **Residual risk and decision:** Numeric thresholds require baselines and provider commitments. Staging does not prove production behavior.

### Days 31–60: Prepare recovery dependencies

- **Risk if delayed:** Malaysia may lack capacity, access, replication state, or provider connectivity during regional impairment.
- **Investment and owner:** Validate warm capacity, Aurora promotion/fencing, MSK topics/offsets, IAM/KMS, secrets, network, provider allowlists, SSM approvals, and ARC controls. Recovery Authority owns readiness; Platform, Database, and Streaming own components.
- **Capacity and dependencies:** Platform, Database, Streaming, Security, Payments, Risk, and provider relationship owners. Confirm regional support, quotas, cost, and data residency.
- **Evidence:** Reviewed failover/failback runbooks; permission and dependency checks; tested approval gates; assigned readiness gaps.
- **Residual risk and decision:** Resilience Hub is planning evidence, not proof of actual RTO/RPO. Replication lag and consistency remain exercise-dependent.

### Days 61–90: Rehearse and measure

- **Risk if delayed:** Recovery remains unproven; leadership cannot assess RTO/RPO feasibility or remaining gaps.
- **Investment and owner:** Run mandatory representative staging/UAT failover and failback; close critical findings. A production exercise requires explicit approval and safeguards. IC/Recovery Authority coordinate; service owners execute; Risk/Product approve scope.
- **Capacity and dependencies:** Cross-team exercise time, staging fidelity, and an approved production window if pursued.
- **Evidence:** Signed scorecard with detection/declaration, approval/promotion timestamps, measured RPO/RTO, single-writer evidence, payment reconciliation, and retested actions.
- **Residual risk and decision:** Staging does not prove production objectives. Production exercise and customer-impact tolerance require separate approval.

## 6.3 Metrics

Track a small set of decision-oriented measures:

| Measure | Leadership question it answers |
|---|---|
| Per-journey availability and error-budget burn | Are customers receiving the critical outcomes we committed to? |
| MTTD, time to mitigate, and repeat-incident rate | Are detection and response improving, and are the same failures recurring? |
| Payment unknown/pending age, duplicate indicators, reconciliation backlog | Is financial ambiguity being contained and resolved safely? |
| Retry amplification, MSK lag/oldest-event age, breaker events | Are dependencies or queues approaching overload? |
| DR exercise RTO/RPO evidence and open findings | Can the recovery design meet targets, and what risk remains? |
| Operational toil hours, recurring manual tasks, action closure | Is toil falling, and are corrective investments completing? |
| Named ownership coverage and committed-versus-delivered capacity | Are teams accountable and is the funded plan achievable? |

Collect baselines during Days 1–30. Set numeric targets with Product, Platform, Risk,
and service owners after reviewing data quality and risk tolerance. Review trends monthly;
revisit ownership and residual risk quarterly.

## 6.4 Governance guardrails

Product teams own their journey and service behavior, data correctness, provider
contracts, SLOs, and service remediation. Platform teams own shared foundations,
platform reliability, and reusable capabilities. SRE provides standards, coaching,
shared enablement, and validation; it does not become the default owner of service
operations or unassigned technical debt.

Reliability work enters the backlog with a named service owner, customer/financial risk,
evidence, expected toil reduction, dependencies, capacity estimate, validation plan, and
residual-risk owner. Product and Platform leaders approve the capacity trade-off and
identify feature work that must move. SRE engagement has a defined scope and exit owner;
repeated operational requests become owned backlog work rather than permanent SRE
support.

Review the roadmap monthly with Product and Platform leaders. Review residual risk,
error-budget exceptions, toil trends, game-day findings, cost, and overdue actions
quarterly with Risk/Security as appropriate. Emergency reliability work is time-boxed,
has an accountable service owner, and returns to the backlog with follow-up capacity.

## 6.5 Trade-offs

The roadmap defers broad active/active behavior until consistency, reconciliation, and
game-day evidence justify its added cost and operational complexity. It invests early
in customer-impact detection and payment safety because delay leaves visibility and
financial-integrity risks unresolved. Managed AWS services can reduce undifferentiated
operations, while adding AWS dependency, quota, regional-support, and cost considerations.
Where incident probability, financial impact, or toil is not yet quantified, record the
unknown, collect baseline evidence, and present the residual risk for leadership
acceptance rather than manufacture a business case number.

