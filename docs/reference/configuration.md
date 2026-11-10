# Configuration reference

Every component is configured entirely by environment variables. Nothing secret is
committed to any repository; each has an `env.example` / `.env.example` that
documents its keys.

## QC Gateway

Read by both the proxy and the reporter, which must be given the same
configuration. Split across three modules so plugin settings never leak into the
core: `middleware/config.py`, `middleware/vendors/<name>/config.py`,
`middleware/sites/<name>/config.py`.

### Core

| Variable | Default | Purpose |
|---|---|---|
| `VENDOR_PLUGIN` | `iqm` | Which vendor plugin to load. |
| `SITE_PLUGIN` | `spark` | Which site plugin to load: `spark`, or `sqed` for [Sqed](../hpc/index.md). |
| `AUTH_PLUGINS` | `keycloak` | The authentication chain, comma-separated: `keycloak`, `sqed`. A token carrying a plugin's prefix (`sqed_`) goes to that plugin alone; others are tried in order. |
| `STRICT_PREFIX_MODE` | `false` | Refuse any token without a known prefix, for a deployment that accepts scheduler tokens only. |
| `POLICY_PLUGIN` | `passthrough` | Admission policy: `passthrough` (the concurrency limits below) or `shot_budget` (concurrent shots scale with the SLURM licenses a job holds, `SHOTS_PER_LICENSE`, default `625000`). |
| `MIDDLEWARE_MODE` | `production` | `production`, `authentication`, `reporting`, `maintenance`. See [modes](../architecture/job-lifecycle.md#the-gateways-operation-modes). |
| `KEYCLOAK_ISSUER` | — | Expected `iss` of incoming tokens, e.g. `https://idp.example.org/auth/realms/cortex`. |
| `KEYCLOAK_JWKS_URL` | — | Where to fetch signing keys. Usually the issuer + `/protocol/openid-connect/certs`. |
| `AUDIENCE` | — | Expected `aud`, if you enforce one. |
| `MACHINE_URL` | — | Upstream machine base URL. |
| `UPSTREAM_TIMEOUT` | `30` | Seconds. Also used as the timeout for the authorization call. |
| `VERIFY_UPSTREAM_SSL` | `false` | **Set to `true` unless the machine has a self-signed certificate.** |
| `FRONTEND_URL` | — | This deployment's public host. Also whitelists the CORS origin. |
| `LOG_LEVEL` | `INFO` | |

### Object store (required — no defaults)

| Variable | Purpose |
|---|---|
| `MINIO_SERVER_URL` | S3 endpoint, as resolvable **by clients** — artifact links point here. There is deliberately no default: the uploader raises on startup if it is unset, because a wrong value only fails later, when someone follows a dead artifact link. |
| `BUCKET_NAME` | Bucket for payloads and artifacts. |
| `APP_USER`, `APP_PASSWORD` | Application credentials for that bucket. |
| `S3_ENDPOINT_URL` | *Optional, newer gateway versions.* Where the gateway's own S3 client connects, when that must differ from `MINIO_SERVER_URL` (which links keep using) — e.g. `http://rustfs:9000` on the Compose network, for a deployment whose public URL is unreachable from inside a container. Unset, the client uses `MINIO_SERVER_URL`. A value without `http://` or `https://` is ignored with a warning. |

The reference Compose stack additionally uses `MINIO_ROOT_USER`,
`MINIO_ROOT_PASSWORD` and `MINIO_STORAGE_PATH` to bring the object store up —
RustFS reads the `MINIO_*` names, which predate it — and
`AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` for tooling.

!!! note "`MINIO_INTERNAL_URL` is not a setting"
    Early versions' `env.example` carried `MINIO_INTERNAL_URL`, which nothing has
    read for a long time. One gateway version did read it, and an old
    `minio-job-data` value left in a deployment's environment sent its uploads to
    a host that no longer existed; the setting is `S3_ENDPOINT_URL` for that
    reason. A leftover `MINIO_INTERNAL_URL` line is harmless and can be deleted.

### Redis

| Variable | Default |
|---|---|
| `REDIS_HOST` | `redis` |
| `REDIS_PORT` | `6379` |
| `REDIS_DB` | `0` |
| `REDIS_PASSWORD` | — |

### Concurrency limits

| Variable | Default | Purpose |
|---|---|---|
| `MAX_CONCURRENT_SHOTS` | `2500000` | Per-user cap on `shots × circuits` in flight. |
| `MAX_CONCURRENT_SWEEPS` | `10` | Per-user cap on submissions in flight; used for sweeps, where shots cannot be counted. |

These are fairness knobs, and the right values depend on your machine's throughput.
Too high and one user can monopolise the queue; too low and legitimate batch work
gets `429`s.

### Reporter

| Variable | Default | Purpose |
|---|---|---|
| `JOB_REPORTER_INTERVAL` | `60` | Seconds between reconciliation passes. |
| `JOB_REPORTER_HTTP_TIMEOUT` | `30.0` | |
| `JOB_REPORTER_VERIFY_TLS` | — | Overrides TLS verification for reporter calls. |
| `JOB_REPORTER_MAX_ERRORS` | `10` | Consecutive errors before backing off. |
| `JOB_REPORTER_BACKOFF_FACTOR` | `2.0` | |
| `JOB_REPORTER_MAX_BACKOFF` | `300` | Seconds. |
| `METRICS_POLL_INTERVAL_SECONDS` | `5` | Prometheus gauge refresh. |

### Vendor plugin (IQM)

| Variable | Default | Purpose |
|---|---|---|
| `IQM_SERVER_TOKEN` | — | The machine's service token. **The gateway is its only holder.** |
| `CALIBRATION_POLL_INTERVAL` | `3600` | Seconds between calibration-run polls. |

### Site plugin

| Variable | Default | Purpose |
|---|---|---|
| `PORTAL_API_HOST` | `http://host.docker.internal:8500` | quantum-api base URL, as resolvable from the gateway container. |
| `BASE_DOMAIN` | — | Used to derive public URLs. |
| `JOBS_PORTAL_URL` | `https://jobs.${BASE_DOMAIN}` | Results-portal base URL. Set explicitly when there is no FQDN. |

### `sqed` auth plugin

Read only when `sqed` is in `AUTH_PLUGINS`: `SQED_TOKEN_PREFIX`,
`SQED_DEFAULT_ROLES`, `SQED_ROLES_LOOKUP_URL`, `SQED_ROLES_CACHE_TTL`,
`SQED_ROLES_VERIFY_TLS`. They are described, with the settings around them, in
[Deploying Sqed](../hpc/deployment.md#5-the-gateway-sqed-plugins).

### Deployment-stack variables

Used by the reference `docker-compose.yaml` and by `configure_nginx.sh` rather
than by the application itself:

| Variable | Purpose |
|---|---|
| `POSTGRES_PASSWORD` | Local job database. |
| `DASHBOARD_URL`, `JOB_PORTAL_API_URL` | Host names of the dashboard and backend vhosts. |
| `EXTRA_DOMAINS` | Additional certificate SANs. |
| `BACKUP_ENDPOINT` | Off-site backup target for the object store and database volume. |
| `DOCS_URL` | Where the `docs.<BASE_DOMAIN>` vhost redirects. The vhost is rendered only when this is set. |
| `INTERNAL_DOCS_URL` | Target of that vhost's `/private` path; defaults to `DOCS_URL`. |
| `FAIL2BAN_IGNOREIP_EXTRA` | Address ranges appended to fail2ban's `ignoreip`, on top of the RFC1918 ranges — typically the operators' own public range. **Quote it** if it lists more than one range: the deploy sources `.env`. |
| `OVERVIEW_UPSTREAM` | Full URL of a monitoring overview service the deployment runs beside the gateway, fronted at `overview.<BASE_DOMAIN>` the way grafana is. The vhost is rendered only when this is set. With the proxy on the host network a local service is `http://127.0.0.1:<port>`. **Also add `overview.<domain>` to `EXTRA_DOMAINS`**, or the new host answers with a certificate name mismatch. |

`configure_nginx.sh` renders the nginx vhosts and `config/fail2ban/jail.local`
from the templates in `config/`, substituting these values. The templates are
named after the service they front (`frontend`, `api`, `dashboard`, `store`,
`jobs`, `grafana`, `prometheus`, `status`, `overview`, `docs`), and each rendered file is
named after the host it answers on — so a deployment that does not run one of
those upstreams simply drops the template.

---

## quantum-api

### Required

| Variable | Purpose |
|---|---|
| `KEYCLOAK_BASE_URL` | Identity-provider base URL. The realm path is appended, so this is e.g. `https://idp.example.org/auth`. |
| `KEYCLOAK_REALM` | Realm name. |
| `BACKEND_SECRET` | **Symmetric** HS256 key for platform tokens. Random, long, and never shared with another service. |
| `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD` | Database credentials. |

The server refuses to start without the Keycloak variables, and the database layer
throws without user and database name.

### Optional

| Variable | Default | Purpose |
|---|---|---|
| `API_PORT` | `8500` | |
| `DB_HOST` | `localhost` (with a warning) | |
| `DB_PORT` | `5432` (with a warning) | |
| `CORS_ORIGIN` | — (permissive, with a warning) | Dashboard origin, **exact match including port**. Leaving it unset allows any origin — acceptable for local development only. |
| `TOKEN_DURATION` | `24h` | Platform-token lifetime. |
| `SSL_CERT`, `SSL_KEY` | — | Set both to terminate TLS in-process instead of at a proxy. |
| `LOG_LEVEL` | | Logger verbosity. |

### Deployment policy flags

Every one of these defaults to the previous behaviour, so upgrading a running
deployment without touching its `.env` changes nothing.

| Variable | Default | Effect |
|---|---|---|
| `SLOT_CONSTRAINED_RESERVATIONS` | `true` | Reservations must sit inside a pre-allocated slot of the project's organization. With `false`, `slot_id` becomes optional and the dashboard hides the slot views (it learns this from `GET /config`). |
| `ALLOW_FOREIGN_ORG_QUEUE_IN_SLOTS` | `true` | During another organization's slot, outside reservations, the open queue stays available to everyone. With `false`, submissions are restricted to the slot's organization and organizations referencing it. |
| `USERNAME_FORMAT` | `email` | Shape of the user identity. See below. |
| `JOB_PORTAL_ONLY` | `false` | Serve a job viewer only. The management API and `/jobAuthorizer` are not mounted, nothing is billed, and a job report naming an unknown project creates it. For a site whose scheduler owns access and accounting, see [Deploying Sqed](../hpc/deployment.md#3-quantum-api-cluster-identities-job-portal-only). |
| `PULSE_ACCESS_OIDC_ROLE` | `pulla_user` | The realm role that seeds `users.pulla_user` when a user is first registered. |

#### `USERNAME_FORMAT`

The user identity — `users.email`, reported by the gateway as a job's `username` —
is either:

- **`email`** (default) — must be an e-mail address;
- **`any`** — any whitespace-free string, for deployments whose identities are not
  e-mail addresses (HPC cluster account names, for instance).

!!! danger "Frozen at first deployment"
    The value is written to the `system_configs` table on first boot. A later boot
    with a **conflicting** value **refuses to start**, because changing it once
    users exist would invalidate existing identities and silently change billing
    attribution. An unset variable adopts the frozen value.

    To change it deliberately on a deployment that can tolerate it: stop the
    service, delete the `username_format` row from `system_configs`, set the new
    value, and start. You are responsible for whatever existing identities that
    invalidates.

Decide this **before** the first boot of a new deployment.

#### `PULSE_ACCESS_OIDC_ROLE`

The realm role that seeds sweep ("Pulse") access. When a token exchange *creates* a
user row, the backend sets `users.pulla_user` if the token carries this role.
Configurable because realms name it differently; the rest of the realm-role →
platform-role mapping is currently fixed in code.

It is a **default, not a mirror**. After the row exists the flag belongs to the
platform: an administrator grants or revokes it in the dashboard, and no later
login changes it. That is what makes revocation possible at all — a per-login
mirror would restore the flag for anyone whose realm role still named them. The
converse is that an identity which gains the realm role after its first login is
granted Pulse access by an administrator, not automatically.

### Compose-only variables

Read by the reference `docker-compose.yaml`, not by the application. Both the
database-backup sidecar and pgAdmin are behind **compose profiles** and do not
start unless selected, so a development instance can ignore this section
entirely.

| Variable | Purpose |
|---|---|
| `COMPOSE_PROFILES` | Which optional services to start: `backup`, `pgadmin`, `test`. Compose reads it from `.env`, so it is deployment configuration rather than something the deploy command has to pass. |
| `AWS_ENDPOINT` | S3 endpoint (`host:port`) for the off-site database backup. Needed only with the `backup` profile. |
| `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` | Credentials for it. |
| `PGADMIN_DEFAULT_EMAIL`, `PGADMIN_DEFAULT_PASSWORD` | The opt-in pgAdmin profile. Never expose it on a real deployment. |

!!! note "Optional services fail by being absent, not by complaining"
    Compose interpolates the whole file eagerly, so a "required variable"
    (`${VAR:?…}`) would break every compose command in an environment that has
    no S3 endpoint — including a service you did not ask for. Profiles are the
    mechanism instead, which means a deployment that *meant* to back up but
    forgot the profile gets no backup container rather than an error. Check
    `docker compose ps`, and monitor the bucket.

---

## quantum-dashboard

All configuration is `VITE_*` and is **baked in at build time** — changing one
requires a rebuild, not a restart.

| Variable | Purpose |
|---|---|
| `VITE_API_URL` | quantum-api REST base, e.g. `https://api.example.org/api`. |
| `VITE_TOKEN_URL` | Token exchange, e.g. `https://api.example.org/auth/token`. |
| `VITE_KEYCLOAK_URL` | Identity-provider base URL. |
| `VITE_KEYCLOAK_REALM` | Realm. |
| `VITE_KEYCLOAK_CLIENTID` | Public client for the dashboard. |

Optional, and each feature is simply absent when its variable is unset — a
deployment without a monitoring stack has no monitoring panels rather than empty
frames:

| Variable | Purpose |
|---|---|
| `VITE_MACHINE_STATUS_URL` | Status page (Kener). Drives the **Machine Status** panel, which reads that site's per-monitor badges. |
| `VITE_MACHINE_METRICS_URL` | Monitoring overview page — the `overview.<domain>` vhost above. Embedded as the **Machine Metrics** panel. |
| `VITE_MACHINE_STATUS_MONITORS` | Which monitors the status panel shows, as `tag` or `tag:Label`, comma separated. Unset falls back to the status site's aggregate pseudo-monitor. |
| `VITE_DOCS_URL` | User documentation. A sidebar link opening in a new tab, not a panel. |

!!! warning "Two traps with these four"
    **Do not quote `VITE_MACHINE_STATUS_MONITORS`.** Vite parses `.env` with
    dotenv, which takes the whole line, so spaces in labels are safe and quote
    characters would be kept literally — as would quotes set through a CI
    variable.

    **The panels are cross-origin frames.** Whatever serves those pages must not
    send `X-Frame-Options`, and any `Content-Security-Policy: frame-ancestors` it
    does send must name the dashboard's origin. The status panel sidesteps this
    for its own data by reading badge **images**, which are exempt from CORS; the
    metrics panel is a real frame.

    And because all of this is baked in at build time, a deployment that forgets
    them gets a build with no monitoring panels and no error anywhere.

---

## Cross-component consistency

Most misconfigurations are a mismatch between two components rather than a wrong
value in one. The pairs that must agree:

| Must match | Where |
|---|---|
| Realm and issuer URL | `KEYCLOAK_ISSUER` / `KEYCLOAK_JWKS_URL` (gateway), `KEYCLOAK_BASE_URL` + `KEYCLOAK_REALM` (backend), `VITE_KEYCLOAK_URL` + `VITE_KEYCLOAK_REALM` (dashboard), and the realm that actually issues the tokens |
| Dashboard origin | `CORS_ORIGIN` (backend) and the origin the dashboard is served from |
| Backend URL | `PORTAL_API_HOST` (gateway, internal) and `VITE_API_URL` / `VITE_TOKEN_URL` (dashboard, public) — these are usually *different* addresses for the same service |
| Object-store URL | `MINIO_SERVER_URL` (gateway) must be resolvable by **browsers**, not just by containers, because artifact links point there |
| Results-portal URL | `JOBS_PORTAL_URL` (gateway) ends up stored in job records and followed by browsers |
| Identity shape | `USERNAME_FORMAT` (backend) and what the identity provider puts in the `email` claim |
| Embedded monitoring | `VITE_MACHINE_STATUS_URL` / `VITE_MACHINE_METRICS_URL` (dashboard) and the vhosts that serve them — including `OVERVIEW_UPSTREAM` plus `overview.<domain>` in `EXTRA_DOMAINS` (gateway) |

A useful rule of thumb: the gateway's view of the backend is **internal**;
everything the backend stores in a job record is **public**. Getting those two
backwards produces a stack that works until someone clicks an artifact link.
