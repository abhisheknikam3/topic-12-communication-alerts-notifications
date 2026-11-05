# Topic 12: Communication, Alerts & Workflow Notifications

> **Institution:** Tayana Academy | LMS Assessment & Hands-On Guide  
> **Course:** n8n Workflow Automation — Level 2 (Intermediate)  
> **Topic / Assessment:** Topic 12: Communication, Alerts & Workflow Notifications  
> **Workflow Name:** `Topic 12 - Communication, Alerts & Workflow Notifications`  
> **Target Audience:** Site Reliability Engineers (SRE), DevOps & Incident Automation Specialists  
> **Difficulty Level:** Intermediate  
> **Platform Version:** n8n version 1.0.0 or higher  
> **Student:** Abhishek Nikam (`abhishek.n@tayanasolutions.com`)  
> **Repository:** [topic-12-communication-alerts-notifications](https://github.com/abhisheknikam3/topic-12-communication-alerts-notifications)  

---

## 1. Executive Summary & Objective

In modern mission-critical cloud and microservices infrastructures, incident alerting systems are plagued by alert fatigue and notification storms. When upstream dependencies (e.g., payment gateways or databases) experience outages, hundreds of repetitive error events burst across communication channels, overwhelming engineering response teams, causing critical signals to be lost in noise, and breaching rate limits on collaboration platforms like Slack.

### Architectural Solution
This assessment implements an enterprise-grade **Incident Classification, Throttling & Multi-Tier Notification Pipeline** in n8n. The workflow:
1. Ingests simulated system incidents with varying blast radiuses, environmental tiers, and error signatures.
2. Evaluates an **Incident Severity Matrix**:
   - **P1 Critical:** Impacting $>100$ users in `production` (routes to `#critical-incidents`).
   - **P2 Major:** Occurring in `production` with $\le 100$ users or non-zero staging impact (routes to `#dev-alerts`).
   - **P3 Minor:** Non-production warnings with zero impacted users (enqueued into daily digest).
3. Applies a **Stateful In-Memory Throttling Engine** (`Code_EvaluateSeverityAndThrottle`):
   - Generates composite throttling keys (`${service}:${errorSignature}`).
   - Enforces a 15-minute suppression window ($900,000\text{ ms}$). Repeated alerts within the window are quarantined to `Log_SuppressedAlert`, terminating duplicate messages before dispatch.
4. Constructs rich interactive **Slack Block Kit** card payloads for active P1 alerts and structured markdown cards for P2 alerts.
5. Dispatches messages across channel targets via a verified HTTP mock receiver (`HTTP_PostToMockSlack`), while staging low-priority alerts into `Code_StageInDigestQueue`.
6. Consolidates notification delivery telemetry into an immutable audit record (`Code_AuditNotificationDelivery`).

---

## 2. Canvas Architecture & Workflow Diagram

```
                 [ Manual_Trigger ]
                         │
                         ▼
             [ Code_SimulateIncidents ]
             (3 Incidents: P1 Storm, Repeat, P3 Warning)
                         │
                         ▼
        [ Code_EvaluateSeverityAndThrottle ]
        (Severity Matrix & 15-Min Throttling Cache)
                         │
                         ▼
                  [ IF_IsThrottled ]
                         │
        ┌────────────────┴────────────────┐
 [ True (Active Alert) ]          [ False (Throttled) ]
        │                                 │
        ▼                                 ▼
[ Switch_RouteBySeverity ]      [ Log_SuppressedAlert ]
        │                       (Quarantine Repeated Storm)
   ┌────┼────────────────────────┐
P1 │ P2 │                     P3 │
   ▼    ▼                        ▼
[ Code_BuildP1BlockKit ]   [ Code_StageInDigestQueue ]
[ Code_BuildP2SlackCard ]                │
   │                             │
   ▼                             │
[ HTTP_PostToMockSlack ]         │
(Deliver to Slack Endpoint)      │
   │                             │
   └──────────────┬──────────────┘
                  │
                  ▼
   [ Code_AuditNotificationDelivery ]
   (Master Delivery Audit Summary)
```

---

## 3. Incident Severity Matrix & Throttling Rules

### Severity Classification Logic

| Severity Level | Environmental Scope | User Impact Criteria | Target Destination | Payload Type |
| :---: | :---: | :---: | :---: | :--- |
| **`P1_CRITICAL`** | `production` | $> 100$ impacted users | `#critical-incidents` | Interactive Slack Block Kit Card |
| **`P2_MAJOR`** | `production` or staging | $\le 100$ users / any impact | `#dev-alerts` | Structured Slack Markdown Attachment |
| **`P3_MINOR`** | `staging` / development | $0$ impacted users | `digest_queue` | Enqueued for Daily Batch Digest |

---

### Executed Incident Scenario Trace

| Incident ID | Service | Error Signature | Severity Tier | Throttling Status | Dispatch Action |
| :---: | :--- | :--- | :---: | :---: | :--- |
| **`INC-901`** | `payment-gateway` | `AUTH_TIMEOUT_504` | `P1_CRITICAL` | `shouldAlert: true` | Dispatches Block Kit Card to `#critical-incidents` |
| **`INC-902`** | `payment-gateway` | `AUTH_TIMEOUT_504` | `P1_CRITICAL` | **`isThrottled: true`** | **SUPPRESSED** ➔ Quarantined to `Log_SuppressedAlert` |
| **`INC-903`** | `search-indexer` | `HEAP_MEMORY_WARN_85` | `P3_MINOR` | `shouldAlert: true` | Staged to `Code_StageInDigestQueue` (No Live Alert) |

---

## 4. Node-by-Node Settings Blueprint

| Node Name | Node Type | Critical Parameters & Settings | Architectural Role |
| :--- | :---: | :--- | :--- |
| **`Manual_Trigger`** | `manualTrigger` (v1) | On-demand execution trigger | Initiates execution of the incident testing suite. |
| **`Code_SimulateIncidents`** | `code` (v2) | Emits 3 incident cases (`INC-901`, `INC-902` duplicate storm, `INC-903`) | Simulates incoming telemetry alerts from APM/monitoring agents. |
| **`Code_EvaluateSeverityAndThrottle`** | `code` (v2) | Stateful 15-minute cache (`alertCache`), composite key evaluation | Core throttling & severity categorization engine. |
| **`IF_IsThrottled`** | `if` (v1) | Condition: `{{ $json.shouldAlert }} == true` | Security gate filtering out storm duplicates from live channels. |
| **`Log_SuppressedAlert`** | `code` (v2) | Formats `ALERT_SUPPRESSED_BY_THROTTLE_POLICY` audit item | Isolates suppressed alerts for SecOps observability without noise. |
| **`Switch_RouteBySeverity`** | `switch` (v3) | Rules: `P1_CRITICAL` (Out 0), `P2_MAJOR` (Out 1), `P3_MINOR` (Out 2) | Multi-channel dispatcher directing alerts to appropriate teams. |
| **`Code_BuildP1BlockKit`** | `code` (v2) | Builds interactive Slack Block Kit JSON with header, fields & blockquote | Formats executive P1 incident card for `#critical-incidents`. |
| **`Code_BuildP2SlackCard`** | `code` (v2) | Builds color-coded `#f59e0b` Slack attachment card | Formats engineering alert for `#dev-alerts`. |
| **`Code_StageInDigestQueue`** | `code` (v2) | Enqueues warnings with `digestStatus: "QUEUED_FOR_DIGEST"` | Batches minor warnings to prevent channel clutter. |
| **`HTTP_PostToMockSlack`** | `httpRequest` (v4.2) | `POST /mock/notifications/slack`, body: `{{ $json.dispatchPayload }}` | Delivers formatted message to Slack webhook endpoint. |
| **`Code_AuditNotificationDelivery`** | `code` (v2) | Formats final `NOTIFICATION_DISPATCHED` delivery verification | Central operational audit record confirming notification state. |

---

## 5. Verification Checkpoints & Testing Guide

### Test 1: Full Throttling & Routing Verification
1. **P1 Critical Dispatch (`INC-901`):**
   - Service: `payment-gateway`, 240 impacted production users.
   - Evaluated as `P1_CRITICAL`, `shouldAlert: true`.
   - Formatted into interactive Block Kit card and posted to `#critical-incidents`.
2. **Duplicate Alert Storm Suppression (`INC-902`):**
   - Identical key `payment-gateway:AUTH_TIMEOUT_504` received within 15-minute window.
   - Evaluated as `isThrottled: true`, `shouldAlert: false`.
   - Diverts to `Log_SuppressedAlert`; zero duplicate messages dispatched to Slack.
3. **P3 Minor Warning Staging (`INC-903`):**
   - Staging memory alert with 0 impacted users.
   - Evaluated as `P3_MINOR`.
   - Enqueued into `Code_StageInDigestQueue`; live channels remain completely silent.
4. **Delivery Audit (`Code_AuditNotificationDelivery`):**
   - Confirms `DELIVERED` status for `#critical-incidents` and `QUEUED` status for digest items.

---

## 6. Production Risks & Mitigation Strategies

| Production Risk | Operational Impact | Architectural Mitigation |
| :--- | :--- | :--- |
| **Unbounded Cache Memory Growth** | High variety of transient error signatures exhausts in-memory heap over time. | Enforce TTL pruning inside `alertCache` or offload state to an external Redis cluster with automated key expiration. |
| **Slack Block Kit JSON Syntax Errors** | Malformed JSON in `blocks` array triggers HTTP 400 Bad Request rejections from Slack. | Validate Block Kit structures against Slack's Block Kit schema validator prior to deployment. |
| **Critical Incident Escalation Under Cooldown** | An incident escalates in severity while still under an active cooldown window. | Invalidate active cooldown timer if `impactedUsers` increases by $>50\%$ or severity changes from P2 to P1. |

---

## 7. LMS Submission Notes & Assessor Text

```text
Title: Topic 12: Communication, Alerts & Workflow Notifications
Repository: https://github.com/abhisheknikam3/topic-12-communication-alerts-notifications
Student: Abhishek Nikam (abhishek.n@tayanasolutions.com)

Assessment Deliverables Summary:
1. Workflow JSON: Topic_12_Communication_Alerts_Notifications.json
   - 11 nodes fully configured: Incident Telemetry Simulator, Severity Classification & 15-Minute Throttling Cache Engine, Throttling Decision Gate, Suppressed Storm Alert Logger, 3-Way Severity Switch Router (P1 Critical, P2 Major, P3 Minor), Interactive Slack Block Kit Builder, Standard Slack Markdown Formatter, Digest Staging Queue, Mock Slack HTTP Dispatcher, and Master Notification Delivery Auditor.
   - Pre-pinned execution test data proving suppression of duplicate alert storms, Block Kit construction, and multi-channel routing.
2. Comprehensive Documentation: README.md & README.pdf
   - Business problem statement, ASCII & Mermaid architecture diagrams, severity classification matrices, test verification traces, and production risk mitigations (memory growth pruning, Block Kit syntax validation, cooldown invalidation).
```
