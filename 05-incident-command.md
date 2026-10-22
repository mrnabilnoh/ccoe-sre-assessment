# 5. Incident command

See [05-incident-command-flow.png](./diagrams/05-incident-command-flow.png).

**At a glance:** Severity follows customer impact and financial risk. The IC coordinates;
journey and service owners retain accountability. Operations or Technical Support can
record decisions and evidence. Use scheduled updates, explicit escalation, and accepted
shift handovers to preserve continuity.

| Operating rule | Approach |
|---|---|
| Severity | Customer impact, safety, and blast radius, not alert count |
| Coordination | IC sets objectives; journey/service owners remain accountable |
| Communication | Scheduled updates plus immediate updates for material changes |
| Evidence | Lightweight UTC decision/evidence log during response; blameless RCA after recovery |
| Handover | Incoming IC explicitly accepts command |

## 5.1 Severity matrix

| Severity | Impact guide | Response | Proposed update cadence |
|---|---|---|---|
| SEV1 | Widespread critical-journey failure, materially wrong balances, credible financial-integrity risk, major regional impact, or rapid SLO burn. | Immediate IC, bridge, journey owner, technical leads, Business/Risk. | Every 15 minutes; continuous bridge. |
| SEV2 | Significant journey/segment degradation, provider impairment, or fast burn with safe workaround and no confirmed integrity breach. | IC or delegate coordinates named service owners. | At least every 30 minutes. |
| SEV3 | Limited/intermittent impact or contained issue; no current financial-integrity evidence. | Owning team manages with incident record and journey-owner visibility. | Agreed checkpoints. |
| SEV4 | No current customer impact; preventive defect or improvement. | Ticket with owner and priority. | Normal workflow. |

Severity reflects customer impact, safety, and blast radius, not the number of red
component alerts. A clear balance-unavailable response is safer than an unverified stale
value, but affected valid requests still count as customer impact. The IC may change
severity with a recorded reason. Calibrate numeric scope/duration thresholds with
Product/Risk using baselines and incident history.

The 15-minute SEV1 and 30-minute SEV2 cadences are proposed internal targets, not
regulatory requirements; confirm bank policy and notification obligations. Send an
immediate update for material changes such as failover, impact, recovery milestones, or
new customer guidance. Do not wait for the next scheduled update.

## 5.2 Roles

| Role | Responsibility |
|---|---|
| Incident Commander | Set objectives, priority, authority, and next checkpoint; coordinate response. |
| Operations Lead | Execute mitigation, rollback, traffic control, and runbooks. |
| Technical Leads | Represent EKS, database, streaming, security, and providers. |
| Communications Lead | Prepare approved internal, customer, executive, and regulatory updates without sensitive data. |
| Incident recorder | Operations/Technical Support assigned by IC maintains UTC timeline and decision/evidence log. IC records directly if support is unavailable. |
| Business/Risk Lead | Confirm payment safety, customer impact, and reporting duties. |

The IC delegates investigation but remains accountable for coordination. Roles are not
automatically assigned to the team that owns the failing component. The journey owner
owns the end-to-end customer outcome; service owners diagnose components; the
provider-relationship owner handles external escalation. Populate actual team names,
primary/backup contacts, and paths from the service catalog and on-call directory.

## 5.3 Operating cadence and decision log

The bridge is the live coordination channel; the recorder maintains the incident record.
Each update covers:

- Current customer impact and severity.
- Confirmed facts versus hypotheses, and what changed.
- Current action, accountable owner, safe customer guidance, and next checkpoint.

Customer and regulatory messages go through designated Communications and Business/Risk
roles using approved wording. Do not publish unverified causes.

Use a short decision log:

| Time UTC | Decision / hypothesis | Evidence | Accountable owner | Review/rollback condition |
|---|---|---|---|---|
| `HH:MM` | Traffic or data action | Dashboard, trace/request ID, runbook/query | Named role | Condition and time |

Capture only what supports immediate decisions; this is not the full RCA. Record:

| Category | Examples |
|---|---|
| Impact and context | Incident ID, affected CUJ, customer-safe impact counts, Region, release, severity, confirmed facts/hypotheses |
| Data and dependencies | Aurora writer/replication state; MSK replication, lag, and offsets; provider status |
| Actions and telemetry | ARC routing; SSM execution history; OTel trace IDs; CloudFront request IDs; CloudWatch, Application Signals, Synthetics, and CloudTrail evidence |

Link to approved telemetry instead of copying payloads. Use least privilege; exclude
credentials, full account identifiers, and payment payloads. Complete the blameless RCA
after containment and recovery using preserved evidence.

## 5.4 Escalation and handover

| Trigger | Escalation |
|---|---|
| Affected service/provider | Accountable on-call owner; IC confirms the owner accepts the escalation. |
| Regional recovery or data promotion | DR Recovery Authority. |
| Possible RPO exception, payment ambiguity, or material financial risk | Business/Risk. |
| Security indicator or potential notification threshold | Security or designated Compliance contact. |

Escalate immediately. Notification deadlines follow bank policy and regulatory
obligations.

Handover is required when an incident outlasts an on-call shift, including within the
same team. The outgoing IC briefs the incoming IC and updates the record with:

- Impact/severity, timeline, confirmed facts/hypotheses, and actions/results.
- System, data, and ARC state; unresolved risks and decisions.
- Each action's owner/contact, communications sent, and next checkpoint.

The incoming IC explicitly accepts command. The recorder updates ownership and handover
time. Do not leave command implicit during shift change.

## 5.5 Closure and learning

Close only when all conditions hold:

- Customer impact is resolved and data is reconciled.
- Monitoring is normal and the product owner accepts residual risk.
- Blameless review actions each have one accountable owner, due date, success measure,
  and retest evidence.

Route recurring toil to the roadmap rather than permanently absorbing it in SRE.

