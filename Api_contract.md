# Uptime Tracker: API Contract

| | |
|---|---|
| **Version** | 0.1 (draft) |
| **Status** | For review |
| **API version** | `v1` |
| **Style** | REST over HTTPS, JSON |
| **Companion docs** | `app_concept.md`, `schema.md` |
| **Next docs** | OpenAPI file (`openapi.yaml`), flow diagrams, security plan |

---

## 1. Purpose and scope

This document is the agreement between the Uptime Tracker backend and everything that talks to it: the web dashboard, customer integrations, client agents, and our own services. It is written **before** the code, so the backend, the frontend, and the OpenAPI file are all built from one source.

It defines:

- Conventions that apply to every endpoint (authentication, errors, pagination, versioning)
- Every endpoint: method, path, purpose, permitted roles, and behaviour
- Example requests and responses for the important endpoints
- Real-time updates, outgoing webhooks, and ingestion endpoints
- The schema changes this contract requires (section 18)

It does **not** define internal service-to-service protocols in full (see section 16 for an outline).

---

## 2. API surfaces

Uptime Tracker exposes four distinct surfaces. They have different callers, authentication, and rate limits, so they are separate on purpose.

| Surface | Base URL (placeholder) | Callers | Authentication |
|---|---|---|---|
| **Management API** | `https://api.uptimetracker.example/v1` | Dashboard, customer scripts | User session token or API key |
| **Public status API** | `https://api.uptimetracker.example/v1/public` | Status pages, anyone | None (read-only, cached) |
| **Ingestion API** | `https://ingest.uptimetracker.example/v1` | Client agents, log shippers, monitoring tools | Ingestion key or webhook token |
| **Internal API** | Private network only | Probes, workers | Service credentials (see section 16) |

The ingestion surface runs on its own hostname and its own servers, so a flood of log traffic can never slow down the dashboard.

---

## 3. Conventions

### 3.1 General

| Item | Rule |
|---|---|
| Format | JSON in and out (`Content-Type: application/json`), except where noted (log ingestion accepts NDJSON) |
| Field naming | `snake_case` |
| IDs | UUIDs |
| Timestamps | ISO 8601 in UTC, for example `2026-11-12T14:03:22Z` |
| Durations | Integers with the unit in the name (`interval_seconds`, `response_time_ms`) |
| Enums | Lowercase strings, identical to the database enums in `schema.md` |
| Unknown fields in requests | Rejected with `400` (catches typos early) |
| Null vs missing | On `PATCH`, a missing field means "unchanged", an explicit `null` means "clear it" |
| Request IDs | Every response carries `X-Request-Id`. Clients should quote it in support requests |
| Character encoding | UTF-8 |

### 3.2 Tenant scoping

A user can belong to several organizations, so the organization is explicit in the path:

```
/v1/orgs/{org_id}/clients
/v1/orgs/{org_id}/platforms/{platform_id}
```

The server verifies on every request that the caller is a member of `{org_id}` (or that the API key belongs to it). A resource that exists in another organization returns `404`, never `403`, so IDs cannot be probed.

### 3.3 Response envelope

Successful single-resource response:

```json
{
  "data": { "id": "6c1f...", "name": "Online Banking Portal" }
}
```

Successful list response:

```json
{
  "data": [ { "id": "..." }, { "id": "..." } ],
  "meta": {
    "next_cursor": "eyJ0IjoiMjAyNi0xMS0xMlQxNDowMzoyMloifQ",
    "has_more": true
  }
}
```

`204 No Content` has no body.

### 3.4 Pagination

Cursor-based (stable under constant inserts, unlike page numbers).

| Parameter | Meaning | Default | Max |
|---|---|---|---|
| `limit` | Items per page | 50 | 200 |
| `cursor` | Opaque value from the previous response's `meta.next_cursor` | none | |

Cursors are opaque and expire after 24 hours.

### 3.5 Filtering and sorting

- Filters are plain query parameters: `?status=down&client_id=...&tag=production`
- Multiple values for one filter are comma-separated: `?status=down,degraded`
- Time ranges use `from` and `to` (inclusive start, exclusive end)
- Sorting uses `sort`, with a leading `-` for descending: `?sort=-started_at`
- Each list endpoint documents which filters and sort fields it supports. Unsupported ones return `400`.

### 3.6 Errors

All errors use the RFC 9457 problem format (`application/problem+json`):

```json
{
  "type": "https://docs.uptimetracker.example/errors/validation-failed",
  "title": "Validation failed",
  "status": 422,
  "code": "validation_failed",
  "detail": "One or more fields are invalid.",
  "request_id": "req_01JD4Y8K2M",
  "errors": [
    { "field": "interval_seconds", "code": "out_of_range", "message": "Must be between 10 and 86400." },
    { "field": "target", "code": "invalid_url", "message": "Must be a valid http or https URL." }
  ]
}
```

| HTTP status | Meaning |
|---|---|
| 400 | Malformed request (bad JSON, unknown field, bad query parameter) |
| 401 | Missing, invalid, or expired credentials |
| 403 | Authenticated but the role or scope does not allow this action |
| 404 | Not found (or belongs to another organization) |
| 409 | Conflict (duplicate name, version mismatch, state does not allow this action) |
| 412 | `If-Match` precondition failed (someone else changed the resource) |
| 422 | Valid JSON but fails validation rules |
| 429 | Rate limit exceeded (see `Retry-After`) |
| 500 | Server error (safe to retry with backoff) |
| 503 | Temporarily unavailable (see `Retry-After`) |

The full list of `code` values is in section 17.

### 3.7 Idempotency

Every `POST` that creates something or triggers an action accepts an `Idempotency-Key` header (a client-generated UUID). Repeating the same key within 24 hours returns the original response instead of creating a duplicate. This protects against network retries.

Reusing a key with a different request body returns `422` with code `idempotency_key_reused`.

### 3.8 Concurrency control

Resources that people edit (platforms, alert rules, escalation policies, status pages, incidents) return an `ETag` header. A `PATCH` or `PUT` may send `If-Match` with that value. If the resource changed in between, the server returns `412` instead of overwriting someone else's edit. `If-Match` is optional for API keys and required for the dashboard.

### 3.9 Rate limiting

Every response includes:

```
X-RateLimit-Limit: 600
X-RateLimit-Remaining: 598
X-RateLimit-Reset: 1731420240
```

Exceeding a limit returns `429` with a `Retry-After` header (seconds).

| Surface | Limit (per credential) | Burst |
|---|---|---|
| Management API, user session | 600 requests per minute | 100 |
| Management API, API key | 1,200 requests per minute (by plan) | 200 |
| Public status API | 120 requests per minute per IP (cached at the edge) | 30 |
| Ingestion API | By plan, measured in events and bytes per minute | Documented per plan |
| Authentication endpoints | 10 requests per minute per IP and per account | 5 |
| "Run check now" and "send test notification" | 10 per minute per organization | 3 |

### 3.10 Versioning and deprecation

- The version is in the path (`/v1`).
- Within a version, changes are **additive only**: new endpoints, new optional fields, new enum values (clients must tolerate unknown enum values).
- Breaking changes require a new version (`/v2`).
- A deprecated endpoint returns `Deprecation` and `Sunset` headers at least **6 months** before removal, and is announced in the changelog.

### 3.11 Caching

- `GET` responses on the public status API are cacheable for 30 seconds (`Cache-Control: public, max-age=30`).
- Management API responses are `Cache-Control: no-store`, except `ETag` revalidation on single resources.

---

## 4. Authentication and authorization

### 4.1 Credential types

| Credential | Format | Used for | Lifetime |
|---|---|---|---|
| **Access token** | JWT in `Authorization: Bearer <token>` | Dashboard users | 15 minutes |
| **Refresh token** | Opaque, in an `HttpOnly`, `Secure`, `SameSite=Strict` cookie | Obtaining new access tokens | 30 days, rotated on every use |
| **API key** | `Authorization: Bearer ut_live_<random>` | Customer automation | Until revoked or expired |
| **Ingestion key** | `X-Ingest-Key: ing_<random>` | Client agents on the ingestion surface | Until rotated |
| **Webhook token** | A secret segment in the URL path | Inbound webhooks from other tools and heartbeat pings | Until rotated |

API keys, ingestion keys, and webhook tokens are shown **once** at creation. Only a hash is stored.

### 4.2 Refresh token safety

Refresh tokens rotate on every use. Presenting an already-used refresh token is treated as theft: the whole session family is revoked and the user must sign in again.

### 4.3 Roles

| Role | Capability summary |
|---|---|
| `owner` | Everything, including deleting the organization and managing billing |
| `admin` | Everything except deleting the organization and changing ownership |
| `engineer` | Create and edit clients, platforms, alert rules, incidents. Acknowledge alerts. No member, API key, or billing management |
| `viewer` | Read-only on monitoring data. Can see but not change anything |
| `billing` | Billing and subscription only. Cannot see monitoring data |

### 4.4 API key scopes

An API key carries scopes that limit it further than a role would.

| Scope | Allows |
|---|---|
| `read` | All `GET` endpoints on monitoring data |
| `platforms:write` | Create, edit, pause, delete platforms |
| `clients:write` | Create, edit, delete clients |
| `incidents:write` | Create and update incidents |
| `alerts:write` | Acknowledge and resolve alerts, manage rules and policies |
| `maintenance:write` | Manage maintenance windows |
| `status_pages:write` | Manage status pages |
| `ingest:write` | Reserved for ingestion keys |

Keys never get member management, API key management, or billing access.

### 4.5 Permission matrix (Management API)

Legend: **R** read, **W** create/edit/delete, **-** none. Billing role sees only the billing row.

| Resource | owner | admin | engineer | viewer | billing |
|---|---|---|---|---|---|
| Organization settings | W | W | R | R | - |
| Members and invitations | W | W | R | - | - |
| API keys | W | W | - | - | - |
| Audit log | R | R | - | - | - |
| Clients and contacts | W | W | W | R | - |
| Platforms and monitoring settings | W | W | W | R | - |
| Check results, uptime, metrics | R | R | R | R | - |
| Incidents | W | W | W | R | - |
| Alerts (acknowledge, resolve) | W | W | W | R | - |
| Alert rules, escalation, on-call | W | W | W | R | - |
| Notification channels | W | W | W | R | - |
| Maintenance windows | W | W | W | R | - |
| Status pages | W | W | W | R | - |
| Reports | W | W | W | R | - |
| Ingestion sources, error groups | W | W | W | R | - |
| Integrations | W | W | R | - | - |
| Billing and subscription | W | R | - | - | W |

---

## 5. Endpoint catalog: Authentication and account

Base path: `/v1`. Rate limit tier: authentication.

| Method | Path | Purpose | Auth |
|---|---|---|---|
| POST | `/auth/register` | Create a user and a first organization | None |
| POST | `/auth/login` | Exchange email and password for tokens. Returns an MFA challenge if enabled | None |
| POST | `/auth/mfa/verify` | Complete login with a one-time code | MFA challenge token |
| POST | `/auth/refresh` | Get a new access token (rotates the refresh token) | Refresh cookie |
| POST | `/auth/logout` | Revoke the current session | Access token |
| POST | `/auth/logout-all` | Revoke every session of the user | Access token |
| POST | `/auth/password/forgot` | Send a reset email (always returns `202`, never reveals whether the email exists) | None |
| POST | `/auth/password/reset` | Set a new password using the emailed token | Reset token |
| POST | `/auth/email/verify` | Confirm an email address | Verification token |
| POST | `/auth/email/resend` | Resend the verification email | Access token |
| GET | `/me` | The current user and their organization memberships | Access token |
| PATCH | `/me` | Update name, phone, locale | Access token |
| POST | `/me/password` | Change password (requires the current one) | Access token |
| POST | `/me/mfa/setup` | Start MFA enrollment, returns a secret and QR payload | Access token |
| POST | `/me/mfa/enable` | Confirm enrollment with a code, returns recovery codes (shown once) | Access token |
| POST | `/me/mfa/disable` | Disable MFA (requires password and a code) | Access token |
| GET | `/me/sessions` | List active sessions | Access token |
| DELETE | `/me/sessions/{session_id}` | Revoke one session | Access token |

**Login response (no MFA):**

```json
{
  "data": {
    "access_token": "eyJhbGciOi...",
    "token_type": "Bearer",
    "expires_in": 900,
    "user": { "id": "0f9a...", "email": "adrien@example.com", "name": "Adrien" },
    "organizations": [
      { "id": "a31b...", "name": "Acme Consulting", "slug": "acme-consulting", "role": "owner" }
    ]
  }
}
```

The refresh token is set as a cookie, never placed in the JSON body.

---

## 6. Endpoint catalog: Organizations, members, API keys, audit

Base path: `/v1/orgs/{org_id}` (except where noted).

| Method | Path | Purpose | Roles |
|---|---|---|---|
| POST | `/v1/orgs` | Create another organization (the caller becomes owner) | any user |
| GET | `/v1/orgs/{org_id}` | Get organization settings | all members |
| PATCH | `/v1/orgs/{org_id}` | Update name, slug, timezone, locale | owner, admin |
| DELETE | `/v1/orgs/{org_id}` | Close the organization (starts the 90-day purge clock) | owner |
| GET | `/members` | List members and roles | owner, admin, engineer |
| PATCH | `/members/{member_id}` | Change a member's role | owner, admin |
| DELETE | `/members/{member_id}` | Remove a member | owner, admin |
| POST | `/ownership/transfer` | Transfer ownership to another member | owner |
| GET | `/invitations` | List pending invitations | owner, admin |
| POST | `/invitations` | Invite someone by email with a role | owner, admin |
| DELETE | `/invitations/{invitation_id}` | Revoke an invitation | owner, admin |
| POST | `/v1/invitations/accept` | Accept an invitation using its emailed token | any user |
| GET | `/api-keys` | List keys (prefix and metadata only) | owner, admin |
| POST | `/api-keys` | Create a key. The full secret is returned **once** | owner, admin |
| DELETE | `/api-keys/{api_key_id}` | Revoke a key | owner, admin |
| GET | `/audit-logs` | Search the audit trail | owner, admin |

**Rules:**

- An organization must always have at least one `owner`. Demoting or removing the last owner returns `409` (`last_owner`).
- The `audit-logs` endpoint supports filters `actor_user_id`, `action`, `entity_type`, `entity_id`, `from`, `to`.
- Audit entries are read-only. There is no endpoint to modify or delete them.

**Create API key request and response:**

```json
// POST /v1/orgs/{org_id}/api-keys
{
  "name": "Grafana sync script",
  "scopes": ["read", "incidents:write"],
  "expires_at": "2027-05-12T00:00:00Z"
}
```

```json
// 201 Created
{
  "data": {
    "id": "e4d2...",
    "name": "Grafana sync script",
    "key_prefix": "ut_live_a1b2",
    "scopes": ["read", "incidents:write"],
    "expires_at": "2027-05-12T00:00:00Z",
    "secret": "ut_live_a1b2c3d4e5f6g7h8i9j0..."
  }
}
```

`secret` appears only in this response.

---

## 7. Endpoint catalog: Clients and contacts

Base path: `/v1/orgs/{org_id}`.

| Method | Path | Purpose | Roles (write) |
|---|---|---|---|
| GET | `/clients` | List clients. Filters: `q` (name search), `has_open_incident`, `status`. Sort: `name`, `-created_at` | read: all |
| POST | `/clients` | Create a client | owner, admin, engineer |
| GET | `/clients/{client_id}` | Get a client, including a health summary | all |
| PATCH | `/clients/{client_id}` | Update name, reference, SLA target, notes | owner, admin, engineer |
| DELETE | `/clients/{client_id}` | Soft-delete the client and its platforms | owner, admin |
| GET | `/clients/{client_id}/summary` | Aggregated health: platform count by status, open incidents, uptime for a period (`from`, `to`) | all |
| GET | `/clients/{client_id}/contacts` | List contacts | all |
| POST | `/clients/{client_id}/contacts` | Add a contact | owner, admin, engineer |
| PATCH | `/clients/{client_id}/contacts/{contact_id}` | Update a contact | owner, admin, engineer |
| DELETE | `/clients/{client_id}/contacts/{contact_id}` | Remove a contact | owner, admin, engineer |

**Create client:**

```json
// POST /v1/orgs/{org_id}/clients
{
  "name": "ABC Bank",
  "external_ref": "CL-0042",
  "sla_target_percent": 99.9,
  "notes": "Primary contact is the CTO."
}
```

**Rules:**

- Client names are unique per organization, case-insensitive. A duplicate returns `409` (`duplicate_name`).
- `DELETE` is a soft delete. Related incidents, reports, and history remain readable until the retention purge.
- Contacts need at least an email or a phone number.

---

## 8. Endpoint catalog: Platforms (monitors)

Base path: `/v1/orgs/{org_id}`.

### 8.1 Management

| Method | Path | Purpose | Roles (write) |
|---|---|---|---|
| GET | `/platforms` | List platforms. Filters: `client_id`, `status`, `check_type`, `tag`, `q`, `is_active`. Sort: `name`, `-created_at`, `status_since` | read: all |
| POST | `/platforms` | Create a platform | owner, admin, engineer |
| GET | `/platforms/{platform_id}` | Get a platform with its current state | all |
| PATCH | `/platforms/{platform_id}` | Update settings (supports `If-Match`) | owner, admin, engineer |
| DELETE | `/platforms/{platform_id}` | Soft-delete (stops checks, keeps history) | owner, admin, engineer |
| POST | `/platforms/{platform_id}/pause` | Pause monitoring, optionally until a time (`paused_until`) | owner, admin, engineer |
| POST | `/platforms/{platform_id}/resume` | Resume monitoring | owner, admin, engineer |
| PUT | `/platforms/{platform_id}/regions` | Replace the probe regions for this platform | owner, admin, engineer |
| POST | `/platforms/{platform_id}/run-check` | Run one check immediately from chosen regions and return the result (does not affect state or incidents) | owner, admin, engineer |
| POST | `/platforms/bulk` | Create up to 100 platforms in one call (all or nothing) | owner, admin, engineer |
| POST | `/platforms/bulk-action` | Pause, resume, tag, or delete many platforms by ID or filter | owner, admin, engineer |
| GET | `/probe-regions` | List available probe regions | all |

### 8.2 Monitoring data

| Method | Path | Purpose | Roles |
|---|---|---|---|
| GET | `/platforms/{platform_id}/state` | Current status, status since, last check, open incident | all |
| GET | `/platforms/{platform_id}/check-results` | Raw check results. Filters: `from`, `to`, `region`, `is_up`, `error_category`. Cursor pagination. Limited to the plan's raw retention | all |
| GET | `/platforms/{platform_id}/metrics` | Aggregated series for charts. Parameters: `from`, `to`, `resolution` (`1m`, `5m`, `1h`, `1d`), `region`. The server picks raw or daily data and may coarsen the resolution (reported in `meta.resolution`) | all |
| GET | `/platforms/{platform_id}/uptime` | **Official uptime** for a period, calculated with the rules in `schema.md` section 10 (incident downtime minus maintenance and no-data time) | all |
| GET | `/platforms/{platform_id}/ssl` | Latest certificate details and expiry | all |
| GET | `/platforms/{platform_id}/events` | Combined timeline of checks transitions, incidents, alerts, and client events | all |

**Create platform:**

```json
// POST /v1/orgs/{org_id}/platforms
{
  "client_id": "9b2c...",
  "name": "Online Banking Portal",
  "check_type": "keyword",
  "target": "https://bank.example.com/login",
  "config": {
    "method": "GET",
    "expected_status": [200],
    "keyword": "Sign in",
    "keyword_must_exist": true,
    "headers": { "User-Agent": "UptimeTracker/1.0" }
  },
  "interval_seconds": 60,
  "timeout_seconds": 10,
  "failure_threshold": 3,
  "recovery_threshold": 2,
  "region_quorum": 2,
  "slow_threshold_ms": 3000,
  "regions": ["rw-kigali", "ke-nairobi", "eu-frankfurt"],
  "tags": ["production", "banking"]
}
```

```json
// 201 Created, header: ETag: "v1"
{
  "data": {
    "id": "5d70...",
    "client_id": "9b2c...",
    "name": "Online Banking Portal",
    "check_type": "keyword",
    "target": "https://bank.example.com/login",
    "interval_seconds": 60,
    "timeout_seconds": 10,
    "failure_threshold": 3,
    "recovery_threshold": 2,
    "region_quorum": 2,
    "slow_threshold_ms": 3000,
    "regions": ["rw-kigali", "ke-nairobi", "eu-frankfurt"],
    "tags": ["production", "banking"],
    "is_active": true,
    "paused_until": null,
    "state": {
      "status": "unknown",
      "status_since": "2026-11-12T14:03:22Z",
      "last_check_at": null,
      "open_incident_id": null
    },
    "created_at": "2026-11-12T14:03:22Z",
    "updated_at": "2026-11-12T14:03:22Z"
  }
}
```

**Official uptime response:**

```json
// GET /platforms/{id}/uptime?from=2026-10-01T00:00:00Z&to=2026-11-01T00:00:00Z
{
  "data": {
    "period": { "from": "2026-10-01T00:00:00Z", "to": "2026-11-01T00:00:00Z" },
    "uptime_percent": 99.9412,
    "downtime_seconds": 1524,
    "maintenance_seconds": 3600,
    "no_data_seconds": 0,
    "incident_count": 2,
    "sla_target_percent": 99.9,
    "sla_met": true
  }
}
```

**Validation rules for create and update:**

| Field | Rule |
|---|---|
| `check_type` | One of the `check_type` enum values. Cannot be changed after creation |
| `target` | Must match the type (valid http or https URL, or a host and port for `tcp`, and so on). Private and internal addresses are rejected (see section 14) |
| `interval_seconds` | 10 to 86400, and not below the plan's minimum interval |
| `timeout_seconds` | 1 to 120, and strictly less than `interval_seconds` |
| `failure_threshold`, `recovery_threshold`, `region_quorum` | At least 1. `region_quorum` may not exceed the number of regions |
| `regions` | At least one, and limited by the plan |
| Platform count | Limited by the plan. Exceeding it returns `403` (`plan_limit_reached`) |

**Notes:**

- `config` secrets (for example authentication headers) are accepted on write and **never returned**. Responses show `"headers": { "Authorization": "[redacted]" }`.
- `run-check` is a diagnostic. It never changes platform state, creates incidents, or sends alerts.
- `check-results` returns `404`-style empty pages beyond the plan's retention rather than an error, with `meta.retention_days` explaining the limit.

---

## 9. Endpoint catalog: Incidents

Base path: `/v1/orgs/{org_id}`.

| Method | Path | Purpose | Roles (write) |
|---|---|---|---|
| GET | `/incidents` | List incidents. Filters: `status`, `severity`, `type`, `client_id`, `platform_id`, `from`, `to`, `is_public`. Sort: `-started_at` (default), `severity` | read: all |
| POST | `/incidents` | Create a manual incident | owner, admin, engineer |
| GET | `/incidents/{incident_id}` | Get an incident with affected platforms and latest updates | all |
| PATCH | `/incidents/{incident_id}` | Edit title, severity, root cause, visibility (supports `If-Match`) | owner, admin, engineer |
| POST | `/incidents/{incident_id}/acknowledge` | Mark as acknowledged by the caller | owner, admin, engineer |
| POST | `/incidents/{incident_id}/resolve` | Resolve manually (requires `resolution_summary`) | owner, admin, engineer |
| POST | `/incidents/{incident_id}/reopen` | Reopen within 24 hours of resolution | owner, admin, engineer |
| PUT | `/incidents/{incident_id}/platforms` | Replace the list of affected platforms and impact levels | owner, admin, engineer |
| GET | `/incidents/{incident_id}/updates` | The full timeline | all |
| POST | `/incidents/{incident_id}/updates` | Post a note or public update. Optionally changes the status | owner, admin, engineer |
| GET | `/incidents/{incident_id}/alerts` | Alerts and notifications linked to the incident | all |
| GET | `/incidents/{incident_id}/context` | Evidence at the time: check results, state changes, and related client events in a window around the incident | all |

**Create manual incident:**

```json
// POST /v1/orgs/{org_id}/incidents
{
  "client_id": "9b2c...",
  "title": "Slow payments reported by client",
  "type": "manual",
  "severity": "major",
  "started_at": "2026-11-12T13:40:00Z",
  "platforms": [ { "platform_id": "5d70...", "impact": "degraded" } ],
  "is_public": false
}
```

**Post a public update:**

```json
// POST /v1/orgs/{org_id}/incidents/{id}/updates
{
  "kind": "public_update",
  "body": "We have identified the cause and are applying a fix.",
  "new_status": "identified",
  "is_public": true
}
```

**Rules:**

- Automatic incidents are opened and closed by the system. Engineers can acknowledge, annotate, and edit them, but a system-detected `started_at` cannot be edited (reports depend on it honestly).
- `resolve` on an automatic incident while the platform is still failing returns `409` (`still_failing`) unless `force: true` is sent. A forced resolve is recorded in the timeline and the audit log.
- Incident `number` (for example `1042`) is assigned by the server and appears as `INC-1042` in the dashboard and in notifications.
- Status changes only move forward: `investigating`, `identified`, `monitoring`, `resolved`. Moving backward uses `reopen`.
- A public update is rejected if the incident is not `is_public`.

---

## 10. Endpoint catalog: Alerting

Base path: `/v1/orgs/{org_id}`.

### 10.1 Notification channels

| Method | Path | Purpose | Roles (write) |
|---|---|---|---|
| GET | `/channels` | List channels | read: all |
| POST | `/channels` | Create a channel (starts unverified) | owner, admin, engineer |
| GET | `/channels/{channel_id}` | Get a channel (secrets redacted) | all |
| PATCH | `/channels/{channel_id}` | Update | owner, admin, engineer |
| DELETE | `/channels/{channel_id}` | Soft-delete. Returns `409` if still used by an escalation step (`channel_in_use`) | owner, admin, engineer |
| POST | `/channels/{channel_id}/verify` | Send a verification code to the destination | owner, admin, engineer |
| POST | `/channels/{channel_id}/verify/confirm` | Confirm with the code | owner, admin, engineer |
| POST | `/channels/{channel_id}/test` | Send a test notification | owner, admin, engineer |

Unverified channels cannot be used by escalation steps. Webhook channels are verified by sending a signed challenge that the receiver must echo.

### 10.2 Escalation policies

| Method | Path | Purpose | Roles (write) |
|---|---|---|---|
| GET | `/escalation-policies` | List policies | read: all |
| POST | `/escalation-policies` | Create a policy with its steps in one request | owner, admin, engineer |
| GET | `/escalation-policies/{policy_id}` | Get a policy with ordered steps | all |
| PUT | `/escalation-policies/{policy_id}` | Replace the policy and all its steps atomically | owner, admin, engineer |
| DELETE | `/escalation-policies/{policy_id}` | Delete. Returns `409` if used by a rule (`policy_in_use`) | owner, admin, engineer |

```json
// POST /v1/orgs/{org_id}/escalation-policies
{
  "name": "Banking, 24/7",
  "repeat_count": 1,
  "steps": [
    { "step_order": 1, "delay_minutes": 0,  "target_type": "schedule", "target_id": "c7aa..." },
    { "step_order": 2, "delay_minutes": 10, "target_type": "user",     "target_id": "0f9a..." },
    { "step_order": 3, "delay_minutes": 30, "target_type": "channel",  "target_id": "e1b4..." }
  ]
}
```

The API accepts a single `target_id` plus `target_type` and maps it to the correct database column.

### 10.3 On-call schedules

| Method | Path | Purpose | Roles (write) |
|---|---|---|---|
| GET | `/on-call/schedules` | List schedules | read: all |
| POST | `/on-call/schedules` | Create a schedule | owner, admin, engineer |
| PATCH | `/on-call/schedules/{schedule_id}` | Update name, time zone, rotation type | owner, admin, engineer |
| DELETE | `/on-call/schedules/{schedule_id}` | Delete (`409` if in use) | owner, admin, engineer |
| GET | `/on-call/schedules/{schedule_id}/shifts` | List shifts in a range (`from`, `to`) | all |
| PUT | `/on-call/schedules/{schedule_id}/shifts` | Replace shifts in a range (rejects overlaps unless marked override) | owner, admin, engineer |
| POST | `/on-call/schedules/{schedule_id}/overrides` | Add a temporary override for one user | owner, admin, engineer |
| GET | `/on-call/schedules/{schedule_id}/now` | Who is on call right now | all |

### 10.4 Alert rules

| Method | Path | Purpose | Roles (write) |
|---|---|---|---|
| GET | `/alert-rules` | List. Filters: `scope`, `platform_id`, `client_id`, `tag`, `condition_type`, `is_enabled` | read: all |
| POST | `/alert-rules` | Create a rule | owner, admin, engineer |
| GET | `/alert-rules/{rule_id}` | Get a rule | all |
| PATCH | `/alert-rules/{rule_id}` | Update (supports `If-Match`) | owner, admin, engineer |
| DELETE | `/alert-rules/{rule_id}` | Soft-delete | owner, admin, engineer |
| POST | `/alert-rules/{rule_id}/enable` | Enable | owner, admin, engineer |
| POST | `/alert-rules/{rule_id}/disable` | Disable | owner, admin, engineer |
| POST | `/alert-rules/{rule_id}/dry-run` | Evaluate the rule against the last N hours of data and report when it would have fired (nothing is sent) | owner, admin, engineer |

```json
// POST /v1/orgs/{org_id}/alert-rules
{
  "name": "Banking: slow responses",
  "scope": "tag",
  "tag": "banking",
  "condition_type": "response_time",
  "condition_config": { "threshold_ms": 3000, "duration_minutes": 5 },
  "severity": "major",
  "escalation_policy_id": "b8c1...",
  "cooldown_minutes": 30,
  "respect_maintenance": true
}
```

Validation: `scope` determines which of `platform_id`, `client_id`, `tag` is required and the others must be absent. `condition_config` is validated against a schema chosen by `condition_type`.

### 10.5 Alerts and notifications

| Method | Path | Purpose | Roles |
|---|---|---|---|
| GET | `/alerts` | List alerts. Filters: `status`, `severity`, `platform_id`, `client_id`, `incident_id`, `from`, `to` | all |
| GET | `/alerts/{alert_id}` | Get an alert with its notification history | all |
| POST | `/alerts/{alert_id}/acknowledge` | Acknowledge (stops further escalation) | owner, admin, engineer |
| POST | `/alerts/{alert_id}/resolve` | Resolve manually | owner, admin, engineer |
| GET | `/notifications` | Delivery log. Filters: `alert_id`, `incident_id`, `channel_id`, `status`, `kind`, `from`, `to` | owner, admin, engineer |

Alerts can also be acknowledged straight from a notification, through a signed one-time link included in emails and chat messages. Those links call `POST /v1/public/alerts/acknowledge` with a token and carry no other permissions.

---

## 11. Endpoint catalog: Maintenance windows

Base path: `/v1/orgs/{org_id}`.

| Method | Path | Purpose | Roles (write) |
|---|---|---|---|
| GET | `/maintenance-windows` | List. Filters: `client_id`, `platform_id`, `active` (now), `from`, `to` | read: all |
| POST | `/maintenance-windows` | Schedule a window | owner, admin, engineer |
| GET | `/maintenance-windows/{window_id}` | Get | all |
| PATCH | `/maintenance-windows/{window_id}` | Update (not allowed for windows that already ended) | owner, admin, engineer |
| DELETE | `/maintenance-windows/{window_id}` | Cancel | owner, admin, engineer |
| POST | `/maintenance-windows/{window_id}/end` | End an active window early | owner, admin, engineer |

```json
// POST /v1/orgs/{org_id}/maintenance-windows
{
  "title": "Database upgrade",
  "client_id": "9b2c...",
  "platform_ids": ["5d70...", "81ae..."],
  "starts_at": "2026-11-15T22:00:00Z",
  "ends_at": "2026-11-16T00:00:00Z",
  "suppress_alerts": true,
  "show_on_status_page": true
}
```

Rules: `ends_at` must be after `starts_at`. Either `client_id` (covers all of that client's platforms) or `platform_ids` (or both) must be given. A window created in the past is rejected with `422` (`window_in_past`), because it would silently rewrite history.

---

## 12. Endpoint catalog: Status pages and the public API

### 12.1 Status pages (management)

Base path: `/v1/orgs/{org_id}`.

| Method | Path | Purpose | Roles (write) |
|---|---|---|---|
| GET | `/status-pages` | List | read: all |
| POST | `/status-pages` | Create a page for a client | owner, admin, engineer |
| GET | `/status-pages/{page_id}` | Get | all |
| PATCH | `/status-pages/{page_id}` | Update title, slug, branding, visibility, custom domain | owner, admin, engineer |
| DELETE | `/status-pages/{page_id}` | Delete | owner, admin |
| PUT | `/status-pages/{page_id}/components` | Replace the ordered list of platforms shown, with display names | owner, admin, engineer |
| GET | `/status-pages/{page_id}/subscribers` | List subscribers | owner, admin, engineer |
| DELETE | `/status-pages/{page_id}/subscribers/{subscriber_id}` | Remove a subscriber | owner, admin, engineer |
| POST | `/status-pages/{page_id}/custom-domain/verify` | Check the DNS records for the custom domain and issue a certificate | owner, admin |

### 12.2 Public status API (no authentication)

Base path: `/v1/public`. Cached for 30 seconds. Returns only data marked public. Never returns internal IDs of organizations, only the page slug.

| Method | Path | Purpose |
|---|---|---|
| GET | `/status/{slug}` | Overall status, component statuses, active incidents, upcoming maintenance |
| GET | `/status/{slug}/incidents` | Public incident history (`from`, `to`, cursor) |
| GET | `/status/{slug}/incidents/{incident_number}` | One public incident with its public updates |
| GET | `/status/{slug}/uptime` | Per-component uptime for the last 90 days |
| GET | `/status/{slug}/feed.rss` | RSS feed of incidents |
| POST | `/status/{slug}/subscribers` | Subscribe by email (sends a confirmation email) |
| POST | `/subscribers/confirm` | Confirm a subscription with the emailed token |
| POST | `/subscribers/unsubscribe` | Unsubscribe with the token from any email |
| POST | `/alerts/acknowledge` | Acknowledge an alert using a signed one-time token |

```json
// GET /v1/public/status/abc-bank
{
  "data": {
    "title": "ABC Bank System Status",
    "overall_status": "degraded",
    "updated_at": "2026-11-12T14:05:00Z",
    "components": [
      { "name": "Online Banking", "status": "up" },
      { "name": "Mobile API", "status": "degraded" }
    ],
    "active_incidents": [
      {
        "number": 1042,
        "title": "Slow mobile API responses",
        "status": "identified",
        "started_at": "2026-11-12T13:58:10Z",
        "latest_update": "We have identified the cause and are applying a fix."
      }
    ],
    "upcoming_maintenance": []
  }
}
```

Status pages on a custom domain serve the same data through the front end. Subscriber endpoints are rate limited per IP and per email address to prevent abuse.

---

## 13. Endpoint catalog: Reports, ingestion management, integrations, billing

Base path: `/v1/orgs/{org_id}`.

### 13.1 Reports

| Method | Path | Purpose | Roles (write) |
|---|---|---|---|
| GET | `/report-schedules` | List schedules. Filter: `client_id` | read: all |
| POST | `/report-schedules` | Create | owner, admin, engineer |
| PATCH | `/report-schedules/{schedule_id}` | Update | owner, admin, engineer |
| DELETE | `/report-schedules/{schedule_id}` | Delete | owner, admin, engineer |
| GET | `/reports` | List generated reports. Filters: `client_id`, `from`, `to` | all |
| POST | `/reports` | Generate a report for a client and period. Returns `202` with a job | owner, admin, engineer |
| GET | `/reports/{report_id}` | Get a report's frozen numbers and status | all |
| GET | `/reports/{report_id}/download` | Get a short-lived signed URL to the PDF (`302` redirect or JSON) | all |
| POST | `/reports/{report_id}/send` | Send the report to the client's contacts or a custom list | owner, admin, engineer |
| GET | `/jobs/{job_id}` | Status of any asynchronous operation (`queued`, `running`, `succeeded`, `failed`) | all |

```json
// POST /v1/orgs/{org_id}/reports
{ "client_id": "9b2c...", "period_start": "2026-10-01", "period_end": "2026-10-31" }

// 202 Accepted, header: Location: /v1/orgs/{org_id}/jobs/7f31...
{ "data": { "job_id": "7f31...", "status": "queued" } }
```

A report for a period that is not yet complete is rejected with `422` (`period_not_finished`). Generated numbers never change afterwards.

### 13.2 Ingestion sources and error groups (management side)

| Method | Path | Purpose | Roles (write) |
|---|---|---|---|
| GET | `/ingestion-sources` | List sources with status and last received time | read: all |
| POST | `/ingestion-sources` | Create a source. The ingestion key is returned **once** | owner, admin, engineer |
| GET | `/ingestion-sources/{source_id}` | Get | all |
| PATCH | `/ingestion-sources/{source_id}` | Update parsing and masking rules, expected interval | owner, admin, engineer |
| POST | `/ingestion-sources/{source_id}/rotate-key` | Issue a new key (the old key keeps working for a grace period, default 24 hours) | owner, admin, engineer |
| DELETE | `/ingestion-sources/{source_id}` | Disable and delete | owner, admin, engineer |
| GET | `/ingestion-sources/{source_id}/setup-instructions` | Ready-to-paste snippets for supported shippers | all |
| GET | `/error-groups` | List grouped errors. Filters: `platform_id`, `status`, `from`, `to`. Sort: `-last_seen_at`, `-occurrence_count` | all |
| GET | `/error-groups/{group_id}` | Get a group with a recent sample | all |
| PATCH | `/error-groups/{group_id}` | Change status (`ongoing`, `resolved`, `ignored`) | owner, admin, engineer |
| GET | `/platforms/{platform_id}/logs` | Search logs for a platform (queries the log store). Parameters: `from`, `to`, `severity`, `q`, `source_type`, cursor. Limited to the plan's log retention | all |

### 13.3 Integrations

| Method | Path | Purpose | Roles (write) |
|---|---|---|---|
| GET | `/integrations` | List | read: owner, admin, engineer |
| POST | `/integrations` | Connect an integration (credentials encrypted, never returned) | owner, admin |
| GET | `/integrations/{integration_id}` | Get status and last sync | owner, admin, engineer |
| PATCH | `/integrations/{integration_id}` | Update settings | owner, admin |
| DELETE | `/integrations/{integration_id}` | Disconnect | owner, admin |
| POST | `/integrations/{integration_id}/test` | Test the connection | owner, admin |
| GET | `/integrations/{integration_id}/mappings` | List mappings | owner, admin, engineer |
| PUT | `/integrations/{integration_id}/mappings` | Replace the mappings between external objects and platforms | owner, admin |

### 13.4 Billing

| Method | Path | Purpose | Roles |
|---|---|---|---|
| GET | `/v1/plans` | List available plans (public, no authentication) | any |
| GET | `/billing/subscription` | Current plan, status, period, limits | owner, admin (R), billing |
| POST | `/billing/subscription/change` | Change plan or billing period | owner, billing |
| POST | `/billing/subscription/cancel` | Cancel at period end | owner, billing |
| GET | `/billing/usage` | Usage against limits (platforms, checks, SMS, WhatsApp, log volume) | owner, admin, billing |
| POST | `/billing/portal-session` | Get a link to the payment provider's hosted portal for payment methods and invoices | owner, billing |

Payment details and invoices live with the payment provider. Uptime Tracker never receives card numbers.

---

## 14. Ingestion API

Base URL: `https://ingest.uptimetracker.example/v1`. Authentication: `X-Ingest-Key`, or a webhook token inside the path where noted. This surface is optimised for volume: small responses, batch endpoints, strict size limits.

### 14.1 Endpoints

| Method | Path | Purpose |
|---|---|---|
| POST | `/ingest/logs` | Send a batch of log lines |
| POST | `/ingest/events` | Send structured error or application events |
| GET, POST | `/heartbeat/{heartbeat_token}` | A scheduled job "checks in" (cron monitoring). Optional `?status=fail` and `?duration_ms=` |
| POST | `/webhooks/alertmanager/{webhook_token}` | Receive Prometheus Alertmanager notifications |
| POST | `/webhooks/grafana/{webhook_token}` | Receive Grafana alert notifications |
| POST | `/webhooks/sentry/{webhook_token}` | Receive Sentry issue and alert events |
| POST | `/webhooks/pagerduty/{webhook_token}` | Receive PagerDuty incident events |
| POST | `/webhooks/generic/{webhook_token}` | Receive a simple documented JSON payload from any tool |
| GET | `/ping` | Connectivity and key check for agent setup (returns the source's name and status) |

### 14.2 Rules

| Item | Rule |
|---|---|
| Batch format | `/ingest/logs` accepts NDJSON (one event per line) or a JSON array |
| Max request size | 5 MB (compressed bodies accepted with `Content-Encoding: gzip`) |
| Max events per request | 5,000 |
| Max single event size | 64 KB (larger events are truncated and flagged) |
| Response | `202 Accepted` with `{ "accepted": 4998, "rejected": 2, "rejections": [ { "index": 17, "code": "invalid_timestamp" } ] }`. Data is queued, not yet searchable |
| Backpressure | `429` or `503` with `Retry-After`. Agents must retry with backoff and must not drop data silently |
| Authentication failure | `401`, with no detail about whether the key exists |
| Timestamps | Clients send `event_time` in UTC. Events more than 24 hours in the future or older than the plan's retention are rejected individually |
| Idempotency | Optional `event_id` per event. Duplicates within 24 hours are discarded |
| Masking | Sensitive-data masking from the source's configuration is applied **before** storage |
| Silence detection | If a source has an `expected_interval_seconds` and nothing arrives for longer, the source becomes `silent` and a `no_data` alert may fire |

### 14.3 Example: log batch

```
POST /v1/ingest/logs
X-Ingest-Key: ing_k3j2...
Content-Type: application/x-ndjson

{"event_time":"2026-11-12T14:01:03Z","source_type":"access_log","severity":"info","message":"GET /login 200 41ms","fields":{"status":200,"path":"/login","latency_ms":41}}
{"event_time":"2026-11-12T14:01:04Z","source_type":"error_log","severity":"error","message":"connection to db refused","fields":{"service":"auth"}}
```

```json
// 202 Accepted
{ "accepted": 2, "rejected": 0, "rejections": [] }
```

### 14.4 Example: heartbeat

```
GET https://ingest.uptimetracker.example/v1/heartbeat/hb_9xk2...
→ 200 OK  { "status": "received", "next_expected_by": "2026-11-12T15:00:00Z" }
```

### 14.5 Mapping external tools

Inbound webhooks are translated into the common event shape (see `schema.md` section 6). Matching to a platform uses `integration_mappings`. An event that matches no mapping is stored against the source's client and flagged `unmapped`, never discarded.

---

## 15. Real-time updates and outgoing webhooks

### 15.1 Real-time stream (dashboard)

`GET /v1/orgs/{org_id}/stream` returns a Server-Sent Events stream (`text/event-stream`). One connection per browser tab. The server sends a heartbeat comment every 25 seconds.

| Event type | When |
|---|---|
| `platform.status_changed` | A platform's status changes |
| `incident.opened` | An incident is created |
| `incident.updated` | Status, severity, or notes change |
| `incident.resolved` | An incident is resolved |
| `alert.fired` | An alert starts |
| `alert.acknowledged` | An alert is acknowledged |
| `alert.resolved` | An alert resolves |
| `ingestion.source_silent` | A source stopped sending data |

```
event: platform.status_changed
id: 01JD4Z0R6V
data: {"platform_id":"5d70...","from":"up","to":"down","at":"2026-11-12T14:03:22Z"}
```

Clients reconnect with the `Last-Event-ID` header and receive any events missed within the last 5 minutes. Beyond that they refetch state through the normal endpoints. The stream carries identifiers and minimal fields, never secrets or full resources, so clients refetch details they need.

### 15.2 Outgoing webhooks

Customers can receive events at their own URL. Outgoing webhook endpoints are managed here:

| Method | Path | Purpose | Roles (write) |
|---|---|---|---|
| GET | `/webhook-endpoints` | List endpoints | read: owner, admin, engineer |
| POST | `/webhook-endpoints` | Register an endpoint with its event types. The signing secret is returned **once** | owner, admin |
| PATCH | `/webhook-endpoints/{endpoint_id}` | Update URL, events, enabled state | owner, admin |
| DELETE | `/webhook-endpoints/{endpoint_id}` | Remove | owner, admin |
| POST | `/webhook-endpoints/{endpoint_id}/rotate-secret` | New signing secret (old one valid for 24 hours) | owner, admin |
| POST | `/webhook-endpoints/{endpoint_id}/test` | Send a test event | owner, admin |
| GET | `/webhook-endpoints/{endpoint_id}/deliveries` | Delivery attempts and responses | owner, admin, engineer |
| POST | `/webhook-endpoints/{endpoint_id}/deliveries/{delivery_id}/redeliver` | Send an event again | owner, admin |

Event types: the same list as the stream, plus `maintenance.started`, `maintenance.ended`, `report.generated`, and `member.added`.

**Payload:**

```json
{
  "id": "evt_01JD4Z0R6V",
  "type": "incident.opened",
  "created_at": "2026-11-12T14:03:25Z",
  "organization_id": "a31b...",
  "data": {
    "incident": {
      "id": "d92e...",
      "number": 1042,
      "title": "Online Banking Portal is down",
      "severity": "critical",
      "status": "investigating",
      "started_at": "2026-11-12T14:00:12Z",
      "platforms": [ { "id": "5d70...", "name": "Online Banking Portal" } ]
    }
  }
}
```

**Signing:** each request carries

```
UT-Signature: t=1731420205,v1=5257a869e7ecebeda32affa62cdca3fa51cad7e77a0e56ff536d0ce8e108d8bd
```

`v1` is an HMAC-SHA256 of `"<t>.<raw request body>"` using the endpoint's secret. Receivers must reject signatures older than 5 minutes and compare in constant time.

**Delivery:** a `2xx` response within 10 seconds counts as delivered. Otherwise retry with exponential backoff (1 min, 5 min, 30 min, 2 h, 6 h, 12 h) for up to 24 hours, then mark failed. Delivery order is not guaranteed, so receivers should use `id` for deduplication and `created_at` for ordering. Endpoints that fail continuously for 7 days are disabled and the owners are notified.

**Destination safety:** webhook and monitoring targets must resolve to public addresses. The server rejects `localhost`, private ranges, link-local ranges, and cloud metadata addresses, and re-checks the resolved address at connect time to defeat DNS tricks.

---

## 16. Internal API (outline)

Not exposed publicly. Included so the boundaries are agreed before the backend is built.

| Interaction | Direction | Transport | Notes |
|---|---|---|---|
| Probe fetches its assignments (platforms, intervals, config) | Probe to control plane | gRPC or HTTPS pull | Authenticated with per-probe credentials (mutual TLS). Configuration secrets are delivered encrypted for that probe only |
| Probe submits check results | Probe to event stream | NATS or Kafka publish, batched | Each result carries `platform_id`, `probe_region_id`, `checked_at` |
| Evaluator reads results and writes state | Worker | Stream consumer, database | Applies thresholds and region quorum, updates `platform_state`, opens and closes incidents in one transaction |
| Alert engine fires and escalates | Worker | Database and queue | Reads `alerts_due_idx`, enqueues notifications |
| Notification senders | Worker to providers | Provider APIs | Records every attempt in `notifications` |
| Report generator | Worker | Database, file storage | Writes frozen numbers and the PDF |
| Health reporting | Every service | HTTP | Each service exposes `/health/live`, `/health/ready` and Prometheus metrics on a private port |

The detailed message formats belong in a separate internal-protocol document.

---

## 17. Meta endpoints and error code catalog

### 17.1 Meta

| Method | Path | Purpose | Auth |
|---|---|---|---|
| GET | `/v1/health` | Basic availability for external monitors (also watched by an independent monitor of ours) | None |
| GET | `/v1/version` | API version and build identifier | None |
| GET | `/v1/openapi.json` | The OpenAPI document | None |
| GET | `/v1/changelog` | Machine-readable API changelog | None |

### 17.2 Error codes

| Code | Status | Meaning |
|---|---|---|
| `invalid_request` | 400 | Malformed JSON, unknown field, or bad parameter |
| `unauthenticated` | 401 | Missing or invalid credentials |
| `token_expired` | 401 | Access token expired. Refresh and retry |
| `mfa_required` | 401 | A second factor is needed to continue |
| `forbidden` | 403 | Role does not permit this action |
| `insufficient_scope` | 403 | API key lacks the scope |
| `plan_limit_reached` | 403 | The organization's plan limit prevents this |
| `organization_suspended` | 403 | The organization is suspended or closed |
| `not_found` | 404 | Resource not found |
| `duplicate_name` | 409 | A resource with that name exists |
| `last_owner` | 409 | Action would leave no owner |
| `channel_in_use`, `policy_in_use`, `schedule_in_use` | 409 | Resource is referenced elsewhere |
| `still_failing` | 409 | Cannot resolve while the platform is still failing without `force` |
| `invalid_state` | 409 | The resource's current state does not allow this action |
| `precondition_failed` | 412 | `If-Match` did not match |
| `validation_failed` | 422 | One or more fields are invalid (see `errors`) |
| `idempotency_key_reused` | 422 | Same key, different request |
| `window_in_past` | 422 | Maintenance window starts in the past |
| `period_not_finished` | 422 | Report period is not complete |
| `unsafe_target` | 422 | Target resolves to a private or forbidden address |
| `rate_limited` | 429 | Too many requests |
| `internal_error` | 500 | Unexpected server error |
| `unavailable` | 503 | Temporarily unavailable |

---

## 18. Schema changes required by this contract

Writing the contract exposed gaps in `schema.md` v0.1. These must be added as new migrations (or edits to unreleased ones) before the matching endpoints can be built.

| # | Addition | Needed for |
|---|---|---|
| 1 | `sessions` (id, user_id, refresh_token_hash, family_id, user_agent, ip, created_at, last_used_at, expires_at, revoked_at) | Refresh token rotation, `/me/sessions`, theft detection |
| 2 | `auth_tokens` (id, user_id, purpose [`email_verify`, `password_reset`, `mfa_challenge`], token_hash, expires_at, used_at) | Email verification, password reset, MFA challenge |
| 3 | `mfa_recovery_codes` (id, user_id, code_hash, used_at) | MFA recovery |
| 4 | `idempotency_keys` (organization_id or user_id, key, request_hash, response_status, response_body, created_at, expires_at) | `Idempotency-Key` support |
| 5 | `jobs` (id, organization_id, type, status, input, result, error, created_at, started_at, finished_at) | Asynchronous operations such as report generation |
| 6 | `reports.status` column (`queued`, `generating`, `ready`, `failed`) and `reports.job_id` | Report lifecycle |
| 7 | `webhook_endpoints` (id, organization_id, url, event_types, secret_enc, previous_secret_enc, previous_secret_expires_at, is_enabled, failing_since, created_at) | Outgoing webhooks |
| 8 | `webhook_deliveries` (id, endpoint_id, event_id, event_type, payload, attempt, status, response_status, response_body_excerpt, next_attempt_at, created_at) | Delivery log, retries, redelivery |
| 9 | `channel_verifications` (id, channel_id, code_hash, expires_at, attempts, confirmed_at) | Channel verification |
| 10 | `platforms.heartbeat_token_hash` (or a `heartbeat_tokens` table) | Heartbeat monitors |
| 11 | `ingestion_sources.previous_key_hash` and `previous_key_expires_at` | Key rotation grace period |
| 12 | `alert_ack_tokens` (or signed tokens without storage) | One-time acknowledge links in notifications |
| 13 | `integration_type` value `webhook`, and `webhook_token_hash` on `integrations` | Generic inbound webhooks and Alertmanager, Grafana, Sentry, PagerDuty tokens |
| 14 | `incidents.reopened_at` (or an update record) | Reopen rule |
| 15 | `status_page_subscribers` token handling for confirmation (`confirm_token_hash`) | Double opt-in |

---

## 19. Open decisions

| # | Decision | Options | Needed by |
|---|---|---|---|
| 1 | Authentication provider | Build in-house, or use a managed identity service (affects SSO later) | Stage 1 |
| 2 | Real-time transport | SSE (as drafted) or WebSockets | Stage 4 |
| 3 | Probe protocol | gRPC pull or message-queue push | Stage 2 |
| 4 | Whether `PUT` replacement semantics are acceptable for steps and shifts, or whether item-level endpoints are needed | Keep `PUT` or add item endpoints | Stage 3 |
| 5 | Public API access for customers on all plans, or paid plans only | All, or Growth and above | Stage 7 |
| 6 | Bulk endpoints: all-or-nothing vs partial success | All-or-nothing (as drafted) or per-item results | Stage 2 |
| 7 | Cursor lifetime and signing | 24 hours, signed | Stage 1 |
| 8 | Whether to expose a GraphQL endpoint for dashboard queries later | Not now | Later |
| 9 | Official client libraries | Python and TypeScript first | Later |
| 10 | Confirmation of API hostnames and whether the status page uses the same API | To be decided with infrastructure | Stage 4 |

---

## 20. Delivery plan for this document

1. Review this contract and settle the open decisions that block Stage 1.
2. Apply the schema changes in section 18 to `schema.md` (version 0.2).
3. Convert this document into `openapi.yaml`, with request and response schemas for every endpoint, so documentation, validation, and test clients can be generated from it.
4. Build in the order of the stages in `app_concept.md`, with contract tests that fail when an implementation drifts from the OpenAPI file.

---

## 21. Change log

| Version | Date | Change |
|---|---|---|
| 0.1 | 2026-11-12 | First full draft |