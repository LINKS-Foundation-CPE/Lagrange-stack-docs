# Troubleshooting

Symptoms mapped to causes, grouped by where the failure actually lives. The
diagnostic order that saves the most time is: **certificate → origin → identity →
tenancy → reporting**.

## Login and the dashboard

| Symptom | Cause and fix |
|---|---|
| Browser login hangs with no error | On a self-signed deployment, the identity-provider origin has not been accepted yet. Visit `https://<idp>` once and accept the certificate. Same for the backend origin. |
| Dashboard loads, then every API call fails | Either the backend's certificate is not accepted in this browser, or `CORS_ORIGIN` in the backend does not match the dashboard's origin **exactly** (scheme, host and port). |
| Login succeeds but the sidebar is almost empty | Expected for a plain user. Administrative items appear only for `platform-admin` / `platform-ro-admin`, organization managers and project admins — see [the role matrix](../architecture/identity.md#resource-authorization-in-the-backend). |
| `401` from `/api/*` shortly after logging in | The platform token expired (`TOKEN_DURATION`, default 24 h) or `BACKEND_SECRET` was rotated. Log in again. |
| Changing a `VITE_*` variable had no effect | Dashboard configuration is baked in at build time. Rebuild. |

## Token and identity

| Symptom | Cause and fix |
|---|---|
| `401` from the gateway with a token that worked a minute ago | Access tokens are short-lived. Re-fetch. |
| `401` from the gateway with a fresh token | `KEYCLOAK_ISSUER` / `KEYCLOAK_JWKS_URL` point at a different realm than the token was issued by, or the issuer URL differs by a trailing path (`/auth`). |
| `POST /auth/token` returns `401` for a valid token | The backend cannot reach the JWKS endpoint, or `KEYCLOAK_BASE_URL` + `KEYCLOAK_REALM` do not compose the realm URL you expect. |
| A user logs in but the backend has no row for them | It should be created automatically on first token exchange. If not, the exchange itself is failing — check the backend log. |
| A user exists twice, or the wrong row is matched | Identities are matched by `sub` first, then by e-mail once (which then attaches the `sub`). Two rows with the same person's e-mail in different cases will not merge; the match is exact. |
| Server refuses to start: `USERNAME_FORMAT … conflicts with the value frozen at first deployment` | Exactly what it says — the setting is frozen in `system_configs` on first boot. Either remove the variable or follow the documented unfreeze procedure. See [`USERNAME_FORMAT`](../reference/configuration.md#username_format). |

## Submissions

| Symptom | Cause and fix |
|---|---|
| `403` with no message body | The route is not in the vendor plugin's route table (nothing is allowed by default), or the token lacks the role that route requires. |
| `403 user not found in database` | The identity has never exchanged a token and was never created by an administrator. Log in to the dashboard once, or create the user. |
| `403 no project provided and no default project defined` | Set a default project for the user, or pass `project_name` in the submission. |
| `403 Project '<name>' not found` / `is not member of project` | The project name does not exist (names are unique and exact) or the user is not in it. |
| `403 Project … already expired` / `not yet started` | The project's validity window excludes now. |
| `403 Slot reserved to another project` | A reservation is active for a different project. Reservations are exclusive for their whole duration. |
| `403 Slot reserved to another organization` | This deployment runs with `ALLOW_FOREIGN_ORG_QUEUE_IN_SLOTS=false` and another organization's slot is active. |
| `403 No budget available for project …` | `remaining_budget` is not positive. Top up, transfer from the vault, or use a `free_queue` project. |
| `429` | Per-user concurrency limit. Wait for in-flight jobs to finish; tune `MAX_CONCURRENT_SHOTS` / `MAX_CONCURRENT_SWEEPS` if the limits are wrong for your machine. |
| Every submission is refused, `403`, right after the backend restarted | Authorization fails closed by design: if the gateway cannot reach `/jobAuthorizer`, nothing is admitted. Check backend reachability from the gateway container. |
| `SSLCertVerificationError` in Python | `REQUESTS_CA_BUNDLE` is not set in *that* process. Notebook kernels do not inherit shell exports made after they started. |
| Circuit fails `compilation_started → failed` | A machine-side compilation error, most often two-qubit gates that do not respect the coupling topology. Let the transpiler map the circuit. |

## Jobs, artifacts and billing

| Symptom | Cause and fix |
|---|---|
| Job ran on the machine but never appears in the dashboard | The initial `POST /jobReport` failed. Check the backend audit log for a `reportingError`; the usual causes are an unknown username or an unknown project name. |
| Job stuck non-terminal in the dashboard forever | The reporter is not running or cannot reach the machine. Check its container; it catches up on restart. |
| Job is terminal but nothing was billed | Work through the legitimate exemptions: `free_queue` project, `usedReservation` flag, no execution window in the report, or the job was already finalised. Otherwise the terminal `PUT` never arrived. |
| Billed amount looks wrong | The amount is exactly the machine's reported `execution_end − execution_start`. Compare against the job's timeline artifact; the platform does not compute it. |
| A project's budget went negative | Expected. The pre-submission check is "budget > 0", not "enough for this job", so a long job can overshoot. Top up or write off. |
| Artifact links are dead | `MINIO_SERVER_URL` or `JOBS_PORTAL_URL` resolve to something the browser cannot reach. They are *public* URLs, not internal ones. |
| `SignatureDoesNotMatch` from the object store | A proxy rewrote the `Host` header, dropping the port. Pass it through unchanged. |
| Results portal pages are empty; the browser console shows blocked cross-origin requests | The object store sends no CORS headers. Give the bucket a CORS rule for `GET`/`HEAD` — see [The object store](../components/qc-gateway.md#the-object-store). |
| Submissions fail with `500` but the jobs run on the machine; the gateway log shows name-resolution or connection errors towards the store | The gateway cannot reach the object store to archive the payload. `MINIO_SERVER_URL` must also be reachable from inside the gateway's containers — or set `S3_ENDPOINT_URL` to an address that is. |
| Concurrency counters seem stuck high | Counters carry a TTL (default 1 h) precisely so this self-heals. If it persists, the reporter is not releasing them — check that it is running. |

## Deployment and infrastructure

| Symptom | Cause and fix |
|---|---|
| Port already in use at install time | A previous stack is still running. Tear it down first. |
| Keycloak exits with `password authentication failed` | Its database password and the Postgres password must match in the same env file. |
| Everything answers `503` | The gateway is in `maintenance` mode. Switch `MIDDLEWARE_MODE`. |
| Gateway starts but immediately fails on a missing setting | The object-store settings (`MINIO_SERVER_URL`, `BUCKET_NAME`, `APP_USER`, `APP_PASSWORD`) have no defaults and are required. |
| Deprecated vendor endpoint returns `410 Gone` | Deliberate: the vendor plugin retires endpoints the platform does not want reachable. |

## Where to look

| Question | Where |
|---|---|
| Was the submission authorized, and why not? | Gateway logs (the `/jobAuthorizer` response is logged), and the `403` message returned to the client |
| Did a report reach the backend? | Backend `action_logs` — `/api/logs` as a platform admin — for `reportingError` / `reportingWarning` |
| What was actually submitted? | The archived payload in the object store, keyed by job id |
| What did the machine do? | The job's timeline artifact, uploaded by the reporter |
| How much was billed and when? | `budget_events` — `GET /api/projects/{id}/transactions` |
| Who changed what in the platform? | Backend `action_logs` |
| Queue depth and per-user load | The gateway's Prometheus metrics |
