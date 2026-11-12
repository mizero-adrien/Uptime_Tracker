# Uptime Tracker: Data Flow Diagrams

| | |
|---|---|
| **Version** | 0.1 (draft) |
| **Status** | For review |
| **Diagram format** | Mermaid (renders in VS Code with a Mermaid preview extension, GitHub, and most Markdown tools) |
| **Companion docs** | `app_concept.md`, `schema.md`, `api_contract.md` |
| **Next docs** | Security and reliability plan, infrastructure plan, `openapi.yaml` |

---

## 1. Purpose and scope

This document shows **how data moves** through Uptime Tracker: where it enters, which process transforms it, where it is stored, and who consumes it. It is the bridge between the data model (`schema.md`), the API (`api_contract.md`), and the code that will be built.

It covers:

- A context diagram and a system-level data flow diagram (DFD)
- Detailed flows for each major process: checking, detection, incidents, alerting, ingestion, status pages, reports, webhooks, authentication
- State machines for the entities whose status changes over time
- Where each kind of data lives, how long it is kept, and what happens when parts fail
- Timing targets for detection and alerting

Every flow names the schema tables and API endpoints involved, so a developer can trace a feature from user action to database row.

---

## 2. Notation and conventions

| Symbol (Mermaid) | Meaning |
|---|---|
| Rectangle `["..."]` | External entity (a person or a system outside Uptime Tracker) |
| Rounded `("...")` | A process that transforms data |
| Cylinder `[("...")]` | A data store |
| Arrow with label | A data flow, labelled with **what** moves |
| Circle `(("..."))` | The whole system (context diagram only) |

Conventions used throughout:

- **D1 to D6** name the data stores (section 4). **P1 to P9** name the main processes.
- Times are UTC. "Round" means one scheduled check tick across all assigned regions for one platform.
- Domain events travel on the **event stream** (NATS or Kafka, decision pending). State changes in PostgreSQL publish events through a **transactional outbox**, so a database change and its event can never disagree (section 5.13).
- A flow that crosses a trust boundary (internet to us, or us to the internet) is marked in the security notes under the diagram.

---

## 3. Level 0: Context diagram

The whole system as one box, with everything it exchanges with the outside world.

```mermaid
flowchart LR
    U["Consultancy users<br/>owners, admins, engineers, viewers"]
    CV["Client viewers and<br/>status page subscribers"]
    MP["Monitored platforms<br/>websites, APIs, services"]
    CS["Client servers and agents<br/>log shippers, error SDKs"]
    ET["External tools<br/>Grafana, Prometheus, Sentry, PagerDuty"]
    NP["Notification providers<br/>email, SMS, WhatsApp, Slack"]
    PP["Payment provider"]
    CP["Cloud providers<br/>AWS, Azure, Google Cloud"]

    UT(("Uptime Tracker"))

    U -->|"configuration: clients, platforms, rules, schedules"| UT
    UT -->|"dashboards, incidents, reports, alerts"| U

    UT -->|"health checks: HTTP, TCP, DNS, SSL, browser"| MP
    MP -->|"responses, timings, certificates"| UT

    CS -->|"logs, error events, heartbeats"| UT
    ET -->|"alert and incident webhooks"| UT
    UT -->|"queries and outbound events"| ET
    UT -->|"read logs and health feeds"| CP
    CP -->|"logs, service health"| UT

    UT -->|"messages to send"| NP
    NP -->|"delivery receipts"| UT

    UT -->|"public status pages, incident updates"| CV
    CV -->|"subscribe, unsubscribe"| UT

    UT -->|"subscription and usage events"| PP
    PP -->|"payment status, plan changes"| UT
```

**Trust boundaries crossed:** every arrow touching an external entity. Inbound data is authenticated (keys, tokens, signatures) and validated before it reaches any process. Outbound requests to monitored platforms and webhook targets are restricted to public addresses (see `api_contract.md` section 15.2).

---

## 4. Level 1: System data flow diagram

### 4.1 Processes and data stores

| ID | Name | Responsibility | Main tables or interfaces |
|---|---|---|---|
| **P1** | Configuration management | Accounts, clients, platforms, rules, channels, schedules | Management API, `organizations` through `alert_rules` |
| **P2** | Scheduling and probing | Decide when checks are due, run them from regions | `platforms`, `platform_regions`, probe fleet |
| **P3** | Evaluation and detection | Turn raw results into state changes and anomalies | `platform_state`, `no_data_periods` |
| **P4** | Incident management | Open, update, resolve incidents | `incidents`, `incident_platforms`, `incident_updates` |
| **P5** | Alerting and notification | Match rules, escalate, send, record delivery | `alert_rules`, `alerts`, `notifications`, `escalation_*`, `on_call_*` |
| **P6** | Client data ingestion | Receive, mask, parse, store logs and errors | Ingestion API, `ingestion_sources`, `error_groups` |
| **P7** | Publishing and reporting | Status pages, subscriber updates, SLA reports | `status_pages`, `reports`, `report_schedules` |
| **P8** | Integrations | Inbound and outbound connections to other tools | `integrations`, `integration_mappings`, webhooks |
| **P9** | Billing and limits | Plan enforcement, usage metering | `subscriptions`, `usage_records` |

| ID | Store | Technology | Holds |
|---|---|---|---|
| **D1** | Configuration database | PostgreSQL | All business data (`schema.md`) |
| **D2** | Check results | TimescaleDB hypertable | `check_results`, `check_results_daily` |
| **D3** | Log and event store | ClickHouse or Loki | Raw logs and error events |
| **D4** | Event stream | NATS or Kafka | In-flight check results, events, jobs |
| **D5** | Cache and counters | Redis | Auth lookups, rate limits, heartbeat deadlines, public status cache |
| **D6** | Object storage | S3-compatible | Report PDFs, exports |

### 4.2 Diagram

```mermaid
flowchart TB
    subgraph EXT["External entities"]
        U["Consultancy users"]
        MP["Monitored platforms"]
        CS["Client agents and tools"]
        NP["Notification providers"]
        CV["Client viewers"]
    end

    subgraph CTRL["Control plane"]
        P1("P1 Configuration management")
        P7("P7 Publishing and reporting")
        P9("P9 Billing and limits")
        P8("P8 Integrations")
    end

    subgraph DATA["Data plane"]
        P2("P2 Scheduling and probing")
        P3("P3 Evaluation and detection")
        P4("P4 Incident management")
        P5("P5 Alerting and notification")
        P6("P6 Client data ingestion")
    end

    D1[("D1 PostgreSQL<br/>configuration and state")]
    D2[("D2 TimescaleDB<br/>check results")]
    D3[("D3 Log store")]
    D4[("D4 Event stream")]
    D5[("D5 Redis")]
    D6[("D6 Object storage")]

    U -->|"configuration requests"| P1
    P1 <-->|"read and write config"| D1
    P1 -->|"limits check"| P9
    P9 <-->|"plan, usage"| D1

    D1 -->|"active platforms and settings"| P2
    P2 -->|"probe requests"| MP
    MP -->|"responses"| P2
    P2 -->|"check results"| D4
    D4 -->|"check results"| P3
    P3 -->|"raw results"| D2
    P3 <-->|"read thresholds, write state"| D1
    P3 -->|"status change events"| D4

    D4 -->|"status change events"| P4
    P4 <-->|"incidents and timeline"| D1
    P4 -->|"incident events"| D4

    D4 -->|"incident and anomaly events"| P5
    P5 <-->|"rules, alerts, notifications"| D1
    P5 -->|"messages"| NP
    NP -->|"delivery receipts"| P5

    CS -->|"logs, errors, heartbeats"| P6
    P6 -->|"normalised events"| D3
    P6 -->|"error spikes, silence events"| D4
    P6 <-->|"source keys, error groups"| D1
    P6 <-->|"key cache, deadlines"| D5

    D4 -->|"status and incident events"| P7
    P7 -->|"public view"| D5
    D5 -->|"cached public status"| CV
    P7 -->|"subscriber emails"| P5
    D1 -->|"incidents, maintenance, summaries"| P7
    D2 -->|"daily aggregates"| P7
    P7 -->|"report PDFs"| D6

    CS -->|"webhooks"| P8
    P8 -->|"translated events"| D4
    D4 -->|"domain events"| P8
    P8 -->|"signed webhooks"| CS
    P8 <-->|"credentials, mappings"| D1

    D2 -->|"charts and metrics"| P1
    D3 -->|"log search"| P1
    D4 -->|"live stream events"| P1
```

---

## 5. Level 2: Detailed flows

### 5.1 Check pipeline (P2)

How a check goes from "due" to a stored result.

```mermaid
sequenceDiagram
    autonumber
    participant SCH as Scheduler
    participant D1 as PostgreSQL
    participant D4 as Event stream
    participant PR as Probe in region
    participant MP as Monitored platform
    participant EV as Evaluator
    participant D2 as TimescaleDB

    SCH->>D1: Load active platforms and settings (cached, refreshed on change events)
    loop Every tick, per platform and region
        SCH->>D4: Publish "check due" job with platform id, region, tick time
    end
    D4->>PR: Deliver job to a probe in that region
    PR->>MP: Run the check (HTTP, TCP, DNS, SSL, keyword)
    alt Reply received
        MP-->>PR: Status code, body excerpt, timings
    else Timeout or connection error
        MP--xPR: No reply
    end
    PR->>PR: Classify outcome (error category, assertions)
    PR->>D4: Publish check result
    D4->>EV: Deliver result
    EV->>D2: Insert raw result (batched)
    EV->>EV: Add to the round for this platform and tick
```

**Notes**

- **Scheduling is spread out.** Each platform gets a stable random offset within its interval, so thousands of checks do not all fire at the same second.
- **Jobs are idempotent.** The job carries a tick time. If a probe crashes and the job is redelivered, a duplicate result is discarded using the primary key `(platform_id, probe_region_id, checked_at)`.
- **Configuration secrets** (such as authentication headers) are delivered to the probe encrypted for that probe only, and never written to logs.
- **Safety.** The probe resolves the target and rejects private, loopback, link-local, and cloud-metadata addresses before connecting, and checks again at connect time.
- **Tables touched:** `platforms`, `platform_regions` (read), `check_results` (write).

### 5.2 Evaluation and detection logic (P3)

This is the most important logic in the product. It decides when a platform counts as down.

```mermaid
flowchart TD
    A["Check results arrive from probes"] --> B["Group results by platform and tick into a round"]
    B --> C{"Did enough regions report<br/>before the collection window closed?"}
    C -- "No" --> D["Mark round inconclusive"]
    D --> D2["Record probe gap, leave failure counters unchanged"]
    C -- "Yes" --> E{"Failing regions at least<br/>region quorum?"}
    E -- "Yes" --> F["Round FAILED<br/>failures plus 1, successes reset to 0"]
    E -- "No" --> G["Round PASSED<br/>successes plus 1, failures reset to 0"]

    F --> F1{"Status is up or degraded<br/>and failures at least failure threshold?"}
    F1 -- "Yes" --> F2{"Platform in a<br/>maintenance window?"}
    F2 -- "Yes" --> F3["Stay in maintenance, record only"]
    F2 -- "No" --> F4["Status becomes DOWN<br/>open incident with started_at = first failed check of this streak"]
    F1 -- "No" --> F5["Request an immediate confirmation round<br/>if failures is below threshold"]

    G --> G1{"Status is down<br/>and successes at least recovery threshold?"}
    G1 -- "Yes" --> G2["Status becomes UP<br/>resolve incident, set resolved_at"]
    G1 -- "No" --> G3{"Response slower than<br/>slow threshold?"}
    G3 -- "Yes" --> G4["Status becomes DEGRADED<br/>after the same threshold logic"]
    G3 -- "No" --> G5["Status stays or becomes UP"]

    F4 --> H["Write platform_state and publish status change event<br/>in one transaction"]
    G2 --> H
    G4 --> H
    G5 --> H
```

**Rules that make detection trustworthy**

1. **A round, not a single check, is the unit.** One platform is checked from several regions per tick. The round fails only when **at least `region_quorum` regions** fail it. One region's network problem never counts as downtime.
2. **Missing results are not failures.** If fewer than `region_quorum` regions reported, the round is inconclusive and counters do not move. The gap is recorded, because a silent probe is our problem, not the client's.
3. **Confirmation rounds shorten detection.** After the first failed round, the evaluator asks the scheduler for immediate extra rounds, spaced about 10 to 15 seconds apart, instead of waiting for the next full interval. This keeps false-alarm protection (several rounds) without a multi-minute delay.
4. **`started_at` is honest.** An incident's `started_at` is the timestamp of the first failed check in the failing streak, not the moment it was confirmed. `detected_at` records confirmation.
5. **State change and event are atomic.** `platform_state` is updated, the incident is opened or closed, and the outbox event is written in **one database transaction**.
6. **Degraded** follows the same threshold logic, using `slow_threshold_ms`.
7. **Maintenance** suppresses incident creation and alerts. Checks still run and results are still stored.

**Tables touched:** `platform_state`, `incidents`, `incident_platforms`, `incident_updates`, `no_data_periods`, `outbox_events` (new, section 12).

### 5.3 Anomaly detection (P3, extension)

```mermaid
flowchart LR
    R["Hourly rollups of response time<br/>and error rate per platform"] --> BL["Baseline builder<br/>per hour of week"]
    BL --> ST[("Baseline store<br/>mean, spread, sample count")]
    CR["Live check rounds"] --> CMP{"Value far outside<br/>baseline for long enough?"}
    ST --> CMP
    CMP -- "No" --> OK["No action"]
    CMP -- "Yes" --> AN["Publish anomaly event<br/>type anomaly, platform, evidence"]
    AN --> P4X("Incident management")
```

Anomaly detection starts with statistical baselines (mean and spread by hour of week). A model-based version can replace the baseline builder later without changing anything downstream, because the output is the same anomaly event. **Needs a store for baselines** (section 12).

### 5.4 Incident lifecycle (P4)

```mermaid
sequenceDiagram
    autonumber
    participant EV as Evaluator
    participant D1 as PostgreSQL
    participant OB as Outbox relay
    participant D4 as Event stream
    participant AE as Alert engine
    participant UI as Dashboard
    participant ENG as Engineer

    EV->>D1: One transaction: update platform_state, insert incident, link platforms, add system update, write outbox event
    OB->>D1: Read unpublished outbox events
    OB->>D4: Publish incident.opened
    D4->>AE: Deliver incident.opened
    D4->>UI: Push through the live stream (SSE)
    ENG->>UI: Acknowledge incident and add a note
    UI->>D1: Set acknowledged_at, insert incident update
    Note over EV,D1: Later, recovery rounds pass the threshold
    EV->>D1: One transaction: platform_state up, incident resolved, resolved_at, system update, outbox event
    OB->>D4: Publish incident.resolved
    D4->>AE: Resolve linked alerts, send recovery notices
    D4->>UI: Push resolution
```

**Rules**

- Incident numbers come from `next_org_number()` inside the creating transaction, so they are gap-free per organization.
- A manual incident follows the same path from step 5 onward, with `source = manual`.
- If a platform fails again within the recovery window, the **same incident stays open** rather than creating a new one every minute.
- **Tables touched:** `platform_state`, `incidents`, `incident_platforms`, `incident_updates`, `organization_counters`.

### 5.5 Alerting and escalation (P5)

```mermaid
sequenceDiagram
    autonumber
    participant D4 as Event stream
    participant AE as Alert engine
    participant D1 as PostgreSQL
    participant ES as Escalation worker
    participant NS as Notification sender
    participant PV as Provider
    participant ENG as Engineer

    D4->>AE: incident.opened or anomaly event
    AE->>D1: Find matching rules (platform, client, tag, organization scope)
    loop Each matching rule
        AE->>AE: Skip if rule disabled, in cooldown, or platform in maintenance
        AE->>D1: Insert alert with dedup key (conflict means already firing)
        AE->>D1: Load escalation policy and first step
        AE->>AE: Resolve step target (user, on-call schedule now, or channel)
        AE->>D1: Insert queued notifications and set next_escalation_at
    end
    NS->>D1: Fetch queued notifications
    NS->>PV: Send message
    PV-->>NS: Accepted or error
    NS->>D1: Record sent, failed, or retry
    PV-->>NS: Delivery receipt (callback)
    NS->>D1: Record delivered or bounced
    loop Every few seconds
        ES->>D1: Find firing alerts whose next_escalation_at has passed
        ES->>D1: Move to next step, queue notifications, set new next_escalation_at
    end
    ENG->>D1: Acknowledge (dashboard, API, or signed link in the message)
    Note over ES,D1: Acknowledged alerts leave the firing set, so escalation stops
```

**Rules**

- **Deduplication.** The `alerts_active_dedup_uq` index makes a second identical alert fail to insert, so one outage produces one alert, not fifty.
- **Failure of a channel is not failure of alerting.** If sending to one channel fails after retries, the escalation proceeds to the next target immediately instead of waiting out the delay.
- **Fallback provider.** SMS and WhatsApp use at least two providers where possible. A provider outage switches to the backup and is itself logged.
- **Recovery notices** are sent when the incident resolves, to everyone who was notified.
- **Quiet hours and severity** are applied while resolving targets. A critical alert ignores quiet hours.
- **Who is on call** is a lookup in `on_call_shifts` at the time of escalation, so schedule changes take effect immediately.
- **Tables touched:** `alert_rules`, `alerts`, `notifications`, `escalation_policies`, `escalation_steps`, `on_call_schedules`, `on_call_shifts`, `notification_channels`, `maintenance_windows`.

### 5.6 Client log and error ingestion (P6)

```mermaid
flowchart TD
    A["Agent or shipper sends a batch<br/>POST /ingest/logs with X-Ingest-Key"] --> B["Ingestion gateway"]
    B --> C{"Key valid?<br/>lookup by prefix in Redis, fall back to PostgreSQL,<br/>compare hash in constant time"}
    C -- "No" --> C1["401, count the failure per IP"]
    C -- "Yes" --> D{"Within plan rate and size limits?"}
    D -- "No" --> D1["429 with Retry-After"]
    D -- "Yes" --> E["Validate each event<br/>timestamp, size, required fields"]
    E --> E1["Reject bad events individually,<br/>report in the response"]
    E --> F["Mask sensitive data<br/>using the source masking rules"]
    F --> G["Publish accepted events to the stream<br/>and answer 202"]
    G --> H["Parser and normaliser<br/>common event shape, tenant ids attached"]
    H --> I["Batch writer"]
    I --> D3[("D3 Log store")]
    H --> J["Error fingerprinting"]
    J --> K["Upsert error group<br/>count, first seen, last seen"]
    K --> D1[("D1 error_groups")]
    H --> L["Rate counters<br/>errors per minute per platform"]
    L --> M{"Spike above threshold?"}
    M -- "Yes" --> N["Publish error_spike event"]
    N --> P4X("Incident management and alerting")
    B --> O["Update last_received_at in Redis"]
    O --> P["Silence detector<br/>source quiet longer than expected interval"]
    P --> Q["Mark source silent, publish ingestion event"]
```

**Notes**

- **Acknowledge early, process later.** The gateway answers `202` once events are safely on the stream. Searchable storage follows within seconds. This keeps the ingestion surface fast under heavy load.
- **Masking happens before anything is stored or queued beyond the gateway**, so sensitive values never reach D3 or the error samples.
- **Backpressure.** If the stream or writer falls behind, the gateway answers `429` or `503`. Agents retry with backoff. Data is never dropped silently.
- **Tenancy.** The key identifies the source, the source identifies the organization, client, and platform. Clients cannot choose their own tenant ids.
- **Tables and stores touched:** `ingestion_sources`, `error_groups`, D3, D5.

### 5.7 Inbound webhooks from other tools (P8)

```mermaid
flowchart TD
    A["External tool sends a webhook<br/>POST /webhooks/grafana/{token}"] --> B["Ingestion gateway"]
    B --> C{"Token valid and<br/>signature valid where the tool supports it?"}
    C -- "No" --> C1["401"]
    C -- "Yes" --> D["Adapter for the tool<br/>translate into the common event shape"]
    D --> E["Find the platform through integration_mappings"]
    E --> F{"Mapping found?"}
    F -- "No" --> F1["Store against the client, flag unmapped<br/>show in the dashboard for mapping"]
    F -- "Yes" --> G{"Event says firing or resolved?"}
    G -- "Firing" --> H["Create or attach an incident<br/>source = integration, link external_ref"]
    G -- "Resolved" --> I["Resolve the linked incident or alert"]
    H --> J["Publish event, alert engine takes over"]
    I --> J
```

**Notes:** An unmapped event is never discarded. Duplicate deliveries are recognised through the external reference and ignored. **Tables touched:** `integrations`, `integration_mappings`, `incidents`, `alerts`.

### 5.8 Heartbeat (cron job) monitoring (P2 and P6)

```mermaid
sequenceDiagram
    autonumber
    participant JOB as Client scheduled job
    participant GW as Ingestion gateway
    participant RD as Redis
    participant SW as Deadline sweeper
    participant EV as Evaluator

    JOB->>GW: GET /heartbeat/{token} (optionally status=fail)
    GW->>RD: Reset deadline to now plus expected interval plus grace
    GW-->>JOB: 200 with next expected time
    Note over RD,SW: If no check-in arrives before the deadline
    SW->>RD: Find expired deadlines
    SW->>EV: Publish a synthetic failed result for that platform
    EV->>EV: Normal round evaluation, incident type heartbeat_missed
    JOB->>GW: Next check-in (after outage)
    GW->>EV: Publish a synthetic passing result, recovery logic runs
```

A check-in with `status=fail` is treated as an immediate failed result. Deadlines live in Redis for speed and are rebuilt from `platforms` after a Redis restart, so a cache loss never silences monitors.

### 5.9 Maintenance windows

```mermaid
flowchart TD
    A["Engineer schedules a window<br/>POST /maintenance-windows"] --> B["Store window and platform links"]
    B --> C["Publish maintenance.scheduled event"]
    C --> D["Status page shows upcoming maintenance"]
    B --> E["Timer fires at starts_at"]
    E --> F["Set platform_state to maintenance<br/>suppress alerts for matching rules"]
    F --> G["Checks keep running, results stored<br/>no incidents opened"]
    G --> H["Timer fires at ends_at, or window ended early"]
    H --> I["Restore normal status from the next round"]
    I --> J{"Platform failing<br/>when the window ends?"}
    J -- "Yes" --> K["Normal detection takes over,<br/>incident opens as usual"]
    J -- "No" --> L["Status returns to up"]
```

Time spent in a window is excluded from official uptime (see `schema.md` section 10), so planned work never counts against an SLA. A window can never be created in the past.

### 5.10 Status page publication (P7)

```mermaid
flowchart LR
    EVT["Events: platform status, incident, maintenance"] --> PUB["Status publisher"]
    PUB --> FILT{"Component on a public page<br/>and incident marked public?"}
    FILT -- "No" --> X["Nothing published"]
    FILT -- "Yes" --> PROJ["Build the public view<br/>component statuses, public incidents, public updates"]
    PROJ --> D5[("Redis cache and CDN")]
    D5 --> API["GET /public/status/slug"]
    API --> CV["Client viewers"]
    PUB --> SUB["Subscriber fan-out<br/>incident created, updated, resolved"]
    SUB --> NS("Notification sender")
    NS --> EMAIL["Subscriber emails"]
```

**Notes**

- The public view is a **projection**: it contains only fields deliberately marked public. Internal notes, root causes, and error details are never copied into it.
- The cache is purged when a status or incident event arrives, and also expires after 30 seconds, so a missed purge self-heals.
- Subscriber emails use double opt-in and a signed unsubscribe link.
- **Tables touched:** `status_pages`, `status_page_components`, `status_page_subscribers`, `incidents`, `incident_updates`.

### 5.11 Report generation (P7)

```mermaid
sequenceDiagram
    autonumber
    participant TRG as Scheduler or user
    participant JOB as Job queue
    participant GEN as Report generator
    participant D1 as PostgreSQL
    participant D2 as TimescaleDB
    participant D6 as Object storage
    participant NS as Notification sender

    TRG->>JOB: Create report job (client, period)
    JOB->>GEN: Deliver job
    GEN->>GEN: Check the period has fully ended
    GEN->>D1: Read incidents, maintenance windows, no-data periods for the period
    GEN->>D2: Read daily aggregates for charts
    GEN->>GEN: Compute official uptime and SLA result
    GEN->>D1: Insert frozen report row (numbers cannot change afterwards)
    GEN->>GEN: Render PDF with client branding
    GEN->>D6: Store PDF
    GEN->>D1: Save file reference, mark report ready
    GEN->>NS: Queue delivery to client contacts
    NS->>D1: Record notifications
```

Official uptime follows the formula in `schema.md` section 10 (incident downtime, minus approved maintenance and no-data time). The generator never recomputes a delivered report. A correction creates a new report that references the old one.

### 5.12 Outgoing webhooks (P8)

```mermaid
sequenceDiagram
    autonumber
    participant D1 as PostgreSQL
    participant OB as Outbox relay
    participant D4 as Event stream
    participant WD as Webhook dispatcher
    participant EP as Customer endpoint

    D1->>OB: Domain event written in the same transaction as the change
    OB->>D4: Publish event and mark published
    D4->>WD: Deliver event
    WD->>D1: Find enabled endpoints subscribed to this event type
    loop Each endpoint
        WD->>WD: Build payload and sign with HMAC (timestamp plus body)
        WD->>EP: POST within 10 seconds
        alt 2xx response
            EP-->>WD: Success
            WD->>D1: Record delivered
        else Error or timeout
            EP--xWD: Failure
            WD->>D1: Record attempt, schedule retry (1m, 5m, 30m, 2h, 6h, 12h)
        end
    end
```

### 5.13 Transactional outbox (how events stay consistent)

```mermaid
flowchart LR
    A["Application change"] --> T["Database transaction"]
    T --> S["Update business tables"]
    T --> O["Insert row into outbox_events"]
    S --> C["Commit together"]
    O --> C
    C --> R["Relay polls or listens for new outbox rows"]
    R --> D4[("Event stream")]
    R --> M["Mark row published"]
    D4 --> CON["Consumers: alert engine, status publisher,<br/>webhook dispatcher, live stream"]
```

Consumers must be **idempotent**: the relay delivers at least once. Each event carries a unique id, and consumers keep a short record of ids they have handled.

### 5.14 Authentication and sessions

```mermaid
sequenceDiagram
    autonumber
    participant B as Browser
    participant API as Management API
    participant D1 as PostgreSQL
    participant RD as Redis

    B->>API: POST /auth/login (email, password)
    API->>RD: Check login rate limit
    API->>D1: Load user, verify password hash
    alt MFA enabled
        API-->>B: mfa_required with a short-lived challenge token
        B->>API: POST /auth/mfa/verify (challenge, code)
    end
    API->>D1: Create session and refresh token family
    API-->>B: Access token in the body, refresh token in an HttpOnly cookie
    Note over B,API: Access token expires after 15 minutes
    B->>API: POST /auth/refresh (cookie)
    API->>D1: Verify refresh token and rotate it
    alt Token already used before
        API->>D1: Revoke the whole session family
        API-->>B: 401, sign in again
    else Valid
        API-->>B: New access token and new refresh cookie
    end
```

**Tables touched:** `users`, `sessions` (new), `auth_tokens` (new), `organization_members`.

### 5.15 Tenant-isolated request (every Management API call)

```mermaid
flowchart TD
    A["Request with Authorization header"] --> B["Authenticate<br/>verify JWT or hash the API key"]
    B --> C{"Valid?"}
    C -- "No" --> C1["401"]
    C -- "Yes" --> D["Read org id from the path"]
    D --> E{"Caller is a member of that organization<br/>or key belongs to it?"}
    E -- "No" --> E1["404 so that ids cannot be probed"]
    E -- "Yes" --> F{"Role or scope allows this action?"}
    F -- "No" --> F1["403"]
    F -- "Yes" --> G["Begin transaction<br/>SET LOCAL app.current_org"]
    G --> H["Run queries<br/>row-level security filters every table"]
    H --> I["Write audit log entry for changes"]
    I --> J["Commit and respond"]
```

Two independent checks (the membership check and row-level security) must both fail before data could leak. This is deliberate.

### 5.16 Onboarding a client platform (end to end)

```mermaid
sequenceDiagram
    autonumber
    participant ENG as Engineer
    participant UI as Dashboard
    participant API as Management API
    participant D1 as PostgreSQL
    participant SCH as Scheduler
    participant PR as Probes
    participant EV as Evaluator

    ENG->>UI: Add client "ABC Bank"
    UI->>API: POST /clients
    API->>D1: Insert client
    ENG->>UI: Add platform (URL, regions, thresholds, tags)
    UI->>API: POST /platforms
    API->>API: Validate target is public and plan allows another platform
    API->>D1: Insert platform, regions, platform_state (unknown)
    API->>D1: Write outbox event platform.created
    SCH->>D1: Pick up the new platform through the change event
    SCH->>PR: First checks within one interval
    PR->>EV: Results
    EV->>D1: platform_state becomes up
    UI-->>ENG: Live status appears through the SSE stream
    ENG->>UI: Attach alert rule and escalation policy
    UI->>API: POST /alert-rules
```

### 5.17 Watching the watcher (our own reliability)

```mermaid
flowchart TD
    subgraph OURS["Uptime Tracker"]
        PRB["Probes in each region"]
        REG["Probe registry"]
        SCH["Scheduler"]
        EV["Evaluator"]
        CAN["Canary platforms<br/>known-good and known-bad test targets"]
    end

    subgraph INDEP["Independent of our infrastructure"]
        EXT["External uptime monitor<br/>different provider and region"]
        DMS["Dead man's switch service"]
    end

    PRB -->|"heartbeat every 15 seconds"| REG
    REG -->|"probe silent for 45 seconds"| RA["Mark region degraded<br/>recalculate quorum, page our on-call"]
    REG -->|"no probes alive"| NO["Write no_data_periods for affected platforms<br/>never record as up"]
    SCH -->|"I am alive ping every minute"| DMS
    DMS -->|"ping missing"| PAGE["Page the platform team by SMS and phone"]
    EXT -->|"checks public API and status pages"| PAGE
    CAN --> EV
    EV -->|"canary result unexpected"| PAGE
```

**Why this exists:** a monitoring tool that fails silently is worse than none. Three layers cover three failure kinds: a probe region dying (registry), our whole control plane dying (dead man's switch), and a logic bug producing wrong results (canaries with known answers). Our own paging path must not depend on the systems being watched.

---

## 6. State machines

### 6.1 Platform status

```mermaid
stateDiagram-v2
    [*] --> unknown: platform created
    unknown --> up: first rounds pass
    unknown --> down: failures reach threshold
    up --> degraded: slow beyond threshold
    up --> down: failures reach threshold
    degraded --> up: back under slow threshold
    degraded --> down: failures reach threshold
    down --> up: successes reach recovery threshold
    up --> maintenance: window starts
    degraded --> maintenance: window starts
    down --> maintenance: window starts
    maintenance --> unknown: window ends, state re-evaluated
    up --> paused: user pauses
    degraded --> paused: user pauses
    down --> paused: user pauses
    unknown --> paused: user pauses
    paused --> unknown: user resumes
    paused --> [*]: platform deleted
    up --> [*]: platform deleted
    down --> [*]: platform deleted
```

### 6.2 Incident status

```mermaid
stateDiagram-v2
    [*] --> investigating: opened by system or person
    investigating --> identified: cause found
    identified --> monitoring: fix applied
    investigating --> monitoring: recovery seen
    monitoring --> resolved: recovery confirmed
    identified --> resolved: recovery confirmed
    investigating --> resolved: recovery confirmed
    resolved --> investigating: reopened within 24 hours
```

Status only moves forward, except through `reopen`. `resolved` always sets `resolved_at`, and nothing else may (enforced by a database CHECK).

### 6.3 Alert status

```mermaid
stateDiagram-v2
    [*] --> firing: rule matched and not duplicate
    firing --> acknowledged: person acknowledges
    firing --> resolved: incident resolves
    acknowledged --> resolved: incident resolves or person resolves
    firing --> suppressed: maintenance begins
    suppressed --> firing: maintenance ends and problem persists
    suppressed --> resolved: problem gone
    resolved --> [*]
```

### 6.4 Notification status

```mermaid
stateDiagram-v2
    [*] --> queued: created by alert engine
    queued --> sent: provider accepted
    queued --> failed: retries exhausted
    sent --> delivered: receipt received
    sent --> bounced: bounce or invalid destination
    sent --> failed: no receipt and provider error
    failed --> [*]
    delivered --> [*]
    bounced --> [*]
```

A `failed` or `bounced` notification triggers immediate escalation to the next step and raises an internal warning when it affects a critical alert.

### 6.5 Ingestion source status

```mermaid
stateDiagram-v2
    [*] --> waiting: source created
    waiting --> connected: first valid data received
    connected --> silent: nothing for longer than expected interval
    silent --> connected: data resumes
    connected --> disabled: user disables
    silent --> disabled: user disables
    waiting --> disabled: user disables
    disabled --> waiting: user re-enables
```

---

## 7. Where data lives

| Data | Written by | Store | Read by | Retention |
|---|---|---|---|---|
| Accounts, clients, platforms, rules | P1 | D1 | All processes | Until deleted, then purged after the grace period |
| Current platform status | P3 | D1 (`platform_state`) | Dashboard, P5, P7 | Always current |
| Raw check results | P3 | D2 | Dashboard charts, P7 | 30 to 365 days by plan, compressed after 7 |
| Daily summaries | D2 aggregation job | D2 | Reports, long charts | Indefinitely |
| Incidents and timelines | P4 | D1 | Dashboard, P5, P7 | Indefinitely |
| Alerts and notifications | P5 | D1 | Dashboard, audit | 2 years |
| Logs and error events | P6 | D3 | Dashboard log search | 7 to 30 days by plan |
| Grouped errors | P6 | D1 | Dashboard | Until resolved, then 90 days |
| In-flight events and jobs | P2, P3, P4, P6, P8 | D4 | Consumers | Hours to days (replay window) |
| Heartbeat deadlines, rate limits, key cache | P6, gateway | D5 | P6, gateway | Seconds to minutes. Rebuildable from D1 |
| Public status views | P7 | D5 and CDN | Public API | 30 seconds |
| Report files | P7 | D6 | Dashboard, email | Indefinitely |
| Audit trail | P1 and all writers | D1 | Admins | 2 years or more |
| Secrets (keys, credentials) | P1, P8 | D1, encrypted or hashed | Only the process that needs them | Until rotated or deleted |

**Rule of thumb:** anything that cannot be rebuilt (configuration, incidents, reports, audit) lives in D1, D2, or D6 with backups. Anything rebuildable (caches, deadlines, projections) lives in D5. Anything high-volume and disposable (logs) lives in D3 with a short TTL.

---

## 8. Failure and backpressure behaviour

| Failure | Immediate effect | Designed response | How it is detected |
|---|---|---|---|
| One probe machine dies | Some checks missed | Other probes in the region take over. Duplicates are discarded | Registry heartbeat |
| A whole probe region dies | Rounds have fewer results | Region excluded, quorum recalculated, rounds below quorum are inconclusive and recorded as gaps | Registry heartbeat |
| All probes down | No new results | `no_data_periods` written, never counted as up. Internal page | Dead man's switch, canaries |
| Event stream slow or down | Results queue up | Probes buffer locally and retry. Gateway answers `503` with `Retry-After` for ingestion | Stream lag metrics |
| Evaluator crashes mid-round | Round not applied | Consumer group redelivers. Evaluation is idempotent per round | Consumer lag, restarts |
| PostgreSQL primary fails | Config writes fail | Automatic failover to replica. Probes keep running from cached config. Evaluator pauses state changes and resumes | Health checks, replication monitoring |
| TimescaleDB write path slow | Result inserts back up | Batched writes with a bounded buffer. State evaluation continues from the stream, not from the stored rows | Insert latency metrics |
| Redis lost | Caches and deadlines gone | Rebuilt from D1. Auth falls back to the database. Heartbeat deadlines recomputed | Redis health |
| Notification provider down | Messages fail | Retry, switch to backup provider, escalate to the next step | Delivery failure rate |
| All notification channels fail | Nobody is told | Page through the independent path (dead man's switch service) and show a banner in the dashboard | Notification failure alarms |
| Log store down | Logs not searchable | Ingestion keeps accepting into the stream (bounded). Log search shows a clear "temporarily unavailable" state | Writer errors |
| Outbox relay down | Downstream not notified | Rows stay in the outbox. On restart, delivery resumes. Nothing is lost | Outbox age metric |
| Duplicate event delivery | Possible double action | Idempotent consumers using event ids | Duplicate counters |
| Clock skew on a probe | Misordered timestamps | Probes sync time and the evaluator uses its own receive time as a sanity bound. Results far from receive time are flagged | Skew metric |

**Principle:** under every failure, the system prefers saying "we do not know" (a recorded gap) over saying "up".

---

## 9. Timing targets

| Measure | Target | How it is achieved |
|---|---|---|
| Check scheduling accuracy | Within 1 second of the planned tick | Distributed scheduler, per-region job queues |
| Real outage to confirmed incident | Under 60 seconds at a 30-second interval, under 2 minutes at a 60-second interval | Several regions per tick plus fast confirmation rounds |
| Confirmed incident to first notification sent | Under 10 seconds | Event stream to alert engine to sender, no polling on the hot path |
| Notification sent to provider acceptance | Under 5 seconds | Pre-warmed provider connections |
| Escalation timing error | Under 15 seconds | Worker polls the due-alerts index every few seconds |
| Dashboard status update after a change | Under 3 seconds | Live stream (SSE) |
| Public status page update | Under 30 seconds | Cache purge on event, 30-second expiry |
| Ingested log searchable | Under 30 seconds | Small writer batches |
| Heartbeat miss to incident | Expected interval plus grace plus one round | Deadline sweeper |
| Recovery detection | Recovery threshold times the check interval, or faster with confirmation rounds | Same round logic as failure |

These are design targets. They become service level objectives for Uptime Tracker itself once measured, and are reported on our own status page.

---

## 10. Traceability: flows to schema and API

| Flow | Main tables | Main endpoints |
|---|---|---|
| 5.1 Check pipeline | `platforms`, `platform_regions`, `check_results` | `POST /platforms`, `GET /platforms/{id}/check-results` |
| 5.2 Detection | `platform_state`, `incidents`, `no_data_periods` | `GET /platforms/{id}/state`, `GET /incidents` |
| 5.4 Incident lifecycle | `incidents`, `incident_platforms`, `incident_updates` | `/incidents/*` |
| 5.5 Alerting | `alert_rules`, `alerts`, `notifications`, `escalation_*`, `on_call_*` | `/alert-rules`, `/alerts`, `/escalation-policies`, `/on-call/*` |
| 5.6 Log ingestion | `ingestion_sources`, `error_groups`, D3 | `POST /ingest/logs`, `/ingestion-sources`, `/error-groups` |
| 5.7 Inbound webhooks | `integrations`, `integration_mappings` | `/webhooks/*/{token}`, `/integrations` |
| 5.8 Heartbeats | `platforms` (heartbeat type), D5 | `GET /heartbeat/{token}` |
| 5.9 Maintenance | `maintenance_windows`, `maintenance_window_platforms` | `/maintenance-windows` |
| 5.10 Status pages | `status_pages`, `status_page_components`, `status_page_subscribers` | `/public/status/*`, `/status-pages` |
| 5.11 Reports | `reports`, `report_schedules`, `jobs` (new) | `/reports`, `/jobs/{id}` |
| 5.12 Outgoing webhooks | `webhook_endpoints`, `webhook_deliveries` (new) | `/webhook-endpoints` |
| 5.14 Authentication | `users`, `sessions` (new), `auth_tokens` (new) | `/auth/*`, `/me` |
| 5.15 Tenant isolation | all tenant tables (RLS) | every Management API endpoint |

---

## 11. Assumptions made in these flows

1. Probes run in separate regions and hosting providers from the control plane.
2. Evaluation works on **rounds** (all regions for one tick), which the schema already supports through `region_quorum` and the per-platform thresholds.
3. An event stream (NATS or Kafka) sits between probes, evaluator, and downstream consumers. Direct database polling is used only for schedules and escalation timers.
4. State changes publish events through a transactional outbox.
5. During maintenance, incidents are **not** opened and alerts are suppressed, but checks continue and results are stored.
6. Confirmation rounds after a first failure are allowed to run faster than the platform's normal interval.

---

## 12. Schema additions required by these flows

These extend the list in `api_contract.md` section 18.

| # | Addition | Needed for |
|---|---|---|
| 1 | `outbox_events` (id, organization_id, type, payload, created_at, published_at) with an index on unpublished rows | Transactional outbox (5.13) |
| 2 | `processed_events` or a Redis-based dedup record for consumers | Idempotent consumers (5.13) |
| 3 | `platform_baselines` (platform_id, hour_of_week, mean, spread, sample_count, updated_at) | Anomaly detection (5.3) |
| 4 | `probe_instances` (id, region_id, status, last_heartbeat_at, version) | Probe registry (5.17) |
| 5 | `canary_platforms` (platform_id, expected_status, owner) or a flag on `platforms` | Canary monitors (5.17) |
| 6 | `round_evaluations` (optional audit table: platform_id, tick, regions_reported, regions_failed, outcome) | Debugging "why did this open an incident?", short retention |
| 7 | `platforms.confirmation_interval_seconds` (nullable override) | Confirmation round spacing (5.2) |

---

## 13. Open decisions

| # | Decision | Options | Needed by |
|---|---|---|---|
| 1 | Event stream technology | NATS (simple, light) or Kafka (heavy, durable, large scale) | Stage 2 |
| 2 | Scheduler design | Single leader with failover, or sharded schedulers by platform hash | Stage 2 |
| 3 | Where evaluation runs | One stateless evaluator fleet reading the stream, or per-shard evaluators | Stage 2 |
| 4 | Whether round evaluations are persisted | Always, sampled, or only around incidents | Stage 3 |
| 5 | Confirmation round spacing and cap | 10 to 15 seconds, up to the failure threshold | Stage 2 |
| 6 | Whether maintenance should still record incidents silently | No (as drafted), or record with a maintenance flag | Stage 3 |
| 7 | SSE vs WebSocket for the live dashboard | SSE (as drafted) | Stage 4 |
| 8 | Log store | ClickHouse or Loki | Stage 5 |
| 9 | Dead man's switch and external monitor providers | Choose two independent providers | Stage 2 |
| 10 | Anomaly detection starting method | Seasonal baseline, then evaluate models | Stage 7 |
| 11 | Public status page hosting | Same API behind a CDN, or a separate static publisher | Stage 4 |

---

## 14. Change log

| Version | Date | Change |
|---|---|---|
| 0.1 | 2026-11-12 | First full draft |