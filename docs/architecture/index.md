# Architecture

## The problem the stack solves

A quantum computer is a single, indivisible, expensive instrument. Vendor
control software typically assumes one operator and offers a service token, or
at best a flat list of users. Operating the machine as a shared research
facility needs a layer the vendor does not provide:

- **Identity** — real people, authenticated against an institutional identity
  provider, not a shared token.
- **Tenancy** — organizations, projects, and membership, so that access can be
  granted and revoked at the granularity funding is granted.
- **Accounting** — a defensible answer to "who used how much QPU time, against
  which project, and when", down to a per-job ledger entry.
- **Fair use** — nobody can monopolise a machine that executes one circuit at a
  time.
- **Exclusive access** — some work (calibration campaigns, demonstrations,
  paid-for blocks) needs the machine to itself for a window of time.
- **Auditability** — every submission archived, every artifact retained, every
  administrative action logged.

…and it needs all of it *without* asking users to adopt a new SDK. Researchers
arrive with Qiskit, Cirq or Qrisp code written against the vendor API, and the
platform must be invisible to it.

## Control plane and data plane

The stack splits along that seam. **Circuits** flow through the QC Gateway;
**decisions and records** live in quantum-api.

```mermaid
flowchart TB
    subgraph clients [" "]
        direction LR
        SDK["User code<br/>(Qiskit / Cirq / Qrisp,<br/>vendor client SDK)"]
        BROWSER["Browser"]
    end

    IDP["Identity provider<br/>(OIDC — Keycloak)"]

    subgraph dataplane ["Data plane"]
        GW["<b>QC Gateway</b><br/>authenticate → RBAC → authorize<br/>→ limit → proxy → capture → report"]
        REPORTER["Job reporter<br/>(background worker)"]
        REDIS[("Redis<br/>counters")]
        JOBDB[("PostgreSQL<br/>job log")]
        S3[("Object store<br/>payloads + artifacts")]
    end

    subgraph controlplane ["Control plane"]
        API["<b>quantum-api</b><br/>users · orgs · projects · budgets<br/>slots · reservations · jobs · ledger"]
        APIDB[("PostgreSQL<br/>system of record")]
        DASH["<b>quantum-dashboard</b><br/>react-admin SPA"]
    end

    MACHINE["Quantum computer<br/>(vendor control software)"]

    SDK -- "vendor API + OIDC bearer token" --> GW
    BROWSER --> DASH
    BROWSER -. "login" .-> IDP
    SDK -. "login" .-> IDP
    GW -. "JWKS" .-> IDP
    API -. "JWKS" .-> IDP

    GW --> REDIS
    GW --> JOBDB
    GW --> S3
    GW -- "proxied verbatim<br/>+ service token" --> MACHINE
    REPORTER -- "poll status,<br/>fetch artifacts" --> MACHINE
    REPORTER --> S3
    REPORTER --> REDIS

    GW -- "/jobAuthorizer<br/>/jobReport" --> API
    REPORTER -- "/jobReport" --> API
    DASH -- "REST /api/*" --> API
    API --> APIDB
```

Consequences of that split worth understanding before reading further:

- **The gateway never owns policy.** It asks. Whether a submission is allowed is
  a question answered by `POST /jobAuthorizer` in the backend, which consults
  project membership, project validity dates, active reservations, slot rules
  and remaining budget. The gateway only enforces what it is told, plus its own
  concurrency limits.
- **The backend never sees a circuit execute.** It learns about jobs from the
  gateway's reports, and bills on the final one. Its notion of "QPU time" is
  whatever the gateway reported as `execution_start`/`execution_end`.
- **Neither service stores passwords.** Both validate JWTs issued by the identity
  provider against its JWKS endpoint.
- **The two databases are separate on purpose.** The gateway's PostgreSQL is an
  operational job log local to the data plane; the backend's PostgreSQL is the
  system of record. A gateway can be rebuilt from scratch without touching
  accounting data.

## Trust boundaries

```mermaid
flowchart LR
    subgraph internet ["Untrusted network"]
        U["Users, browsers"]
    end
    subgraph edge ["Public edge — TLS terminated here"]
        NGINX["Reverse proxy / TLS<br/>vhosts"]
    end
    subgraph internal ["Internal network"]
        GW2["QC Gateway"]
        API2["quantum-api"]
        IDP2["Identity provider"]
        STORES[("Redis · PostgreSQL · Object store")]
    end
    subgraph machinenet ["Machine network"]
        M2["Quantum computer"]
    end

    U --> NGINX --> GW2
    NGINX --> API2
    NGINX --> IDP2
    GW2 --> API2
    GW2 --> STORES
    GW2 --> M2
```

Three properties of this layout are load-bearing:

1. **The machine is never directly reachable.** The only route to it is through
   the gateway, which is the only holder of the machine's service token. Users
   authenticate as themselves; the machine sees one service identity.
2. **The backend's machine endpoints are internal.** `/jobAuthorizer`,
   `/jobReport` and `/userRoles` are the gateway-to-backend contract and are
   expected to be reachable only from the gateway. See
   [Gateway ↔ backend contract](../reference/machine-endpoints.md) for exactly
   what each one trusts today — this is the part of the stack you should review
   most carefully before exposing a deployment.
3. **The browser never talks to the gateway.** The dashboard talks only to
   quantum-api. The gateway serves machine traffic, plus a static jobs-results
   viewer.

## End-to-end: a circuit submission

The full path, with the two reference plugins (IQM as vendor, a portal-backed
site plugin talking to quantum-api):

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant IDP as Identity provider
    participant GW as QC Gateway
    participant API as quantum-api
    participant R as Redis
    participant M as Quantum computer
    participant S3 as Object store
    participant JR as Job reporter

    Note over User,IDP: Once per session
    User->>IDP: OIDC login
    IDP-->>User: access token

    User->>GW: POST /api/v1/jobs/default/circuit (Bearer token)
    GW->>IDP: fetch JWKS / validate signature
    GW->>GW: RBAC — does the role allow this route?
    GW->>GW: parse submission (shots, circuits, project, type)
    GW->>API: POST /jobAuthorizer
    API->>API: user? project member? active? reserved? budget?
    API-->>GW: 200 authorized  /  403 with reason
    GW->>R: reserve concurrency (INCRBY shots/jobs)
    R-->>GW: within limit  /  over limit → 429 + rollback
    GW->>M: proxy request verbatim (+ machine service token)
    M-->>GW: 201 { id: job_id }
    GW->>S3: archive submitted payload
    GW->>GW: write local job row
    GW->>API: POST /jobReport (submitted)
    GW-->>User: 201 { id: job_id }

    Note over JR,API: Background reconciliation loop
    JR->>M: GET job status
    M-->>JR: terminal status + timeline + results
    JR->>S3: upload artifacts, timeline, results index
    JR->>R: release concurrency (idempotent)
    JR->>API: PUT /jobReport/{job_id} (terminal + QPU window)
    API->>API: debit project budget, append ledger entry
```

Read [Life of a job](job-lifecycle.md) for the failure modes, the rollback
rules, and what is idempotent.

## Design rules

These are the invariants the codebase is held to. They are worth knowing because
they explain most of the structure you will find.

### The vendor API is preserved on the wire

The gateway passes paths, bodies, headers and status codes through unchanged.
The vendor plugin parses requests *for bookkeeping* — how many shots, which
project, is this a circuit or a sweep — and never rewrites them. This is what
makes existing SDKs work with nothing but a URL change, and it means the gateway
never has to track vendor API drift in a translation layer.

### Vendor and site specifics live in plugins

The gateway core may not import from `middleware.vendors.*` or
`middleware.sites.*`, and may not branch on a vendor or site name. Two `Protocol`
contracts define the seams:

- `VendorPlugin` — the route table and its role requirements, request/response
  parsing, upstream auth headers, job-status polling, artifact classification,
  calibration handling.
- `SitePlugin` — job authorization and job reporting against whatever the site
  uses as its portal, plus results-URL construction.

Both are selected by environment variable and resolved through a registry. See
[Extending the stack](../development/extending.md).

### Fail open on non-critical paths, fail closed on authorization

Redis, object storage and portal hiccups are logged and the request continues:
losing a counter or an archived payload is worse than losing the job, but not
much worse. Authentication and authorization failures always short-circuit. This
asymmetry is deliberate and is stated as a rule in the gateway's `CLAUDE.md`.

### Money is QPU-time, and it only moves through the ledger

Budgets are denominated in **QPU-milliseconds**. A project's
`remaining_budget` is never mutated without appending the `budget_events` row
that explains the change. Reports, invoices and audits are built from the ledger,
not from the balance. See [Tenancy, budgets and billing](accounting.md).

### New behaviour ships behind a flag whose default changes nothing

Every configurable behaviour added to the backend has a default that reproduces
the previous behaviour, so a new version can be rolled onto a running deployment
without changing what it does. `SLOT_CONSTRAINED_RESERVATIONS`,
`ALLOW_FOREIGN_ORG_QUEUE_IN_SLOTS`, `USERNAME_FORMAT` and
`PULSE_ACCESS_OIDC_ROLE` all follow this rule; see the
[configuration reference](../reference/configuration.md).

### Layers in the backend point one way

`routes → middleware → controllers → services → repositories → models`.
Controllers are thin adapters; business logic is in services; only repositories
touch Sequelize. Any schema change arrives with its migration in the same commit.
