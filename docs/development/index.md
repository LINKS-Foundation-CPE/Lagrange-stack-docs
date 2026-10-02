# Development

Each repository carries its own `CONTRIBUTING.md` with the authoritative
instructions and a `CLAUDE.md` stating the architectural rules. This page is the
cross-repository view: what the conventions have in common, and what to know before
touching more than one component.

## Toolchains at a glance

| | QC Gateway | quantum-api | quantum-dashboard |
|---|---|---|---|
| Language | Python 3.11 | TypeScript (ESM) | TypeScript + React |
| Install | `pip install -r requirements-dev.txt` | `npm install` | `npm install` |
| Run | `MIDDLEWARE_MODE=… docker compose up -d` | `npx tsx ./src/server` | `npm run dev` |
| Test | `pytest` (with `fakeredis`) | `npm test` (`jest --runInBand`) | — |
| Lint / format | `ruff check .`, `ruff format --check .` | `npm run lint` (ESLint + Prettier) | `npm run lint` |
| Types | `mypy` | `tsc --noEmit` | `tsc --noEmit` |
| Hooks | `pre-commit install` | — | — |
| Build | Docker image | none — runs under `tsx` | `npm run build` → `dist/` |

Two things that surprise people:

- **quantum-api has no build step.** It runs as TypeScript ESM under `tsx`, with
  `.ts`-extension imports. There is no `tsc` emit in the normal flow, so do not
  look for a `dist/`.
- **quantum-api tests must run serially.** `npm test` is `jest --runInBand`
  because the suite shares one test database. Running Jest in parallel fails
  wholesale, in a way that looks like a broken tree rather than a concurrency
  problem.

## Gates

| Repository | The gate |
|---|---|
| QC Gateway | `ruff check`, `ruff format --check` and `mypy` must all pass. `pre-commit` runs them on staged files; do not bypass it with `--no-verify`. |
| quantum-api | **`npm test` must stay green.** Lint and strict type-checking carry pre-existing debt across the tree — do not add to it in files you touch, and do not treat red elsewhere as licence to skip local checks. |
| quantum-dashboard | Lint and type-check clean for the files you touch. |

## Getting a development environment

The fastest path for any of the three is the
[one-command stack](../getting-started/portable-deployment.md): it gives you a
working identity provider, database, object store and a mock machine to submit
against, with a component checkout you can edit in place. The dashboard runs as a
hot-reloading dev server; the gateway and backend are bind-mounted and need only a
container restart.

For a component in isolation you need less:

- **quantum-api** — PostgreSQL plus any OIDC issuer. Tests need no issuer.
- **quantum-dashboard** — a running quantum-api, or a stub that honours
  `Content-Range`.
- **QC Gateway** — its Compose stack plus something answering the vendor API.
  Tests need neither: they use `fakeredis` and mock the upstream.

## Conventions shared by all three repositories

**Documents.** `README.md` for users and evaluators, `CONTRIBUTING.md` for the
workflow, `CLAUDE.md` for the architectural rules — written for AI assistants but
the fastest orientation for a new human contributor too — and `CHANGELOG.md` in
Keep-a-Changelog format.

**Comments explain *why*, not *what*.** A comment restating the code is noise.
Docstrings are short and factual.

**No casual dependencies.** These run in production. Each addition to
`requirements.txt` or `package.json` is justified in the merge request.

**Nothing secret in git.** `.env` files, credentials, certificates, database data
and object-store contents are all gitignored, in every repository. Keep it that way.
Configuration examples use placeholders.

**Deployment-neutral code.** No repository hardcodes a host name, an IP address or
an institution. If a behaviour differs per deployment it is an environment variable
with a documented default — and in quantum-api, that default must preserve the
previous behaviour so the change can be rolled onto a running deployment.

## Working across components

A change that spans components needs a compatibility story, because the components
are released and deployed independently.

**Adding a field to a job report.** The gateway must keep working against a backend
that ignores the new field, and the backend against a gateway that does not send it.
Make it optional on both sides; deploy the backend first.

**Changing an authorization rule.** It lives in the backend. The gateway forwards
the refusal message verbatim, so the message *is* the user-facing text — write it
for a user, not for a log.

**Adding a resource.** Backend first (endpoint plus `Content-Range`), dashboard
second. The reverse order produces a UI that 404s.

**Adding a deployment flag the UI must react to.** Expose it in the backend's public
`GET /config` and read it in the dashboard through `useConfig()`, rather than adding
a build-time `VITE_*` variable. That keeps one dashboard build valid for every
deployment.

**Touching authentication.** JWT validation and the RBAC middleware are load-bearing
for the gateway integration: a change there affects who can authorize and report
jobs. Call it out explicitly in the merge request rather than letting it ride along
with something else.

## Where the risk is

Three areas where a change costs more than it looks:

1. **`proxy_and_capture` in the gateway.** The order of authenticate → RBAC →
   authorize → reserve → proxy → account is the request contract. Reordering it is a
   significant change: read the whole function first and explain the new flow.
2. **Billing in `jobReport.service.ts`.** The early return on an already-finalised
   job is what makes terminal reports idempotent. Removing it double-bills.
3. **Migrations.** Never edit a released one. Prefer additive, nullable changes so a
   migration can be applied before the code that needs it.

## Next

- [Extending the stack](extending.md) — vendor and site plugins, new resources,
  new deployment flags.
- [Release and publishing](releases.md) — CalVer, changelogs, and the
  private-to-public publishing workflow.
