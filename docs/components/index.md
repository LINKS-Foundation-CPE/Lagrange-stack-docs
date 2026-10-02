# Components

One page per component: what it is, what it exposes, what it needs, and where its
own documentation lives.

| Component | Language / stack | Exposes | Needs |
|---|---|---|---|
| [QC Gateway](qc-gateway.md) | Python 3.11, FastAPI | The vendor API, verbatim, plus health and metrics | OIDC issuer, quantum-api, Redis, PostgreSQL, object store, the machine |
| [quantum-api](quantum-api.md) | Node.js, TypeScript, Express, Sequelize | `/api/*` REST, `/auth/token`, machine endpoints, OpenAPI at `/api-docs` | OIDC issuer, PostgreSQL |
| [quantum-dashboard](quantum-dashboard.md) | React, Vite, react-admin | Static SPA | OIDC issuer, quantum-api |
| [Identity provider](identity-provider.md) | Keycloak (not developed here) | OIDC | PostgreSQL |

For how they fit together see [Architecture](../architecture/index.md); for the
contracts between them see [Reference](../reference/index.md).

## Which component owns what

When you are trying to work out where a behaviour lives, this table is usually
enough:

| Concern | Owner |
|---|---|
| Vendor API shapes, job status polling, artifact classification, calibration | QC Gateway → vendor plugin |
| Who may call which machine route | QC Gateway → vendor plugin's route table |
| Per-user fairness limits | QC Gateway (Redis counters) |
| Archived payloads and artifacts | QC Gateway (object store) |
| Whether a submission is allowed at all | quantum-api (`/jobAuthorizer`) |
| Users, organizations, projects, membership | quantum-api |
| Budgets, reservations, slots, the ledger | quantum-api |
| Job records used for accounting | quantum-api |
| Platform roles and resource scoping | quantum-api |
| Which UI a role sees | quantum-dashboard (enforced by quantum-api) |
| Authentication of humans | Identity provider |
| Machine credentials | QC Gateway only |
