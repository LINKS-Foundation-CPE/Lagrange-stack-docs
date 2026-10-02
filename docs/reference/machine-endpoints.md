# Gateway ↔ backend contract

Three endpoints on quantum-api exist for the gateway rather than for humans. They
are the seam between the data plane and the control plane, and they are the part of
the stack most worth reading carefully before you deploy it — both because
integrating a new site plugin means implementing this contract, and because their
authentication story is incomplete in the current release.

| Endpoint | Called by | When | Effect |
|---|---|---|---|
| `POST /jobAuthorizer` | proxy | Before every submission, in `production` mode | Read-only decision |
| `POST /jobReport` | proxy, and the reporter as a fallback | Immediately after the machine accepts a submission | Creates the job record |
| `PUT /jobReport/{jobid}` | reporter | When a job reaches a terminal state | Finalises the job and **bills the project** |
| `GET /userRoles/{username}` | auth plugin | When the credential carries no roles | Read-only lookup |

---

## `POST /jobAuthorizer`

**"May this identity submit, and against which project?"**

### Request

```http
POST /jobAuthorizer
Authorization: Bearer <the submitting user's OIDC token>
Content-Type: application/json

{ "username": "user@example.org", "project_name": "example-project", "job_type": "sweep" }
```

`project_name` is present only when the submission carried one. `job_type` is
present only when the vendor plugin classified the request — for IQM it is
`circuit` or `sweep`, derived from the submission path, not from the body (a
sweep is submitted as an opaque blob). **A request with no `job_type` is treated
as the ordinary circuit path**, which is what a gateway older than the field
sends, so the two sides can be deployed in either order.

The gateway forwards the original request's **identity** headers — the user's
`Authorization` above all, which is how it reaches the backend. Headers that
describe the submission's own body are deliberately not forwarded (`Host`,
`Content-Length`, `Content-Type`, `Content-Encoding`, `Transfer-Encoding`,
`Connection`, `Expect`, `Accept-Encoding`): this is a different request with a
body the gateway builds itself, and a forwarded `Content-Type` would describe
the wrong one.

### Response

| Status | Meaning |
|---|---|
| `200` | Authorized. Body ignored. |
| `403` + `{"message": "..."}` | Refused. The gateway returns this body to the client verbatim, so the message is what the user sees. |
| anything else | Treated as a refusal. |

### Decision

The full decision tree — project resolution, validity window, reservation
precedence, slot rules, budget — is in
[Authorization of a submission](../architecture/accounting.md#authorization-of-a-submission).
The messages, in the order they can occur:

| Message | Cause |
|---|---|
| `user not found in database` | The identity has never logged in and was never created. |
| `User '<email>' does not have pulse access` | `job_type` is `sweep` and `users.pulla_user` is false. |
| `Project '<name>' not found` | No project by that name. |
| `User '<id>' is not member of project <name>` | Not a member. |
| `no project provided and no default project defined` | No `project_name`, no default project. |
| `unknown default project` | The default project id no longer resolves. |
| `Project '<name>' already expired` / `not yet started` | Outside the validity window. |
| `Slot reserved to another project` | A reservation for another project is active. |
| `Slot reserved to another organization` | `ALLOW_FOREIGN_ORG_QUEUE_IN_SLOTS=false` and another organization's slot is active. |
| `No budget available for project <name>` | `remaining_budget` is not positive and the project is not `free_queue`. |

### Failure behaviour

If the backend is unreachable or errors, the site plugin returns *not authorized*
and the gateway refuses the submission. **Authorization fails closed**: a backend
outage stops new work rather than admitting unbilled work.

### Implementation notes

The call is read-only with one exception: it back-fills the user's `sub` if the row
was created by an administrator before the user's first login. It creates no job
row and no budget event. Note also that **the authorization call and the subsequent
report resolve the project independently** — `/jobAuthorizer` uses the
`project_name` it was given or the default project, and
[`/jobReport` re-resolves](../architecture/accounting.md#attribution-when-the-job-is-recorded)
including a reservation check. In normal operation they agree.

---

## `POST /jobReport`

**"This job now exists."** Sent immediately after the machine returns a job id.

```http
POST /jobReport
Content-Type: application/json

{
  "username": "user@example.org",
  "jobid": "b7c1e0f2-…",
  "project_name": "example-project",
  "status": "pending_compilation",
  "job_type": "circuit",
  "submitted_datetime": "2026-03-04T09:40:58.000Z",
  "submitted_circuit": "https://store.example.org/…/payload.json"
}
```

| Field | Required | Notes |
|---|---|---|
| `username` | ✅ | The identity. Must resolve to an existing user, or the call is `400` and an audit-log `reportingError` is written. |
| `jobid` | ✅ | The machine's job id. The key for every later reference. |
| `submitted_datetime` | ✅ | ISO 8601. Parsed as **UTC** — send an offset. |
| `project_name` | | Overrides the resolved project. |
| `status`, `job_type` | | Free-form strings from the vendor plugin. |
| `execution_start`, `execution_end` | | Normally absent here. |
| `submitted_circuit`, `results` | | URLs into the object store / results portal. |

**Response:** `200` on success; `400` if the user or the project cannot be
resolved.

Project attribution happens here and is not the same computation as the
authorization call — see
[attribution](../architecture/accounting.md#attribution-when-the-job-is-recorded).
In particular, a job submitted while a reservation the user participates in is
active gets flagged `usedReservation`, which exempts it from billing.

If this call fails, the job exists on the machine but not in the accounting system.
The terminal `PUT` will then `404`, and the reporter falls back to `POST` — which
can still create the record, late. Nothing is billed for a job that was never
created.

---

## `PUT /jobReport/{jobid}`

**"This job is finished."** Sent by the reporter on a terminal status. This is the
call that moves money.

```http
PUT /jobReport/b7c1e0f2-…
Content-Type: application/json

{
  "status": "ready",
  "execution_start": "2026-03-04T09:41:02.115Z",
  "execution_end":   "2026-03-04T09:41:05.842Z",
  "results": "https://jobs.example.org/…/index.html"
}
```

Inside one transaction the backend:

1. loads the job by `jobid` — `404` if unknown;
2. **returns early if the job already has both execution timestamps** — the call is
   idempotent and a repeat cannot double-bill;
3. applies the supplied fields;
4. bills `execution_end − execution_start` milliseconds and appends the ledger
   entry, unless the project is `free_queue`, the job is flagged `usedReservation`,
   or the execution window is incomplete.

**Response:** `200` on success (including the idempotent no-op), `404` for an
unknown job id, `500` on an internal failure.

!!! note "PUT-first, POST-fallback"
    The reference site plugin sends the terminal report as `PUT /jobReport/{id}`
    and, if that does not return `200`, retries as `POST /jobReport`. This repairs
    the case where the initial report never landed: the job gets created at the
    end of its life instead of the beginning. The consequence is that a `POST`
    carrying a complete execution window creates a job that is **already
    finalised** — and therefore never billed, because billing happens only in the
    `PUT` path. If jobs are silently unbilled, check whether their initial report
    was lost.

---

## `GET /userRoles/{username}`

**"What is this identity allowed to do at the machine?"** For deployments where
the credential presented to the gateway carries no realm roles — a batch-scheduler
principal, for instance — this is where the gateway gets them instead.

```http
GET /userRoles/user@example.org
```

```json title="200"
{ "username": "user@example.org", "roles": ["cortex_user", "pulla_user"] }
```

`404 {"message":"user not found"}` for an unknown identity.

The response is derived from the user row: `cortex_user` for any known user, plus
`pulla_user` when the `pulla_user` flag is set. That flag is maintained both by
administrators and by the
[Pulse-access mirror](../architecture/identity.md#platform-roles-how-the-backend-derives-them)
at login, so an identity ends up with the same capability whichever way it
authenticates.

---

## Authentication status, and hardening

This is the part to read before exposing quantum-api to anything beyond its own
gateway.

| Endpoint | What it verifies today | Gap |
|---|---|---|
| `POST /jobAuthorizer` | The **submitting user's** OIDC token | Verifies the user, not the *caller*. Any holder of a user token can call it directly. It is read-only, so the impact is information disclosure rather than corruption — but see below. |
| `POST /jobReport` | **Nothing** | Anyone who can reach it can create job records attributed to any user and project. |
| `PUT /jobReport/{jobid}` | **Nothing** | Anyone who can reach it can finalise a job with an arbitrary execution window — i.e. write an arbitrary amount into the billing ledger. |
| `GET /userRoles/{username}` | **Nothing** | Discloses whether an identity exists and what it may do. |
| `GET /metrics/jobs` | **Nothing** | Aggregate counts only. |

The consequence, stated plainly: **`/jobReport` is a billing write with no
authentication.** In the deployments this stack has run in, that is contained by
network topology — the endpoints are reachable only from the gateway, on an
internal network, and are not exposed by the public reverse proxy. That containment
is a deployment responsibility, not a property of the code.

If you deploy this stack, do all of the following:

1. **Restrict the paths at the reverse proxy.** `/jobAuthorizer`, `/jobReport`,
   `/userRoles` and `/metrics` should be reachable only from the gateway's address.
   Everything the dashboard needs is under `/api/*`, `/auth/token` and `/config`, so
   an allowlist on those four paths costs nothing in functionality.
2. **Do not put quantum-api on a public vhost without that restriction.** A
   dashboard-facing vhost and a gateway-facing one, or one vhost with per-path
   rules.
3. **Keep `BACKEND_SECRET` inside quantum-api.** It is a symmetric key: any holder
   can mint an `admin` platform token. It is never a machine-to-machine credential.
4. **Watch the audit log.** Unattributable reports appear as `reportingError` /
   `reportingWarning` entries in `action_logs`, which is the closest thing to a
   tamper signal available today.

Adding a real service identity for these endpoints — a client-credentials service
account, or per-caller credentials with scopes — is on the roadmap and is a
prerequisite for opening the backend to any client other than its own gateway.

---

## Implementing the contract in a new site plugin

A site plugin does not have to talk to quantum-api at all: the `SitePlugin`
protocol is the abstraction, and quantum-api is one implementation of what sits
behind it. To integrate a different portal, billing system or scheduler accounting
database, implement:

| Method | Contract |
|---|---|
| `authorize_job(username, project_name, extra_headers, timeout)` | Return a `JobAuthorizationResult`. `is_authorized=False` must carry a `status_code` and an `error_detail` that is safe to show a user. **Fail closed** on transport errors. |
| `send_initial_report(payload)` | Register a newly submitted job. Best-effort. |
| `report_job_async(job_id, payload)` / `report_job_sync(job_id, payload)` | Report progress and the terminal state. Must tolerate being called more than once for the same job. |
| `build_results_url(...)` | The user-facing URL for a job's results. |

Two properties the core relies on, and that your implementation must preserve:

- **Authorization fails closed, reporting fails open.** A reporting failure must not
  block a submission; an authorization failure must.
- **Terminal reports are idempotent.** The reporter will retry. If your backend
  bills, it must refuse to bill the same job twice.

See `docs/MODULARIZATION.md` in the gateway repository for the full protocol
definitions, and [Extending the stack](../development/extending.md).
