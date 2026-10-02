# quantum-api

The platform backend and system of record. Node.js · TypeScript (ESM) ·
Express 5 · Sequelize 6 over PostgreSQL · zod · Jest · EUPL-1.2 ·
[source on GitHub](https://github.com/LINKS-Foundation-CPE/Lagrange-API).

quantum-api models *who may run what, against which budget, and what they ran*.
It never talks to quantum hardware: if a change involves a device or a vendor API,
it belongs in the [gateway](qc-gateway.md) instead.

## What it owns

- **Identity and tenancy** — users, organizations, projects, membership, and the
  organization- and project-scoped roles derived from them.
- **Budgets and billing** — per-project budgets in QPU-milliseconds, debited by
  budget events, with the ledger that reports are built from.
- **Slots and reservations** — organization allocations and exclusive project
  bookings, with configurable placement policy.
- **Job authorization and reporting** — the two endpoints the gateway calls to ask
  permission and to record what happened.
- **Project tags** — an admin-defined controlled vocabulary of project labels,
  assignable by the project's own administrators.
- **Billing reports** — the per-period aggregates a *consuntivo* is built from,
  computed here rather than by exporting the database.
- **Announcements, notifications and an audit log.**

Semantics are in [Tenancy, budgets and billing](../architecture/accounting.md);
the endpoint-by-endpoint surface is the
[HTTP API reference](../reference/quantum-api-http.md).

## Architecture

Strict one-directional layering:

```
routes/  →  middleware/  →  controllers/  →  services/  →  repositories/  →  models/
(HTTP)      (auth, RBAC,     (thin HTTP      (business      (data access)    (Sequelize)
             validation,      adapters)       logic)
             query parsing)
```

Plus `schemas/` (zod), `mappers/`, `config/`, `types/`, `utils/`, and
`migrations/` for `sequelize-cli`.

The rules that keep it honest:

1. **No layer skipping.** A controller never queries a model; a repository never
   holds business rules.
2. **Validation is declarative.** Request bodies and queries are validated by zod
   schemas applied as middleware, never hand-rolled in a controller.
3. **Schema changes arrive with their migration** in the same change. Released
   migrations are never edited.
4. **Balances move only through the ledger.** Never mutate `remaining_budget`
   without appending the `budget_events` row that explains it, in the same
   transaction.
5. **New deploy-time behaviour ships behind a flag whose default preserves the
   previous behaviour**, so a new version can be rolled onto a running deployment
   without changing what it does.

## HTTP surface

The authoritative mount list is `src/app.ts`:

| Mount | Auth | Purpose |
|---|---|---|
| `/api/users`, `/api/projects`, `/api/organizations`, `/api/slots`, `/api/reservations`, `/api/jobs`, `/api/logs`, `/api/budgetEvents`, `/api/announcements`, `/api/activeAnnouncements`, `/api/notifications`, `/api/projects_users`, `/api/tags` | platform token | CRUD resources consumed by the dashboard |
| `/api/own` | platform token | Self-service: the caller's own projects and default project |
| `/api/reports` | platform token | Billing aggregates for a period; platform admins, organization managers and auditors |
| `/auth/token` | OIDC token | Token exchange → platform token |
| `/jobAuthorizer` | OIDC token (see note) | Machine endpoint: may this submission run? |
| `/jobReport` | — (see note) | Machine endpoint: record and finalise a job; the terminal call bills |
| `/userRoles/{username}` | — (see note) | Machine endpoint: platform roles for an identity |
| `/config` | public | Client-safe subset of deployment flags, for the dashboard |
| `/metrics/jobs` | — | Prometheus series: job counters, and per-organization, per-project, per-tag and per-user aggregates |
| `/api-docs` | public | Swagger UI over `api.yml` |

!!! warning "The machine endpoints are internal by deployment, not by authentication"
    `/jobReport`, `/userRoles` and `/metrics/jobs` have **no authentication** in the
    current release, and `/jobAuthorizer` verifies the *submitting user's* token
    rather than a service identity. They are safe only as long as the deployment
    restricts them to the gateway at the network or proxy layer. Read
    [the contract reference](../reference/machine-endpoints.md#authentication-status-and-hardening)
    before exposing quantum-api to anything but its own gateway.

## The react-admin contract

List endpoints implement the react-admin "simple REST" convention, and this is
load-bearing for the dashboard:

- accept `range=[start,end]` (inclusive), `sort=["field","ASC"|"DESC"]` and
  `filter={...}` as query parameters, translated into Sequelize filters by the
  `parseQuery` middleware;
- filter keys support the suffixes `_gte`, `_lte`, `_gt`, `_lt`, `_ne`;
- return the total row count in the `Content-Range` response header.

Any new list endpoint must honour it.

## Deployment

```bash
cp .env.example .env
npm install
npm run migrate
npx tsx ./src/server
```

or with Compose (which brings its own PostgreSQL, plus an optional self-signed
TLS overlay and an opt-in pgAdmin profile):

```bash
docker compose up -d
docker compose -f docker-compose.yaml -f docker-compose.tls-dev.yaml up -d
```

The service can terminate TLS itself (`SSL_CERT` / `SSL_KEY`) or sit behind a
proxy. Step-by-step in [Deploying component by component](../getting-started/manual-deployment.md#2-quantum-api);
variables in the [configuration reference](../reference/configuration.md#quantum-api).

## Development

```bash
npm install
cp .env.example .env
npm run migrate
npm test          # Jest — the enforced gate
npm run lint      # ESLint + Prettier
```

The application runs under `tsx` (and `ts-jest` for tests) as TypeScript ESM with
`.ts`-extension imports; **there is no `tsc` emit in the normal flow**, so do not
expect a build step. `npm test` is the gate that must stay green.

Adding a resource means: route, thin controller, service for the logic, repository
for persistence, zod schema for input, a migration if the schema changes, and
`Content-Range` on the list endpoint. Walk-through in
[Extending the stack](../development/extending.md#adding-a-resource-to-quantum-api).

Repository documents: `README.md`, `CONTRIBUTING.md`, `CLAUDE.md` (the
architectural rules), `CHANGELOG.md`, and `api.yml` — the OpenAPI 3.1 description,
served as Swagger UI at `/api-docs`.
