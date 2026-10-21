# 1. Reliability model

See [01-cuj-dependency-map.png](./diagrams/01-cuj-dependency-map.png).

**At a glance:** Measure availability separately for each critical journey. The current
three journeys are candidates because the assessment does not provide a confirmed
inventory. Confirm scope, success conditions, latency objectives, and owners before
baselining SLOs.

## 1.1 Critical user journeys

The assessment gives no journey inventory. These three journeys are the working scope
for the assessment, not a claim that they are the service's complete catalog. Confirm
journeys, variants, owners, and objectives with Product, Operations, Risk, and service
teams before baselining SLOs.

| Journey | Customer success condition | Main dependency path |
|---|---|---|
| Sign in | Valid customer receives a usable authenticated session after required checks. | CloudFront/WAF -> API Gateway -> EKS Identity service; conditional fraud/risk check. |
| View balance | Customer receives an accurate balance. If accuracy/freshness is uncertain, show unavailable rather than stale data as current. | CloudFront/WAF -> API Gateway -> EKS Account/Balance service -> Aurora. |
| Make a payment | Customer receives accepted, rejected, or pending status; pending payments resolve without duplicate financial effect. | CloudFront/WAF -> API Gateway -> EKS Payment service -> Aurora ledger -> MSK consumer -> payment provider. |

Discovery confirms channels, account/payment variants, the actual identity and provider
flows, freshness/latency expectations, and one accountable owner per journey. Split a
variant into its own SLO only when outcome, criticality, or dependency behavior differs
materially. Use service catalogs, customer flows, API/event maps, incidents, and owner
interviews as evidence.

Availability is measured separately for each CUJ from eligible customer requests.
Independent synthetic probes are a separate detection and coverage signal; they do not
change the request-based SLI denominator. Invalid input and sign-in attempts using
invalid credentials are expected business outcomes, not service failures. Valid
customer requests affected by provider outages or planned maintenance remain customer
impact unless an explicit, approved SLO policy defines an exclusion.

## 1.2 Failure mode and effects analysis

| Failure domain | Main journey impact |
|---|---|
| Edge, API, EKS/AZ, or bad release | Can affect one or several journeys before or within application services. |
| Sign-in risk provider | May block legitimate sign-in or leave risk undecided; use approved fallback. |
| Aurora / ledger | Balance may be unavailable; payment is not accepted without a durable ledger record. |
| MSK / payment provider | Payment may remain pending or unknown; reconcile before retry and protect event ordering. |
| Regional loss or retry amplification | Can affect multiple journeys and exhaust shared capacity; confirm evidence and recover under authority. |

Expand the detailed matrix for detection, response, and ownership by failure mode.

<details>
<summary>Detailed failure mode analysis</summary>

| Failure mode | Customer or business effect | Detection evidence | Response boundary | Accountable owner |
|---|---|---|---|---|
| Identity service unavailable | Customers with valid credentials cannot sign in. | Sign-in SLI and synthetic checks fail; identity-service errors or latency rise. | Restore identity service; do not bypass authentication to improve availability. | Identity service owner; Sign-in journey owner coordinates customer impact. |
| Fraud/risk provider unavailable during sign-in | Legitimate customers may be blocked, or account risk may be accepted without an assessment. | Risk-provider timeout/error rate and sign-in outcome by risk path. | Apply a Product- and Security-approved risk-based fallback. Use an independent step-up check where approved; block or defer elevated-risk sign-ins if a safe decision is unavailable. | Sign-in journey owner with Security/Product; provider relationship owner escalates externally. |
| Edge/API impairment or misconfiguration | One or more journeys fail before reaching their application service. | Journey synthetics, CloudFront/WAF and API Gateway errors, regional health evidence. | Correct configuration or use the approved regional recovery path. Preserve WAF protections. | Platform; affected journey owner coordinates customer impact. |
| Aurora unavailable or balance read path impaired | Balance cannot be verified; payment writes may fail or have an ambiguous result. | Balance SLI, database events, query latency/errors, durable-write result. | For balance, show a clear maintenance/unavailable state when current accuracy cannot be confirmed. For payment, do not claim acceptance without a durable ledger record. | Database and relevant service owner; journey owner accountable for customer outcome. |
| Payment ledger write fails | The system has no durable proof that the payment was accepted. | Ledger write errors and payment submission outcome. | Stop before external execution or an accepted response. Permit a safe retry using the same idempotency key after recovery. | Payments and database owners; Make-a-payment journey owner coordinates. |
| MSK unavailable or consumer lag after ledger write | A durable payment can remain pending while downstream processing or status updates are delayed. | MSK broker health, consumer lag, pending age, and payment resolution SLI. | Preserve the ledger record and pending state. Resume processing after recovery; verify the event recovery mechanism and prevent lost or duplicate processing. | Streaming and Payments owners; journey owner coordinates. |
| Payment provider timeout after possible receipt | The transaction result is unknown; a blind retry could duplicate a payment. | Provider timeout/error metrics, unresolved pending age, status inquiry and reconciliation results. | Return a safe pending outcome, retain transaction state, and resolve through provider inquiry or asynchronous reconciliation before any retry. | Payments journey owner; provider relationship owner manages provider escalation. |
| Retry amplification | Increased requests can saturate services, connection pools, brokers, or providers and spread a local issue. | Retry count per original request, timeout/503 rate, concurrency, throttling, regional health, and CUJ SLI. The cause of the two reported regional incidents remains unconfirmed until records are reviewed. | Task 1 records the failure and evidence. Retry budgets, backoff, jitter, circuit breakers, and load shedding are specified in Task 3. | Participating service owners; journey owner coordinates, Platform investigates shared saturation. |
| EKS/AZ capacity loss or bad release | Multiple journeys may become slow or unavailable. | Journey SLI, target health, Container Insights, deployment/change correlation. | Restore capacity or roll back through the approved change process; detailed controls are covered in platform and resilience design. | Platform or releasing service team; affected journey owner coordinates. |
| Regional loss | Primary-region journeys are unavailable or impaired. | Independent journey probes, regional health evidence, and recovery telemetry. | Authorized recovery process promotes data and shifts traffic after readiness checks; ARC provides routing controls, not transaction validation. | Named recovery authority; journey owners validate customer outcomes. |

</details>

## 1.3 SLI and SLO specification

Each critical CUJ has its own rolling 365-day 99.99% availability SLO. The formal
request-based error budget is 0.01% of eligible customer requests in that window.
For a 365-day year, the time-based equivalent is 52.56 minutes, or 4.38 minutes per
month on average. Those minutes are an intuitive equivalent, not a separate monthly
allowance for the request-based SLI. Actual budget burn depends on the number and rate
of affected requests. The SLI is:

`CUJ availability = good eligible customer requests / total eligible customer requests`

Good means the response reaches the intended customer outcome correctly, is usable,
and meets the latency and freshness objectives approved for that journey. The result
is calculated independently per CUJ and segmented by Region, AZ, operation, provider,
release, and channel. A passing synthetic probe cannot mask failing customer requests,
and one healthy CUJ cannot mask another CUJ's SLO breach.

The journey inventory and response-time objectives remain assumptions for this
assessment. Product and Risk must confirm them against customer expectations, risk
policy, and measured service baselines before the SLOs are baselined. The scenario
specifies no response-time thresholds, so none are asserted here as regulatory
requirements.

| Measure | Target or definition | Evidence / open decision |
|---|---:|---|
| Sign-in availability | 99.99% per eligible request | Valid credentials reach a usable session. Confirm latency and risk-fallback rules. |
| Balance availability | 99.99% per eligible request | Correct, sufficiently fresh balance. Unavailable counts as impact; confirm latency/freshness. |
| Payment submission availability | 99.99% per eligible request | Accepted/rejected/pending response; acceptance requires durable ledger record. |
| Payment resolution | Separate measure; window to be agreed | Pending transactions reach final state without duplicate effects. |
| Critical-data RPO | <= 5 minutes | Measure acknowledged durable transactions missing after promotion; confirm data scope. |
| Regional RTO | <= 45 minutes | Authorized declaration to validated journeys; also report customer interruption. |

Payment correctness, duplicate payment rate, unresolved pending age, and reconciliation
completion are safety indicators alongside availability. A safe pending submission
may meet the submission SLI while the separate resolution measure remains outstanding.
The 45-minute RTO is a recovery objective, not an availability allowance. Report
customer impact duration and regional restoration time separately. Confirm the
critical data set, transaction acknowledgement semantics, recovery clock start, and
treatment of in-flight or ambiguous payments with business and risk owners.

## 1.4 Error-budget policy

Each journey owner reviews its own rolling-year request budget with Product and the
participating service teams. Short-window fast burn triggers incident response; shorter
windows are operational alerts, not replacements for the annual SLO. Sustained burn
adds reliability work to the affected team's priorities. When a journey exhausts its
budget, pause high-risk discretionary changes affecting that journey until its owner
and Product agree on a recovery plan. Security, regulatory, and urgent customer-safety
work continues. Record exceptions and the decision owner.

## 1.5 Ownership and evidence

Each journey has one accountable owner for its end-to-end customer outcome. Product
teams own behavior and reliability within their services; Platform owns shared
foundations such as EKS, networking, ARC controls, scaling, deployment safety, and
shared telemetry. Database and streaming teams own Aurora/MSK durability and recovery
evidence. Security owns IAM, KMS, WAF, CloudTrail, Config, and forensic access. For
payments, the Payments journey owner remains accountable for the customer outcome;
internal service owners handle their services, and the provider relationship owner
escalates against the external provider's commitments. The incident commander
coordinates response and does not inherit permanent ownership of underlying defects.

Validation evidence includes monthly SLO reports, dependency contract tests, controlled
failure tests, capacity tests, Aurora/MSK recovery measurements, and quarterly game-day
reports. Known limitations are replication lag, provider contract uncertainty, AWS
regional feature differences, and the need to validate the critical data set size.
