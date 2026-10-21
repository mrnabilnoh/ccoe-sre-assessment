# 2. Observability architecture

See [02-xray-observability-flow.png](./diagrams/02-xray-observability-flow.png).

**At a glance:** Instrument applications with AWS-supported OpenTelemetry (ADOT), use
Application Signals for health and service maps, and use AWS trace tools for
investigation. CloudFront is correlated through edge telemetry and request IDs, not
represented as an application span. Canaries provide early warning; payment completion
is monitored through real-user and business signals.

| Need | Design choice |
|---|---|
| Instrument and collect | OTel with ADOT or the CloudWatch agent/OTel Collector |
| Detect customer impact | Request-based journey SLIs plus separate CloudWatch Synthetics checks |
| Find the failing layer | Application Signals, X-Ray trace investigation, and native AWS service telemetry |
| Protect sensitive data | Source minimization/redaction, CloudWatch Logs data protection, least privilege |

## 2.1 Instrumentation and AWS observability services

**Instrumentation choice:** Use vendor-neutral OTel in application services, collected
with AWS Distro for OpenTelemetry (ADOT) or the CloudWatch agent/OTel Collector. This
reduces instrumentation lock-in. A future backend change may still require exporter,
dashboard, alert, sampling, and AWS-specific feature changes.

**AWS support boundary:** X-Ray SDKs and daemon entered maintenance mode on February 25,
2026, with security fixes only. New application instrumentation uses OTel. The AWS
X-Ray service remains supported and accepts OTel traces. Use Application Signals for
service health, SLOs, and application maps; use the CloudWatch X-Ray Trace Map and
Transaction Search to inspect traces. The former ServiceLens map experience is in the
current CloudWatch trace-map experience.

**Managed-service signals complement traces:**
- CloudFront: distribution metrics, access logs, `Server-Timing`, edge request ID.
- WAF/API Gateway: native metrics and logs; REST API stages can optionally emit managed
  X-Ray traces.
- EKS: Container Insights for node, pod, workload, and saturation signals.
- MSK: broker health and consumer-lag metrics.
- Aurora: CloudWatch metrics and Database Insights for load, waits, and query hotspots.

These signals expose service-specific behavior that application spans alone do not.
Availability of the selected observability services in Singapore
(`ap-southeast-1`) and Malaysia (`ap-southeast-5`) is an assumption to validate.

## 2.2 Trace context and edge correlation

- Use W3C Trace Context (`traceparent`, optionally `tracestate`) across OTel services.
- At ingress, validate or replace external trace context and sampling flags. Keep
  credentials, account identifiers, and payment data out of headers and span attributes.
- Start the trace in the customer client or first instrumented application component.
  Forward context through CloudFront to the origin using explicit configuration.
- CloudFront does not emit an OTel span here. Correlate its edge logs/timing to the
  downstream trace with `X-Amz-Cf-Id` / edge request ID and the ingress trace ID.
- API Gateway REST tracing is an optional managed integration. Test whether it joins
  the W3C trace; if not, correlate by request ID and document that boundary.
- Carry W3C context in MSK headers and create linked consumer spans for asynchronous
  work. Correlate database spans with Database Insights; do not assume Aurora creates
  application spans automatically.

## 2.3 Telemetry standards and privacy

- Use consistent service, environment, Region, release, journey, operation, and
  dependency attributes across spans and logs.
- Structured logs include timestamp, trace/request ID, result class, and retry count as
  relevant. Never log raw idempotency keys.
- Keep metric dimensions bounded. Exclude customer identifiers to control cardinality
  and privacy risk.
- Minimize/redact sensitive fields at source. CloudWatch Logs data-protection policies
  add detection and masking for configured identifiers. `logs:Unmask` is restricted and
  audited. This does not automatically mask traces or metrics, and it is not retroactive
  for events ingested before the policy is enabled.

## 2.4 Journey checks and business signals

**Synthetic checks:** Schedule non-destructive canaries in both Regions using a dedicated
sign-in identity and read-only balance checks. Payment canaries stop at preflight and do
not prove settlement. Full synthetic payment requires explicit Product, Risk, Finance,
and provider approval. Canary results are early-warning signals, not customer SLI data;
payment failures can still first appear in real-user/business signals.

**Business signals in Application Signals and CloudWatch:**

- Sign-in: valid-attempt outcome by approved risk path.
- Balance: correctness, freshness, and unavailable responses.
- Payment: accepted/rejected/pending/unknown submission, resolution time, pending age,
  duplicate indicators, and unresolved outcomes.
- Backend: durable ledger failures, MSK event-to-status freshness, provider timeouts,
  throttling, status inquiries, and reconciliation backlog.

## 2.5 Alert routing

| Route | Trigger | Response |
|---|---|---|
| Page | Journey impact, fast SLO burn, ledger failure, duplicate effect, or rapidly rising unresolved payments. | Start incident response; notify journey owner. |
| Ticket/team alert | Slow burn, rising provider errors or MSK lag before impact, recurring toil, or capacity risk. | Assign owner and prioritize planned work. |
| Dashboard | Normal variation without impact or rising risk. | Review trends without paging. |

Set thresholds after baselining with Payments, Product, Risk, and service owners. Group
correlated alarms. Pages include journey, burn, Region, release, dependency, evidence
IDs, owner, runbook, and safe mitigation. CloudWatch alarms route to on-call; lower
severity creates tickets or team notifications.

## 2.6 Retention, access, and cost

Use purpose-specific retention rather than applying the draft's seven-year period to
all telemetry. The following is an illustrative policy; exact durations and archival
rules require approval from Legal, Risk, Security, and service owners.

| Data | Example policy | Purpose |
|---|---|---|
| Traces | Sample; increase capture for errors and incidents. | Limit volume/cost while preserving diagnostic value. |
| Metrics/SLO history | Retain for trend and seasonal review. | Support reliability and capacity decisions. |
| Application logs | Set per log group and purpose; redact sensitive fields. | Separate operations from regulated evidence. |
| Audit evidence | Store under the bank-approved schedule with access/integrity controls. | Meet approved legal and risk requirements. |
| Canary artifacts | Retain failure details only as long as needed. | Screenshots may contain sensitive content. |

Encrypt with approved KMS keys and least-privilege access. Review `logs:Unmask`, sampling,
retention, ingestion, cardinality, queries, canary frequency, and cost regularly.

## 2.7 Validation checklist

- [ ] Verify W3C context from client through API Gateway/EKS and MSK consumer links.
- [ ] Correlate CloudFront request ID with edge logs and downstream trace.
- [ ] Test managed API Gateway tracing and document whether it joins the OTel trace.
- [ ] Run sign-in, balance, and payment-preflight canaries; confirm payment cannot move funds.
- [ ] Simulate dependency failure and verify Application Signals, metrics, alert routing,
  owner, and runbook context.
- [ ] Test log masking and `logs:Unmask` access; confirm retention, deletion, encryption,
  and cost controls against approved policy.

