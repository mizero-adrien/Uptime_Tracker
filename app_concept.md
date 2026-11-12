# Uptime Tracker: App Concept

> Status: Draft v1
> Next documents: `schema.md` (database design), then the API contract

---

## 1. One-line summary

Uptime Tracker is a monitoring platform for **consultancy companies** that maintain many client websites and apps. It watches every client platform, records uptime, logs, and errors, and alerts the right people the moment something goes down or behaves abnormally.

---

## 2. Key terms (read these first)

| Term | Meaning |
|---|---|
| **Consultancy** | A company hired by other businesses to build, run, or maintain their systems. It is our paying customer. |
| **Client** | The consultancy's own customer (for example a bank or hospital). |
| **Platform / Monitor** | One website, API, or service that we watch. |
| **Uptime** | The share of time a platform is working. 99.9% uptime is about 43 minutes of downtime per month. |
| **Check (probe)** | One automatic test of a platform: "are you alive, and how fast do you answer?" |
| **Log** | A saved record of an event, with a timestamp. |
| **Anomaly** | The platform is up but acting abnormally, such as responding five times slower than usual. |
| **Incident** | One period of trouble, from when it starts until it is fixed. |
| **Alert / Notification** | A message (email, SMS, WhatsApp, Slack) sent when something is wrong. |
| **Escalation** | The rule for who is alerted first, and who next if nobody responds. |
| **SLA / SLO** | The reliability promise made to a client (for example 99.9%). Our reports prove whether it was kept. |
| **Multi-tenant** | One system serving many separate customers, each seeing only its own data. |
| **Integration** | A connection to another tool so they share data (Grafana, Prometheus, and others). |

---

## 3. The problem

A consultancy may be responsible for dozens of client platforms. When one fails at 2 a.m., the client blames the consultancy, and the team often learns about it from an angry phone call.

Today's options each fall short:

- **Simple tools (such as UptimeRobot)** check basic availability, but are not organised around clients, and offer limited client reporting.
- **Enterprise platforms (such as Datadog)** are powerful but complex and expensive, and are built for large engineering teams, not for consultancies managing many small and mid-size clients.
- **Doing it manually** does not scale and misses problems.

---

## 4. The solution

A platform built **around the consultancy workflow**: everything is organised as *Consultancy, then Clients, then Platforms*, with monitoring, alerting, incident history, and client-facing reports built in.

### What it does

1. **Watches** every client platform continuously, from several regions.
2. **Records** every check result, error, and log.
3. **Detects** downtime, slowness, errors, and anomalies.
4. **Alerts** the right people through the right channels, with escalation if nobody responds.
5. **Tracks incidents** from start to resolution, with a full timeline.
6. **Reports** to clients through branded status pages and SLA reports.
7. **Integrates** with tools consultancies already use.

---

## 5. How we get the data

There are two ways to know a client's platform has a problem.

### Way 1: From the outside (we do the work)

Our probes visit the platform like a normal visitor would, every minute or so. No access to the client's server is needed.

| What we observe | What it means |
|---|---|
| No reply | The platform is down |
| Very slow reply | The platform is struggling |
| Error reply (500, 503) | The server is broken |
| Expected text missing | The page is blank or broken |
| Certificate near expiry | The platform will break soon |

### Way 2: From the inside (the client's side sends data)

Some problems cannot be seen from outside, such as a failing payment or a database error. For these, the client's side sends us data:

- **Log shippers**: a small program on the server forwards logs to us.
- **Webhooks**: the client's existing tools (Grafana, Sentry, PagerDuty) notify us when something happens.
- **Error reporting**: a small piece of code in the client's app reports errors.

Both ways feed one common pipeline, which detects problems and triggers alerts.

---

## 6. Users

| User | Goal | Frustration today |
|---|---|---|
| **Consultancy admin** | Manage the account, team, clients, and billing | No single place to see all client health |
| **Engineer** | Know instantly when something breaks, and why | Learning about outages from clients, noisy false alarms |
| **Account manager** | Prove reliability to clients | Building uptime reports by hand |
| **Client viewer** | See the current status of their platform | No visibility, has to ask the consultancy |

---

## 7. Feature overview

### Monitoring
- Website (HTTP/HTTPS), keyword and content checks
- API checks (status, response content, authentication)
- Multi-step transaction checks (log in, then perform an action)
- Ping, TCP port, and DNS checks
- SSL certificate and domain expiry warnings
- Heartbeat monitoring for scheduled jobs
- Multi-region probes with consensus (a platform is "down" only when several regions agree)

### Data from clients
- Log ingestion through standard shippers
- Webhook receivers for external tools
- Error tracking integration
- A unified timeline per platform combining checks, logs, errors, and incidents

### Detection
- Threshold rules (for example "down for 2 checks in a row")
- Anomaly detection based on each platform's normal behaviour
- Error-rate spike detection

### Alerting
- Channels: email, SMS, WhatsApp, Slack, Teams, push, voice, webhooks
- Escalation policies and on-call schedules
- Deduplication and grouping
- Maintenance windows
- Acknowledge and resolve

### Incident management
- Automatic open and close
- Timelines with human notes and root-cause fields
- Post-incident reports

### Client-facing
- Branded public status pages
- Monthly and quarterly SLA reports
- Subscriber notifications
- White-labelling

### Platform
- Multi-tenant isolation
- Roles and permissions
- Audit log
- Public API and API keys
- Billing and plans

---

## 8. What makes it different

1. **Built around clients.** Competing tools think in "monitors". We think in "Client A has these platforms, Client B has these".
2. **Reporting as a core feature.** Consultancies must prove reliability. One-click branded reports are central, not an add-on.
3. **Right-sized.** More capable than basic uptime tools, simpler and cheaper than enterprise platforms.
4. **Local market focus.** Alerting over WhatsApp and SMS, local pricing and payment methods, and probes in or near East Africa.
5. **Integrates instead of competing.** Grafana, Prometheus, and others are connected, not rebuilt.

---

## 9. High-level architecture

The system has two halves:

- **Control plane:** dashboard, accounts, clients, rules, reports. It is about correctness and usability.
- **Data plane:** probes, ingestion, detection, alerting. It is about volume, speed, and reliability.

```
Users -> Dashboard -> API (accounts, clients, rules, reports, billing)
                           |
                      monitor settings
                           |
Scheduler -> Probes (many regions) -> Event stream -> Evaluator -> Incident engine -> Alert router -> Channels
                                          ^                              |
Client agents / webhooks -> Ingestion ----+                              v
                                                         Status pages / Reports / Integrations

Storage: PostgreSQL (configuration) | time-series store (checks) | log store | cache
```

### Component roles

| Component | Job |
|---|---|
| Dashboard | What users see and use |
| API | Handles requests, rules, and data for the dashboard |
| Scheduler | Decides when each platform is due for a check |
| Probes | Perform the checks from different regions |
| Ingestion | Receives logs, errors, and webhooks from clients |
| Event stream | A buffer that absorbs bursts of data |
| Evaluator | Decides whether events mean a real problem |
| Incident engine | Opens, updates, and closes incidents |
| Alert router | Applies escalation rules and sends notifications |
| Storage | Keeps configuration, results, and logs |

---

## 10. Proposed technology (to be confirmed)

| Layer | Proposal | Reason |
|---|---|---|
| Probes and ingestion | Go | Handles thousands of simultaneous waits cheaply, small deployable programs for many regions |
| API and business logic | Python (FastAPI) or TypeScript (NestJS) | Fast to build, strong structure. Final choice to be made |
| Frontend | React with TypeScript (Next.js for public status pages) | Industry standard, fast public pages |
| Configuration data | PostgreSQL | Highly connected data |
| Check results | TimescaleDB, later ClickHouse | Built for time-based data |
| Logs | ClickHouse or Loki | Built for huge log volumes |
| Event stream | NATS, Kafka, or Redis streams | Buffers bursts |
| Cache | Redis | Speed and rate limiting |
| Own observability | OpenTelemetry, Prometheus, Grafana | A monitoring tool must be monitored |
| Infrastructure | Docker, managed Kubernetes, Terraform | Repeatable, scalable deployment |
| Anomaly service | Python | Best data analysis tools |

---

## 11. Reliability: who watches the watcher?

A monitoring tool that fails silently is worse than none. The design must include:

- Redundant schedulers and probes, with no single point of failure
- Probes hosted on different providers and regions
- An independent external monitor watching Uptime Tracker itself
- Periods when our own system was down marked as **"no data"**, never as "up"
- Health endpoints and metrics on our own services
- Backups and tested recovery

---

## 12. Security and privacy principles

- Strict tenant isolation: one consultancy never sees another's data
- Role-based access with least privilege
- Secrets and credentials encrypted
- Authenticated, encrypted data flow from client agents
- Masking of sensitive data (passwords, tokens, personal data) in logs
- Audit trail of every change
- Clear agreements about what is collected from clients and how long it is kept

---

## 13. Business model

- **SaaS subscription**, billed monthly or yearly
- Pricing by number of monitored platforms or clients, with tiers (for example Starter, Growth, Pro) differing in check frequency, regions, retention, and alert channels
- Add-ons: white-labelled status pages, branded reports, SMS and WhatsApp credits, extra log retention
- Possible free tier to attract consultancies

---

## 14. Build order

| Stage | Focus |
|---|---|
| 1 | Foundations: architecture decisions, data model, multi-tenancy, authentication, CI/CD, infrastructure |
| 2 | Core monitoring engine: scheduler, probes, event stream, storage, multi-region evaluator |
| 3 | Incident and alert engine: incidents, notifications, escalation, on-call, maintenance windows |
| 4 | Dashboard and client layer: dashboards, status pages, SLA reports, white-labelling |
| 5 | Deep data: log ingestion, agents, error and webhook integrations, unified timeline |
| 6 | Integrations: Prometheus, Grafana, Loki, PagerDuty, ticketing, public API |
| 7 | Intelligence: anomaly detection, trends |
| 8 | Business layer: billing, plans, admin console, SSO |
| 9 | Hardening: load tests, failure drills, security audit, disaster recovery |

Every stage is built to production quality. The order exists because later stages depend on earlier ones.

---

## 15. Risks and how to reduce them

| Risk | Mitigation |
|---|---|
| Scope grows too large | Stay focused on the consultancy niche, prioritise features (must, should, later) |
| False alarms destroy trust | Multi-region consensus, confirmation thresholds, deduplication |
| The tracker itself goes down | Redundancy, external watcher, "no data" handling |
| Security breach | Isolation, encryption, audits, minimal data collection |
| Log volume and cost explode | Separate log storage, short retention, sampling |
| Low adoption | Talk to real consultancies early and validate which features they would pay for |

---

## 16. Success measures

- Time from real outage to alert (target: under one minute)
- False alert rate
- Platform uptime of Uptime Tracker itself
- Number of consultancies, clients, and platforms monitored
- Reports generated and status pages viewed
- Customer retention

---

## 17. Open questions

- [ ] Final backend choice: FastAPI or NestJS
- [ ] Which regions to place probes in first
- [ ] Which alert channels to support first (email, SMS, WhatsApp)
- [ ] Pricing model details
- [ ] Which integrations to prioritise
- [ ] Name, branding, and domain

---

## 18. Next documents

1. `schema.md`: database design (tables, relationships, indexing, retention)
2. API contract: endpoints, permissions, errors, versioning
3. Flow diagrams: check and consensus, incident and escalation, log ingestion
4. Security and reliability plans
5. Mockups of the main screens