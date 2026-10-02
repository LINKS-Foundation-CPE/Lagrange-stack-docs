# quantum-api HTTP API

The complete surface of the platform backend. The machine-readable description is
**`api.yml`** (OpenAPI 3.1) in the repository, served as an interactive Swagger UI
at **`/api-docs`** on any running instance — use that to try calls; use this page
to understand them.

## Authentication

| Scheme | Token | Accepted on |
|---|---|---|
| **BackendJWT** | The platform token from `POST /auth/token`. HS256, signed with `BACKEND_SECRET`, default lifetime 24 h. | Every `/api/*` route |
| **KeycloakJWT** | An OIDC access token from the configured realm. RS256, validated against the realm's JWKS. | `POST /auth/token`, `POST /jobAuthorizer` |
| *(none)* | — | `GET /config`, `GET /api-docs`, `GET /metrics/jobs`, `POST /jobReport`, `PUT /jobReport/{jobid}`, `GET /userRoles/{username}` |

Send it as `Authorization: Bearer <token>`. A missing or malformed header is `401`;
a valid token without the required role is `403`.

!!! warning "Unauthenticated endpoints"
    The endpoints in the third row are **not authenticated in the current
    release**. `/jobReport` in particular creates job records and drives billing.
    They are safe only behind a network or proxy restriction — see
    [Gateway ↔ backend contract](machine-endpoints.md#authentication-status-and-hardening).

## List conventions

All collection endpoints follow the react-admin "simple REST" convention.

| Query parameter | Form | Example |
|---|---|---|
| `range` | `[start, end]`, inclusive, zero-based | `range=[0,24]` |
| `sort` | `["field", "ASC"\|"DESC"]` | `sort=["start","DESC"]` |
| `filter` | JSON object | `filter={"organization_id":3}` |

Filter keys accept the comparison suffixes `_gte`, `_lte`, `_gt`, `_lt`, `_ne`:

```
GET /api/jobs?filter={"submitted_datetime_gte":"2026-01-01T00:00:00Z"}&range=[0,49]&sort=["submitted_datetime","DESC"]
```

Every list response carries the total count:

```
Content-Range: items 0-49/1234
```

A client that ignores `Content-Range` will page incorrectly. Rows are additionally
scoped by the caller's role — see the
[visibility matrix](../architecture/identity.md#resource-authorization-in-the-backend).
Two resources take extra parameters:

- **`administrable=true`** on `GET /api/projects` — restrict to projects the caller
  can administer (all for platform admins, own organization for organization
  managers, administered projects for PIs).
- **`showOld=true`** on the calendar-backed lists — include slots and reservations
  that are already in the past, which are otherwise filtered out.

## Errors

```json
{ "message": "User 'user@example.org' is not member of project alpha" }
```

| Status | Meaning |
|---|---|
| `400` | Invalid body or query (zod validation), or a domain rule violated — overlapping reservation, insufficient budget, unknown project on a report. |
| `401` | Missing, malformed or expired token. |
| `403` | Authenticated but not permitted; also every authorization refusal from `/jobAuthorizer`. |
| `404` | No such resource. |
| `500` | Unhandled failure. |

The `message` on a `403` from `/jobAuthorizer` is what the gateway forwards to the
submitting client, so it is the text a user actually sees.

---

## Auth

### `POST /auth/token`

Exchanges an OIDC access token for a platform token. **Auth: KeycloakJWT.**

Side effects — this is not a pure read:

1. finds the user by `sub`, else by `email` (back-filling `sub`), else **creates
   the user row**;
2. on creation only, seeds `users.pulla_user` from the Pulse-access realm role;
3. derives platform roles and the list of administered projects;
4. writes a login notification and an audit-log entry.

```json title="200"
{ "token": "<platform token>" }
```

`401 {"message":"Missing or Invalid token"}` on any failure — including an
unreachable JWKS endpoint, which is worth checking before assuming the token is at
fault.

---

## Users

| Method | Path | Roles | Notes |
|---|---|---|---|
| `GET` | `/api/users` | scoped | Organization managers see their own organization. Supports substring search on the identity. |
| `POST` | `/api/users` | `admin`, `organization-manager` | Creates a user row before their first login. |
| `GET` | `/api/users/{id}` | `admin`, `organization-manager`, `project-admin` | |
| `PUT` | `/api/users/{id}` | `admin`, `organization-manager` | Partial update. |
| `GET` | `/api/users/{id}/projects` | scoped | Projects the user belongs to. |
| `GET` | `/api/users/{id}/jobs` | scoped | Jobs the user submitted. |

```json title="User"
{
  "id": 12,
  "email": "user@example.org",
  "sub": "b3f1…",
  "organization_id": 3,
  "default_project_id": 7,
  "organization_manager": false,
  "organization_auditor": false,
  "pulla_user": false
}
```

`email` is the identity — an e-mail address or any whitespace-free string
depending on [`USERNAME_FORMAT`](configuration.md#username_format). `sub` is
`null` until the user's first login. `pulla_user` is "Pulse access", writable by
platform administrators only — the field is dropped from a non-admin's payload
rather than refused. It is seeded from the identity provider when the row is
first created and belongs to the platform after that.

Creating a user with `organization_manager` or `organization_auditor` set is how
organization-scoped roles are granted — there is no separate role resource.

## Own (self-service)

| Method | Path | Notes |
|---|---|---|
| `GET` | `/api/own/projects` | The caller's projects, with budgets. |
| `GET` | `/api/own/default_project` | The caller's default project. |
| `PUT` | `/api/own/default_project` | Set it. The project must be one the caller belongs to. |

The default project is what a submission with no `project_name` is billed to, so
this is the one setting an ordinary user genuinely needs.

## Organizations

| Method | Path | Roles |
|---|---|---|
| `GET` | `/api/organizations` | scoped |
| `POST` | `/api/organizations` | `admin` |
| `GET` | `/api/organizations/{id}` | scoped |
| `PUT` | `/api/organizations/{id}` | `admin` |
| `POST` | `/api/organizations/{id}/budgetTransfer` | `admin`, `organization-manager` |
| `GET` | `/api/organizations/{id}/jobs` | scoped |

```json title="POST /api/organizations"
{
  "name": "ExampleOrg",
  "free_queue": false,
  "initial_budget": 100,
  "reference_organization_id": null
}
```

`initial_budget` is in **hours** and is the only budget field in the API that is
not milliseconds. Creating an organization also creates its **vault project**
(`"<name> Vault"`), credits it with the converted amount, and records a
non-billing ledger entry. `free_queue` is applied to the vault project.

`reference_organization_id` points at another organization whose slots this one may
also book into — used for umbrella/consortium arrangements.

```json title="POST /api/organizations/{id}/budgetTransfer"
{ "sourceProjectId": 7, "destinationProjectId": 9, "amount": 3600000 }
```

Both projects must belong to the organization. Two ledger entries are written, one
per side, flagged as administrative rather than billing.

## Projects

| Method | Path | Roles |
|---|---|---|
| `GET` | `/api/projects` | scoped; `administrable=true` supported |
| `POST` | `/api/projects` | `admin`, `organization-manager` |
| `GET` | `/api/projects/{id}` | scoped |
| `PUT` | `/api/projects/{id}` | `admin`, `organization-manager` |
| `GET` | `/api/projects/{id}/users` | scoped |
| `POST` | `/api/projects/{id}/users` | `admin`, `organization-manager`, `project-admin` |
| `GET` | `/api/projects/{id}/transactions` | scoped |
| `GET` | `/api/projects/{id}/jobs` | scoped |
| `PUT` | `/api/projects/{id}/tags` | `admin`, `organization-manager`, `project-admin` |

```json title="POST /api/projects"
{
  "name": "example-project",
  "organization_id": 3,
  "budget": 36000000,
  "start_at": "2026-01-01",
  "end_at": "2026-12-31",
  "free_queue": false
}
```

!!! warning "`budget` here is QPU-milliseconds"
    Unlike an organization's `initial_budget` (hours), a project's `budget` is a
    raw millisecond count. It is drawn from the organization's vault and the call
    fails with `400 Insufficient budget: requested N, available M` if the vault is
    short. Two ledger entries are written: a debit on the vault and a credit on the
    new project.

`name` is **unique platform-wide** and is the string users pass as `project_name`
when submitting. `start_at`/`end_at` bound validity: submissions outside the window
are refused. `free_queue` projects are never billed and their budget is not
consulted.

```json title="POST /api/projects/{id}/users"
{ "email": "user@example.org", "admin": false }
```

Adds a member by identity. `admin: true` makes them a **principal investigator** of
the project: they gain the `project-admin` platform role and can manage its
membership and reservations.

`GET /api/projects/{id}/transactions` returns the project's `budget_events`, i.e.
its ledger, paginated. This is the endpoint reporting is built on.

## Project membership

| Method | Path | Roles | Notes |
|---|---|---|---|
| `PUT` | `/api/projects_users/{id}` | `admin`, `organization-manager`, `project-admin` | Toggle the project-admin flag on a membership. |
| `DELETE` | `/api/projects_users/{id}` | `admin`, `organization-manager`, `project-admin` | Remove the membership. |

## Tags

A controlled vocabulary of project labels. Reading is open to any authenticated
caller — a project administrator needs the list to pick from — and writing is
platform-admin only, which is what makes the vocabulary controlled rather than
free text.

| Method | Path | Roles |
|---|---|---|
| `GET` | `/api/tags` | any authenticated |
| `POST` | `/api/tags` | `admin` |
| `GET` | `/api/tags/{id}` | any authenticated |
| `PUT` | `/api/tags/{id}` | `admin` |
| `DELETE` | `/api/tags/{id}` | `admin` |
| `PUT` | `/api/projects/{id}/tags` | `admin`, `organization-manager`, `project-admin` |

```json title="POST /api/tags"
{ "name": "teaching" }
```

Names are trimmed, at most 64 characters, and unique. Two refusals are `409` and
carry information available nowhere else, so they are worth surfacing verbatim:

- creating or renaming to a name already taken;
- deleting a tag still assigned to projects — the message says how many carry it.

```json title="PUT /api/projects/{id}/tags"
{ "tag_ids": [1, 4] }
```

Replaces the project's whole set and returns it, so an empty array clears the
tags. Ids, not names: assigning cannot create a tag as a side effect. An id that
is not in the vocabulary is `400`. Note the role list — a principal investigator
may tag a project whose other fields they cannot submit, because
`PUT /api/projects/{id}` is admin and organization-manager only. The two cannot
share a request.

Projects carry their tags in their own payload (`tags: [{ id, name }]`), so
listing projects needs no second call.

## Slots

Organization-level allocations of machine time. Platform-administrator only.

| Method | Path | Roles |
|---|---|---|
| `GET` | `/api/slots` | scoped; `showOld=true` supported |
| `POST` | `/api/slots` | `admin` |
| `GET` | `/api/slots/{id}` | scoped |
| `PUT` | `/api/slots/{id}` | `admin` |
| `DELETE` | `/api/slots/{id}` | `admin` |

```json title="Slot"
{ "id": 5, "organization_id": 3, "day": "2026-03-04",
  "start": "2026-03-04T09:00:00Z", "end": "2026-03-04T13:00:00Z" }
```

A slot books nothing. It delimits where that organization's projects may place
reservations, and — when `ALLOW_FOREIGN_ORG_QUEUE_IN_SLOTS=false` — who may use the
open queue during it. Deployments that do not need organization-level allocation
can turn the whole concept off with `SLOT_CONSTRAINED_RESERVATIONS=false`.

## Reservations

Exclusive machine access for one project, paid for at creation.

| Method | Path | Roles |
|---|---|---|
| `GET` | `/api/reservations` | scoped; `showOld=true` supported |
| `POST` | `/api/reservations` | `admin`, `organization-manager`, `project-admin` |
| `GET` | `/api/reservations/{id}` | scoped |
| `PUT` | `/api/reservations/{id}` | **not supported** |
| `DELETE` | `/api/reservations/{id}` | `admin`, `organization-manager`, `project-admin` |

```json title="POST /api/reservations"
{
  "project_id": 7,
  "slot_id": 5,
  "start": "2026-03-04T09:30:00Z",
  "end": "2026-03-04T10:30:00Z",
  "description": "calibration campaign"
}
```

Validated at creation, each failure a `400` with the reason:

- minimum duration and no overlap with an existing reservation;
- inside the project's validity window;
- the project has at least the full duration in budget — **debited immediately**;
- when `SLOT_CONSTRAINED_RESERVATIONS=true` (default): `slot_id` is **required**,
  the slot must belong to the project's organization (or the organization it
  references), and the reservation must fit inside the slot. When `false`,
  `slot_id` may be omitted and placement is free.

**`PUT` is deliberately not supported.** To change a reservation, delete and
recreate it — and note the refund rule below.

`DELETE` refunds the project: **100 %** if the reservation starts more than 24 h
from now, otherwise **25 %** (`round(ms/4)`). A ledger entry records the refund and
the affected users are notified. Deleting a reservation that has already started is
refused.

## Jobs

| Method | Path | Scope |
|---|---|---|
| `GET` | `/api/jobs` | Own jobs; platform admins see all |
| `GET` | `/api/jobs/{id}` | Same |

```json title="Job"
{
  "id": 981, "jobid": "b7c1e0f2-…",
  "organization_id": 3, "project_id": 7, "user_id": 12,
  "status": "ready", "job_type": "circuit",
  "execution_start": "2026-03-04T09:41:02.115Z",
  "execution_end":   "2026-03-04T09:41:05.842Z",
  "submitted_datetime": "2026-03-04T09:40:58.000Z",
  "submitted_circuit": "https://store.example.org/…/payload.json",
  "results": "https://jobs.example.org/…/index.html",
  "usedReservation": false
}
```

Jobs are **never created or edited through `/api/*`** — only by the gateway via
[`/jobReport`](machine-endpoints.md#post-jobreport). `submitted_circuit` and
`results` are URLs into the gateway's object store and results portal.
`usedReservation` marks a job that ran inside a reservation, which is why it was
not billed. Billed amount = `execution_end − execution_start`.

Project- and organization-scoped variants are
`GET /api/projects/{id}/jobs`, `GET /api/organizations/{id}/jobs` and
`GET /api/users/{id}/jobs`.

## Budget events (the ledger)

| Method | Path | Roles |
|---|---|---|
| `GET` | `/api/budgetEvents` | `admin` |
| `GET` | `/api/budgetEvents/{id}` | `admin` |

```json title="BudgetEvent"
{ "id": 4412, "project_id": 7, "value": "-3727",
  "date": "2026-03-04T09:41:06.001Z",
  "description": "Job b7c1e0f2-… (981) …",
  "billing": true }
```

`value` is a **signed millisecond amount serialised as a string** (`BIGINT`).
`billing: true` marks QPU consumption and refunds; `false` marks administrative
movements (vault creation, transfers). Per-project access is
`GET /api/projects/{id}/transactions`.

## Reports (billing)

The aggregates a *consuntivo* is built from, computed by the backend rather than
by exporting the database. Open to platform admins and to organization managers
and auditors, who are scoped to their own organization by the service. **Not**
project admins: these are per-organization accounting figures, and a PI's view of
their own project's consumption is the [ledger](#budget-events-the-ledger).

| Method | Path | Returns |
|---|---|---|
| `GET` | `/api/reports/reservations` | Reserved time |
| `GET` | `/api/reports/jobs` | Executed job time |
| `GET` | `/api/reports/slots` | Allocated slot time, by organization |
| `GET` | `/api/reports/utilization` | Reserved against used, per reservation |
| `GET` | `/api/reports/summary` | All of the above for one period |

All take `from`, `to` and `group_by=project|organization` (default `project`).

!!! warning "The period is exclusive of its own end day, and rows must fit inside it"
    `from`/`to` are instants; a bare date is read as **midnight UTC**, so
    `to=2025-12-31` ends at the *start* of the 31st and excludes that day. A row
    is counted only when it lies entirely inside the bounds — a reservation or
    job straddling either edge belongs to no period. This is deliberate: it is
    what the reconciled figures have always meant, and widening it would silently
    restate past reports.

Two rules make the numbers add up, and are the reason a report and a naive query
over the same tables disagree:

- **Organizations are reached through the project**, never through
  `jobs.organization_id` — that column is a snapshot taken when the job was
  reported, and disagrees for any project that has since moved between
  organizations.
- **Jobs that ran inside a reservation are excluded from job time.** That time is
  already billed as reserved hours; counting both would bill it twice. Rows whose
  end precedes their start are discarded rather than subtracted.

Durations are summed in seconds and converted to hours once. The reservation
utilization table is the only place that compares the two: time booked against
time actually used.

## Announcements and notifications

| Method | Path | Roles |
|---|---|---|
| `GET` | `/api/announcements` | any |
| `POST` `PUT` `DELETE` | `/api/announcements`, `/api/announcements/{id}` | `admin` |
| `GET` | `/api/activeAnnouncements` | any — those whose display window contains now |
| `GET` | `/api/notifications` | own |
| `PATCH` | `/api/notifications/{id}/read` | own |

Announcements are operator broadcasts with a display window (`start`/`end`) and are
rendered on the dashboard landing page. Notifications are per-user and generated by
the system — logins, reservation changes affecting projects the user belongs to.

## Audit log

| Method | Path | Roles |
|---|---|---|
| `GET` | `/api/logs` | `admin` |
| `GET` | `/api/logs/{id}` | `admin` |

```json title="ActionLog"
{ "id": 20551, "timestamp": "2026-03-04T09:12:00.000Z", "user_id": 4,
  "action": "create", "resource": "projects", "resource_id": 7,
  "description": "Project example-project (7) created by admin@example.org (…)" }
```

`action` values include `login`, `registration`, `create`, `update`, `delete`,
`reportingError` and `reportingWarning`. The last two are written when a job report
cannot be attributed, and are the first place to look when a job exists on the
machine but not in the accounting system.

## Deployment configuration

### `GET /config`

Public and unauthenticated. The client-safe subset of deployment flags, so the
dashboard can adapt without being rebuilt per deployment.

```json title="200"
{ "slotConstrainedReservations": true }
```

Only booleans that are safe to reveal publicly are exposed here — never secrets or
internal configuration. If you add a flag the UI must react to, add it here.

## Machine endpoints

Documented in full in [Gateway ↔ backend contract](machine-endpoints.md):

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/jobAuthorizer` | May this submission run? |
| `POST` | `/jobReport` | Record a submitted job. |
| `PUT` | `/jobReport/{jobid}` | Finalise a job — **this is what bills**. |
| `GET` | `/userRoles/{username}` | Platform roles for an identity. |

## Metrics

### `GET /metrics/jobs`

Prometheus text exposition format, unauthenticated, untyped and without
`# HELP` / `# TYPE` lines.

| Series | Meaning |
|---|---|
| `jobs_total` | Every job on record. |
| `jobs_<status>` | One per status **currently present** in the table, so the set follows the data rather than a fixed list; a job with no status counts as `jobs_unknown`. |
| `jobs_by_organization{organization_id,organization}` | Jobs per organization, including organizations with none (`0`). |
| `reservations_by_organization{...}` | Reservations per organization, same shape. |
| `qpu_seconds_by_organization{...}` | Executed QPU time per organization, in seconds. |
| `jobs_by_project{project_id,project,organization_id,organization}` | Jobs per project, including projects with none. The organization rides along as a label, so a query can group by it without joining. |
| `qpu_seconds_by_project{...}` | Executed QPU time per project. |
| `users_total` | Every user on record. |
| `users_by_email_domain{domain}` | Users grouped by the domain of their identity. |
| `projects_by_tag{tag_id,tag}` | Projects carrying each tag, including tags nobody uses. |

```
jobs_total 4
jobs_ready 1
jobs_by_organization{organization_id="1",organization="Example Org"} 2
qpu_seconds_by_organization{organization_id="1",organization="Example Org"} 1830
jobs_by_project{project_id="7",project="example-project",organization_id="1",organization="Example Org"} 2
users_total 42
users_by_email_domain{domain="students.example.org"} 18
projects_by_tag{tag_id="3",tag="teaching"} 5
```

Three things about these series are worth knowing before building a panel on
them.

**`jobs_<status>` are gauges, not counters.** A job moving on decrements the
status it left.

**The counts are strict, except per-tag.** A job or reservation belongs to
exactly one project and so to exactly one organization; a parent organization
does *not* absorb the counts of the organizations that reference it, so summing
the series gives the true total. Per-tag is the exception: a project may carry
several tags, so those series overlap by construction and do not sum to the
number of projects.

**`qpu_seconds_*` is not the billing reports' usage figure.** It sums
`execution_end - execution_start` over **all** jobs, including those that ran
inside a reservation — which [reports](#reports-billing) deliberately exclude. A
metric measures machine time consumed; a report measures what is owed. A job that
never ran, or one whose window ends before it starts, contributes no seconds
while still counting as a job.

There is one series per user identity **domain** rather than a metric per
institution, so a question like "how many student accounts" is a label selector
and no deployment's naming is compiled into the API. An identity with no `@`
counts in `users_total` and in no domain, which is the only way the two disagree.

The gateway exposes its own, richer metrics (queue depth, per-user active jobs)
separately.
