# Contributing to the Lagrange technical documentation

## Workflow

1. Branch from `staging`.
2. Write, and preview locally with `mkdocs serve`.
3. Run `mkdocs build --strict` — this is the CI gate and it fails on broken
   internal links or pages missing from `nav`.
4. Open a merge request against `staging`.
5. `staging` is merged into `main` when a set of changes is ready; `main` deploys
   GitLab Pages.
6. Publishing to GitHub is a separate, deliberate step — see [CLAUDE.md](CLAUDE.md).

## What belongs on this site

Software documentation: architecture, contracts, configuration, API reference,
deployment and extension guides. The test for any sentence is: **would it still be
true and useful for someone deploying this stack somewhere else?**

If the answer is no, it belongs in the private documentation instead. Specifically,
do not add:

- host names, IP addresses, internal service names, or network topology;
- credentials, secrets, certificate details, or backup targets;
- procedures tied to one machine or one institution;
- anything you would not want on a public website.

Use placeholders — `<gateway>`, `example.org`, `user@example.org` — in every example.

## Style

**Write for a competent engineer who has not seen this system.** Assume they know
HTTP, Docker, OIDC and SQL; do not assume they know our conventions or our history.

**Say the useful thing.** Prefer "an organization's `initial_budget` is in hours
while a project's `budget` is in milliseconds — getting this wrong funds a project
3.6 million times over" to "note the units differ". Concrete consequences are what
readers remember.

**Document what the code does, not what it should do.** Where behaviour is
surprising or a known wart, say so and say what the actual behaviour is. Where
something is a known gap, list it in `docs/about/index.md` rather than quietly
omitting it.

**Do not duplicate a repository's own documentation.** Link to it. Duplicated
content diverges, and the copy that diverges is always the one the reader found.

**Structural conventions:**

- One `#` heading per page, matching its `nav` title closely enough to be findable.
- Tables for anything enumerable — endpoints, variables, roles, failure modes.
- Mermaid for diagrams (` ```mermaid ` fences), not images: they stay readable in
  both colour schemes, diff usefully, and need no external tooling.
- Admonitions (`!!! warning`, `!!! note`, `!!! danger`) for things that will bite
  someone. Use them sparingly enough that they keep their force.
- Relative links between pages (`../reference/configuration.md`), so `--strict`
  can verify them.

## Keeping it true

The most useful thing a contributor can do is fix a page that has drifted. When you
change behaviour in a component, the pages most likely to need updating are:

| You changed | Check |
|---|---|
| an endpoint, a payload, a role requirement | `reference/quantum-api-http.md` |
| the gateway ↔ backend contract | `reference/machine-endpoints.md` |
| any environment variable | `reference/configuration.md` |
| a model or migration | `reference/database-schema.md` |
| authorization or billing rules | `architecture/accounting.md`, `architecture/identity.md` |
| the request pipeline or the reporter | `architecture/job-lifecycle.md` |
| a plugin protocol | `development/extending.md` |
| a known gap you closed | `about/index.md` |
