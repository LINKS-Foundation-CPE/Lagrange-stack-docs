# QC Gateway

An authenticating, auditing reverse proxy for quantum-computer HTTP APIs.
Python 3.11 · FastAPI · Uvicorn · EUPL-1.2 ·
[source on GitHub](https://github.com/LINKS-Foundation-CPE/QC-Gateway).

The gateway is the only component that talks to the machine and the only holder of
the machine's service credentials. Everything a production service needs and a
vendor API typically does not provide — authentication, role-based access control,
fairness limits, job accounting, artifact archival, metrics — is added here,
**without changing the API on the wire**.

## Transparent to existing SDKs

Paths, request and response bodies, status codes and headers pass through
unchanged. An existing `qiskit-iqm`, `cirq-iqm` or Qrisp script needs one change —
the server URL — and one addition — an OIDC bearer token. There is no wrapper SDK
to adopt, no translation layer to maintain, and no divergence to track between
what the gateway exposes and what the machine offers.

The vendor plugin makes this possible: it parses requests *for the gateway's own
bookkeeping* (shots, circuits, project, job type) and leaves the payload alone.

## What it does on each request

The pipeline is a single middleware, `proxy_and_capture`, and its order is
load-bearing: public-route bypass → deprecated route `410` → maintenance `503` →
route whitelist → authenticate → RBAC → parse submission → site authorization →
concurrency reservation → proxy → capture and report, with the reservation rolled
back if the upstream call fails.

[Life of a job](../architecture/job-lifecycle.md) covers the whole flow, the
failure modes and what is idempotent.

## Two processes, one image

| Process | Role |
|---|---|
| **proxy** | The request hot path. `uvicorn middleware.main:app`. |
| **reporter** | Background reconciliation: polls non-terminal jobs, uploads artifacts, releases concurrency counters, sends terminal reports. `python -m middleware.job_reporter`. |

They share Redis, the local PostgreSQL job log and the object store, and load the
same plugins. The reporter is not optional: without it, jobs never reach a
terminal state in the accounting system and nothing is ever billed.

## Operation modes

`MIDDLEWARE_MODE` selects how much of the pipeline runs — `production`,
`authentication`, `reporting`, `maintenance`. See
[the mode table](../architecture/job-lifecycle.md#the-gateways-operation-modes).
`maintenance` is what an operator switches on for planned downtime; it answers
everything with `503`.

## Plugins

Two `Protocol` contracts isolate everything that is not generic:

| Plugin | Owns | Selected by |
|---|---|---|
| `VendorPlugin` | Route table and role requirements, request/response parsing, upstream auth headers, terminal statuses, job-status polling, artifact URLs and classification, calibration handling, health endpoint | `VENDOR_PLUGIN` |
| `SitePlugin` | `authorize_job()`, the job-report senders, results-URL construction | `SITE_PLUGIN` |

The core may not import from `middleware.vendors.*` or `middleware.sites.*`, and
may not branch on a vendor or site name — this is stated as a non-negotiable rule
in the repository's `CLAUDE.md`. Data crossing the boundary travels as dataclasses
in `middleware/plugins/datatypes.py`, never as raw vendor-shaped dicts.

The shipped reference implementations are an **IQM vendor plugin** and a **site
plugin that integrates with quantum-api**. Writing your own is covered in
[Extending the stack](../development/extending.md) and, in full detail, in the
repository's `docs/MODULARIZATION.md`.

## The stack it runs in

The reference `docker-compose.yaml` brings up:

| Service | Role |
|---|---|
| nginx + Let's Encrypt (SWAG image) | TLS termination and virtual hosts |
| `fastapi-proxy` | the proxy process |
| `reporter` | the reconciliation worker |
| PostgreSQL 17 | local job log |
| Redis 7 | concurrency counters (append-only persistence) |
| RustFS | payload and artifact store |
| `rustfs-init` | one-shot bootstrap on every start: bucket, application user, anonymous and CORS policies |
| backup sidecars | off-site copies of the object store and the database volume |

Nothing in the code requires this exact arrangement: the gateway needs *a*
PostgreSQL, *a* Redis, *an* S3-compatible endpoint and *a* TLS terminator.

### The object store

Any S3-compatible store works, but three of its settings are load-bearing, and
the reference `rustfs-init` sets all three:

- **Anonymous read, by exact key only** (`s3:GetObject`, no `s3:ListBucket`).
  Artifact links and the results portal fetch objects without credentials; a
  listable bucket would expose every user's identity and job ids.
- **CORS for `GET` and `HEAD` from the results portal's origin** (the reference
  allows any origin, which grants nothing anonymous reads do not). The portal
  fetches every file from the store's host, a different origin: without the
  headers the browser blocks every request and results pages come up empty.
  MinIO sent such headers by default; RustFS, like most stores, sends none
  until the bucket has CORS rules.
- **An application user** limited to the bucket (`APP_USER`/`APP_PASSWORD`),
  which is all the gateway uses.

The gateway's S3 client connects to `MINIO_SERVER_URL`, the same public URL it
writes into artifact links. Newer gateway versions accept an optional
`S3_ENDPOINT_URL` for the client alone, for a deployment where the public URL is
not reachable from inside the containers — see
[Configuration](../reference/configuration.md#object-store-required-no-defaults).

A deployment still on MinIO can move to RustFS in place: the gateway's
`CHANGELOG.md` describes the two-step migration, during which RustFS serves
objects it has not copied yet from MinIO, so no link breaks.

### The vhosts it fronts

`configure_nginx.sh` renders one vhost per service from the templates in
`config/`, each named after the service rather than after one deployment's
domains, with the host names coming from `.env`. Two of them exist only when
configured, so a deployment that runs neither service simply has neither host:

- **`docs.<domain>`** redirects to `DOCS_URL`, with `/private` going to
  `INTERNAL_DOCS_URL`. Rendered only when `DOCS_URL` is set.
- **`overview.<domain>`** fronts a monitoring overview service the deployment
  runs beside the gateway, the way grafana is fronted, from `OVERVIEW_UPSTREAM`.
  Rendered only when that is set — **and the host must also be added to
  `EXTRA_DOMAINS`**, or the certificate does not cover it.

`config/fail2ban/jail.local` is generated the same way, from a template, so the
trusted ranges in `FAIL2BAN_IGNOREIP_EXTRA` are configuration rather than a
committed constant.

### A development stack without TLS certificates

`docker-compose.dev.yaml` plus `env.dev.example` run the gateway with no SWAG, no
CA-issued certificates and no FQDN: every published port speaks HTTPS behind a
`tls-proxy` nginx using a self-signed certificate shared with the development
identity provider. See `docs/DEV_DEPLOYMENT.md` in the repository.

## Configuration

Environment-driven, via Pydantic settings, split three ways so plugin settings
never leak into the core:

- core → `middleware/config.py`
- vendor → `middleware/vendors/<name>/config.py`
- site → `middleware/sites/<name>/config.py`

Full list with defaults: [configuration reference](../reference/configuration.md#qc-gateway).

## Development

```bash
pip install -r requirements-dev.txt
pre-commit install
ruff check . && ruff format --check . && mypy
pytest
```

All three checks must pass before a merge request. `mypy` is permissive for the
core and strict for `middleware/plugins/*` — do not weaken the plugin type hints.
Tests use `fakeredis`, so no Redis instance is needed.

The repository's own documents, in the order a new contributor should read them:

| Document | Audience |
|---|---|
| `README.md` | users and evaluators |
| `docs/DOCUMENTATION.md` | operators and integrators: request flow, configuration, plugin walk-through |
| `docs/MODULARIZATION.md` | plugin authors: interface contracts, using the generic modules standalone |
| `CONTRIBUTING.md` | development setup and workflow |
| `CLAUDE.md` | the architectural rules, stated as rules |
