# Components and repositories

## Core components

Three services make up the platform. Each is an independent repository, released
independently under EUPL-1.2, and deployable on its own.

### QC Gateway

:material-language-python: Python 3.11 · FastAPI · Uvicorn ·
[public source](https://github.com/LINKS-Foundation-CPE/QC-Gateway)

The authenticating, auditing reverse proxy in front of the machine. It is the
only component that talks to the quantum computer, and the only holder of the
machine's service credentials.

```
middleware/
├── main.py              # FastAPI app + proxy_and_capture (the hot path)
├── config.py            # core Pydantic Settings
├── authentication.py    # OIDC/JWT validation (JWKS, RS256) → User
├── authorization.py     # RoleAuthorizationChecker (role → route RBAC)
├── concurrency.py       # ConcurrencyLimiter (Redis counters, fail-open)
├── db.py                # local job log (PostgreSQL)
├── minio.py             # S3Uploader (artifact store)
├── job_capture.py       # submitted-payload archiving
├── artifacts.py         # artifact / timeline / index upload helpers
├── reporting.py         # generic JobReporter, SyncJobReporter
├── job_reporter.py      # background reconciliation worker (separate process)
├── job_counters.py      # Redis client + Prometheus metrics worker
├── utils.py             # generic response builders
├── plugins/
│   ├── interfaces.py    # VendorPlugin, SitePlugin (typing.Protocol)
│   ├── datatypes.py     # dataclasses crossing the plugin boundary
│   └── loader.py        # string-keyed registry, importlib resolution
├── vendors/iqm/         # reference vendor plugin
└── sites/<site>/        # reference site plugin
```

Two processes run from the same image: the **proxy** (the request hot path) and
the **reporter** (background reconciliation). They share Redis, PostgreSQL and
the object store.

Deep dives live with the code: `docs/DOCUMENTATION.md` (request flow,
configuration, plugin walk-through) and `docs/MODULARIZATION.md` (plugin
contracts, using the generic modules standalone). See also
[QC Gateway](../components/qc-gateway.md).

### quantum-api

:material-language-typescript: Node.js · TypeScript (ESM) · Express 5 ·
Sequelize 6 · PostgreSQL · zod · Jest ·
[public source](https://github.com/LINKS-Foundation-CPE/Lagrange-API)

The system of record. Everything the platform knows about who may run what, and
what they ran, lives here.

```
src/
├── app.ts               # Express application wiring (authoritative mount list)
├── server.ts            # process entrypoint
├── config/              # database, logger, constants, error types, policy flags
├── routes/              # HTTP route definitions
├── middleware/          # authenticateJWT, RBAC, validate (zod), parseQuery
├── controllers/         # thin request handlers
├── services/            # business logic (authorization, billing, reservations)
├── repositories/        # data access over Sequelize models
├── models/              # Sequelize model definitions
├── schemas/             # zod validation schemas
├── mappers/             # DTO/entity mapping
├── types/ · utils/      # shared types, generic helpers
migrations/              # sequelize-cli migrations
api.yml                  # OpenAPI 3.1 description, served at /api-docs
```

The layering is enforced by review, not by tooling: requests flow
`routes → middleware → controllers → services → repositories → models` and never
skip a layer. See [quantum-api](../components/quantum-api.md) and the
[HTTP API reference](../reference/quantum-api-http.md).

### quantum-dashboard

:material-react: React 19 · TypeScript · Vite · react-admin 5 · MUI ·
[public source](https://github.com/LINKS-Foundation-CPE/Lagrange-Dashboard)

The management UI. A single-page application whose resources map one-to-one onto
quantum-api entities, plus a handful of custom pages (landing dashboard,
calendars, self-service profile).

```
src/
├── App.tsx              # <Admin> with one <Resource> per backend entity
│                        #   + <CustomRoutes> for calendars, profile, my-*
├── Layout.tsx           # app shell
├── CustomAppBar.tsx     # top bar
├── CustomMenu.tsx       # role-sectioned navigation
├── authProvider.ts      # OIDC login (ra-keycloak); canAccess() per role
├── dataProvider.ts      # ra-data-simple-rest + platform-token httpClient
├── fetchBackendTokenOnce.ts
├── hooks/ · services/   # token exchange and storage, /config fetch
└── pages/<resource>/    # List / Show / Edit / Create per entity
```

See [quantum-dashboard](../components/quantum-dashboard.md).

## What the stack expects from its environment

None of these are implemented by the stack; all are standard components.

| Dependency | Used by | Purpose |
|---|---|---|
| **OIDC identity provider** | gateway, backend, dashboard | Authenticates users; issues the JWTs both services validate via JWKS. Keycloak in every deployment so far — see [Identity provider](../components/identity-provider.md). |
| **PostgreSQL** ×2 | gateway, backend | Gateway job log; backend system of record. Separate databases, ideally separate instances. |
| **Redis** | gateway | Concurrency counters and job-state bookkeeping. Append-only persistence is enough; it holds no authoritative state. |
| **S3-compatible object store** | gateway | Submitted payloads, job artifacts, timelines, results index. RustFS in the reference deployment (MinIO before it). It must serve objects anonymously by key, refuse anonymous listing, and send CORS headers — see [QC Gateway](../components/qc-gateway.md#the-object-store). |
| **TLS-terminating reverse proxy** | all | Public vhosts and certificates. Any proxy works; the reference deployment uses nginx. |
| **Quantum computer** | gateway | The upstream. Reachable only from the gateway, with a service token. |

Optional, and not required to run the stack: Prometheus and Grafana (the gateway
exposes metrics), and an off-site backup target.

## Auxiliary repositories

Around the three core components sit smaller repositories. They are listed here
because the documentation refers to them; not all are public.

| Repository | What it is |
|---|---|
| **portable-deployment** ([Lagrange-stack-portable](https://github.com/LINKS-Foundation-CPE/Lagrange-stack-portable)) | One-command deployment of the entire stack on a single Linux host with a self-signed certificate and no DNS. The fastest way to get a working system — see [Getting started](../getting-started/portable-deployment.md). |
| **prolog-epilog** ([Lagrange-prolog-epilog](https://github.com/LINKS-Foundation-CPE/Lagrange-prolog-epilog)) | SLURM prolog/epilog scripts that mint per-job tokens for the gateway's `sqed` plugins. Only used by [Sqed](../hpc/index.md), the HPC integration. |
| **keycloak-deployment** | Deployment configuration for the identity provider, plus an idempotent realm-bootstrap script that creates the realm, clients, roles and test users the stack expects. |
| **lagrange-billing** | Reporting application that reads the accounting tables and produces usage/billing reports from the budget-event ledger. |
| **statuspage** | Public status page for a deployment. |
| **examples** | User-facing example notebooks (hello world, applications) validated against a live deployment. |

## Cross-component contracts

Everything that crosses a component boundary is one of the following. There are
no shared libraries and no shared database between components.

| Contract | Between | Stability |
|---|---|---|
| **Vendor HTTP API** | client SDK → gateway → machine | Defined by the vendor. Passed through verbatim; the gateway adds no paths of its own on this surface. |
| **`POST /jobAuthorizer`** | gateway → backend | Pre-submission authorization. See [contract](../reference/machine-endpoints.md#post-jobauthorizer). |
| **`POST /jobReport`, `PUT /jobReport/{jobid}`** | gateway and reporter → backend | Job creation and terminal update; the `PUT` is what bills. See [contract](../reference/machine-endpoints.md#post-jobreport). |
| **`GET /userRoles/{username}`** | gateway → backend | Per-user platform roles, for deployments where the token itself carries no roles. See [contract](../reference/machine-endpoints.md#get-userrolesusername). |
| **`POST /auth/token`** | dashboard → backend | Exchanges an OIDC token for the backend platform token. |
| **`/api/*` + `Content-Range`** | dashboard → backend | react-admin "simple REST" convention: `range`, `sort`, `filter` query parameters, total count in the response header. Load-bearing — any new list endpoint must honour it. |
| **`GET /config`** | dashboard → backend | Public, unauthenticated subset of deployment flags, so the UI can adapt (e.g. hide slot views when reservations are not slot-constrained). |
| **Redis key namespaces** | gateway proxy ↔ reporter | Internal to the data plane. |

## Repository conventions

All three core repositories follow the same conventions, which makes moving
between them cheap:

- **`README.md`** for users and evaluators, **`CONTRIBUTING.md`** for the
  development workflow, **`CLAUDE.md`** as a short orientation document stating
  the architectural rules (also the fastest way for a new human contributor to
  learn what not to do), **`CHANGELOG.md`** in Keep-a-Changelog format with
  CalVer versions.
- **EUPL-1.2** in `LICENSE`.
- Linting and type-checking gates: `ruff` + `mypy` for the gateway, ESLint +
  Prettier + TypeScript for the Node/React repositories, with `pre-commit`
  configured where applicable.
- Development happens on a private GitLab; public releases are published to
  GitHub through a curated orphan `public` branch, so internal history never
  reaches the mirror. The procedure is documented in each repository's
  `CLAUDE.md` and summarised in
  [Release and publishing](../development/releases.md).
