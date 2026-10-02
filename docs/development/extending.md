# Extending the stack

The four extensions you are most likely to need, in rough order of how often they
come up.

## Supporting a different machine: a vendor plugin

A vendor plugin is everything the gateway needs to know about one vendor's HTTP
API. Writing one is the intended way to put this stack in front of a machine that
is not IQM-based.

```bash
middleware/vendors/<name>/
├── __init__.py
├── plugin.py          # the class satisfying VendorPlugin
├── config.py          # vendor settings + default route definitions
├── request_parser.py  # submission payload → JobSubmission
├── response_parser.py # upstream response → SubmissionResult
├── headers.py         # upstream auth injection
└── job_status.py      # status polling
```

Register it and select it:

```python
# middleware/plugins/loader.py
_VENDOR_REGISTRY["myvendor"] = "middleware.vendors.myvendor.plugin.MyVendorPlugin"
```

```bash
VENDOR_PLUGIN=myvendor
```

### The contract

| Method | Must return / do |
|---|---|
| `get_routes_config()` | A `RoutesConfig`: role-gated routes as `(path, method) → [roles]`, plus the logged, deprecated and public route lists. **Nothing not listed here is reachable.** |
| `parse_submission_request(body, path)` | A `JobSubmission` — shots, circuits, project, job type. **Must not modify the body.** |
| `parse_submission_response(response_text)` | A `SubmissionResult` — the job id and the artifact types this job will produce. |
| `build_upstream_headers(base_headers)` | The headers to send upstream, with the machine's service credential injected. |
| `get_terminal_statuses()` | The status strings that mean "done". |
| `get_job_status(...)` | A `JobStatusResult` — authoritative status, timeline, available artifacts. |
| `get_artifact_url(job_id, type)`, `get_payload_url(job_id)` | Relative URL patterns for fetching them. |
| `classify_artifacts(status, available)` | Which artifacts to fetch for a given terminal status — a failed job usually wants different artifacts than a successful one. |
| `get_health_endpoint()` | The vendor's health path. |
| `get_calibration_poll_interval()`, `process_calibration_runs(...)`, `enrich_artifacts_with_calibration(...)` | Optional. Return a no-op interval and do nothing if the vendor has no such concept. |

### Getting it right

- **The route table is the security boundary.** There is no default-allow: a path
  you forget is a path nobody can use, which is the safe failure. Be deliberate
  about which methods each route accepts, and use the deprecated list to positively
  retire endpoints you do not want reachable (they answer `410`).
- **Never rewrite the request.** Parsing is for bookkeeping. The moment the plugin
  edits a payload, the "unmodified SDK" property is gone and you own a translation
  layer forever.
- **Cross the boundary with dataclasses**, not dicts. Extend the types in
  `middleware/plugins/datatypes.py` rather than widening a method signature with
  ad-hoc parameters.
- **Keep vendor settings in your own `config.py`.** Do not add them to
  `middleware/config.py`.
- **Type hints are strict for `middleware/plugins/*`.** `mypy` will hold you to it.

## Integrating a different portal: a site plugin

A site plugin is how the gateway asks permission and reports results. The shipped
one talks to quantum-api; yours could talk to a billing system, a scheduler
accounting database, or nothing at all.

```bash
middleware/sites/<name>/
├── __init__.py
├── plugin.py
├── authorization.py
├── reporting.py
└── config.py
```

```python
_SITE_REGISTRY["mysite"] = "middleware.sites.mysite.plugin.MySitePlugin"
```

```bash
SITE_PLUGIN=mysite
```

| Method | Contract |
|---|---|
| `authorize_job(username, project_name, extra_headers, timeout)` | A `JobAuthorizationResult`. On refusal, set `status_code` and an `error_detail` that is safe to show a user — the gateway returns it verbatim. **Fail closed** on transport errors. |
| `send_initial_report(payload)` | Register a submitted job. Best-effort. |
| `report_job_async(job_id, payload)` / `report_job_sync(job_id, payload)` | Progress and terminal reports; the async variant is used on the request path, the sync one by the reporter. **Must tolerate repeats.** |
| `build_results_url(index_url, job_type)` | The user-facing results URL. |

Two invariants the core depends on:

- **Authorization fails closed; reporting fails open.** A reporting failure must not
  block a submission. An authorization failure must.
- **Terminal reports are idempotent.** The reporter retries. If your backend bills,
  it must refuse to bill the same job twice — see how quantum-api does it in
  [the contract reference](../reference/machine-endpoints.md#put-jobreportjobid).

The minimum viable site plugin authorizes everything and reports nowhere. That is a
legitimate configuration: it turns the stack into an authenticating, artifact-
archiving proxy with no accounting.

## Adding a resource to quantum-api

Mechanical, once you follow the layering. For a resource `widgets`:

1. **Model** — `src/models/widget.ts`, registered in `src/models/index.ts`.
2. **Migration** — a new `sequelize-cli` file in `migrations/`, in the **same
   commit** as the model change. Additive and nullable if it must apply to a live
   database.
3. **Repository** — `src/repositories/widget.repository.ts`, the only place that
   touches the model.
4. **Service** — `src/services/widget.service.ts`, all the business logic. If it
   moves budget, it appends a `budget_events` row in the same transaction.
5. **Schema** — `src/schemas/widget.schema.ts`, zod, for body and query.
6. **Controller** — thin: parse, call the service, respond.
7. **Route** — `src/routes/widget.routes.ts`, with `validate(schema)` and the right
   authorization middleware (`authorizeByRole`, `authorizeByProjectAdmin`,
   `restrictToOrganization`, `restrictToUser`).
8. **Mount** — in `src/app.ts`, under `/api/`.
9. **`Content-Range`** on the list endpoint. Not optional: the dashboard's data
   provider requires it.
10. **`api.yml`** — add the paths and schemas, so `/api-docs` stays truthful.
11. **Tests** — Jest, mirroring the source layout.

Then, in the dashboard: a folder under `src/pages/widgets/` with the
List/Show/Edit/Create components, a `<Resource>` in `App.tsx`, and a `canAccess`
rule in `authProvider.ts`.

## Adding a deployment flag

The rule: **a new flag's default must reproduce the current behaviour**, so that a
new version can be deployed onto a running system without changing what it does.

1. Read it in `src/config/policy.ts` (or a dedicated module for something with
   lifecycle, like `usernamePolicy.ts`), never with a scattered
   `process.env.X` at the point of use.
2. Document it in `.env.example` with the default and the consequence of changing
   it.
3. Note it in `CHANGELOG.md`, flagged **Breaking:** only if an operator must act on
   upgrade.
4. If the **dashboard** must react to it, expose it in the public `GET /config`
   response — booleans only, nothing secret — and read it via `useConfig()`. Do not
   add a build-time `VITE_*` variable for something that varies per deployment: it
   would mean one dashboard build per deployment.
5. If it must not change after users exist, freeze it in `system_configs` the way
   `USERNAME_FORMAT` does: persist on first boot, refuse to start on a conflict, and
   document the unfreeze procedure.

Existing flags to copy from: `SLOT_CONSTRAINED_RESERVATIONS` (a rule change, with a
UI consequence), `ALLOW_FOREIGN_ORG_QUEUE_IN_SLOTS` (a rule change with none),
`PULSE_ACCESS_OIDC_ROLE` (a name that varies per identity provider),
`USERNAME_FORMAT` (frozen).

## Using the gateway's generic modules on their own

`authorization`, `concurrency` and `reporting` are written to be usable outside the
FastAPI application — the `ConcurrencyLimiter` takes a Redis client and limits and
nothing else, `RoleAuthorizationChecker` takes a route table. If you need per-user
fairness limits or role-gated routing in a different service, they are worth reading
before writing your own. `docs/MODULARIZATION.md` in the gateway repository has
direct-use examples.
