# CLAUDE.md

Orientation for AI assistants (and new human contributors) working on the
**Lagrange technical documentation**. Read this before making changes.

## What this repository is

The public technical documentation site for the Lagrange quantum-computer
management stack: QC Gateway, quantum-api, quantum-dashboard. MkDocs +
Material for MkDocs, published to GitLab Pages (internal) and GitHub Pages
(public).

It documents **the software**, for someone deploying or extending it elsewhere.

## Non-negotiable rules

1. **Nothing deployment-specific.** No host names, IP addresses, internal service
   names, network topology, credentials, certificate details or backup targets. No
   procedures tied to one machine or one institution. Examples use placeholders
   (`<gateway>`, `example.org`). This site is published publicly; treat every commit
   as public even before it is.

2. **Do not duplicate a repository's own documentation.** Each component's
   `README.md`, `CONTRIBUTING.md`, `CLAUDE.md` and (for the gateway) `docs/` are
   authoritative for their internals. Link to them. A duplicated paragraph will
   diverge, and the divergent copy is the one the reader finds.

3. **Document what the code does.** Verify against the source before writing a
   behavioural claim — units, defaults, field names, status codes. Where the code
   does something surprising or a spec file is wrong, document the actual behaviour
   and say the spec disagrees. Known gaps go in `docs/about/index.md`, not into
   silence.

4. **Diagrams are Mermaid**, in ` ```mermaid ` fences. Not images: Mermaid renders in
   both colour schemes, diffs usefully, and needs no external tooling. The
   `pymdownx.superfences` custom fence is already configured.

5. **`mkdocs build --strict` must pass.** It fails on broken internal links and on
   pages absent from `nav`. Add a page to `nav` in `mkdocs.yml` in the same change
   that creates it.

## Structure

```
mkdocs.yml               # theme, extensions, nav (the nav is the source of truth)
requirements.txt         # pinned build dependencies, shared by both CI targets
docs/
├── index.md             # what the stack is, component table, where to start
├── architecture/        # the big picture, components, identity, accounting, job lifecycle
├── getting-started/     # the two deployment paths, first circuit, troubleshooting
├── components/          # one page per component: what it is, needs, exposes
├── reference/           # HTTP API, gateway↔backend contract, configuration, schema
├── development/         # toolchains and gates, extending, releases and publishing
├── about/               # project, status, known gaps, licensing
└── assets/              # partner logos
.gitlab-ci.yml           # build check on branches/MRs + GitLab Pages from default branch
.github/workflows/       # GitHub Pages from GitHub main
```

Where content goes, when it is not obvious:

- **A behavioural rule** (who may do what, what gets billed) → `architecture/`.
- **An exact interface** (endpoint, variable, column) → `reference/`.
- **A task the reader performs** → `getting-started/` or `development/`.
- **"What is this component"** → `components/`, kept short and linking outward.

If a fact belongs in two places, put it in `reference/` and link to it.

## Branches and publishing

| Branch | Purpose | Published to |
|---|---|---|
| `staging` | Where documentation is written; MRs target it | nothing |
| `main` | Reviewed content | GitLab Pages |
| `public` | Orphan branch, cherry-picked from `main` | GitHub Pages via `github main` |

This mirrors the model used by every other repository in the stack. `public` is an
**orphan branch**: its history starts from a clean "Initial public release" commit
and never contains the private history.

### Creating the public branch (first time)

Once `main` holds reviewed content and the GitHub repository exists:

```bash
git remote add github git@github.com:<org>/<repo>.git

git checkout --orphan public
git add -A
git commit -m "Initial public release"
git push origin public
git push github public:main
git checkout main
```

Then, in the GitHub repository: **Settings → Pages → Source: GitHub Actions**, and
fill the published URL into `mkdocs.yml` (`site_url`) and `README.md`.

### Publishing subsequent changes

```bash
git checkout public
git cherry-pick <commit-sha>     # repeat for each commit to publish
git push origin public
git push github public:main
git checkout main
```

**Review every cherry-pick before pushing.** `public` is the gate that keeps
internal detail out of the mirror. Never `git push github main:main`, and never
rebase `public` onto `main` — either carries the full private history to GitHub
irreversibly.

## Local commands

```bash
pip install -r requirements.txt
mkdocs serve             # live preview on :8000
mkdocs build --strict    # exactly what CI runs
```

## What NOT to do

- Don't add a page without adding it to `nav`; `--strict` will fail the build, and
  an unlisted page is unreachable anyway.
- Don't paste a private-documentation page in wholesale. Software content moves here
  *rewritten to be deployment-neutral*; deployment content stays private.
- Don't add build dependencies. The site deliberately uses only
  `mkdocs-material` so that both CI targets build identically and offline.
- Don't reference internal GitLab URLs in page content. Public mirrors, or a
  description of the repository, or nothing.
- Don't soften a known problem into a euphemism. "`/jobReport` is a billing write
  with no authentication" is the sentence a reader needs.

## Working on this repository on its own

Clone it anywhere and it is workable: everything needed to build, test and change
it is in here. Nothing in this file depends on another repository being at hand,
or on any private document.

What it talks to, and where each contract is written down:

This repository has no runtime counterpart — it is documentation. What it
describes:

| Subject | Whose repository owns the detail |
|---|---|
| QC Gateway | `nginx-reverse-proxy` — `README.md`, `docs/DOCUMENTATION.md`, `docs/MODULARIZATION.md` |
| Portal backend | `quantum-api` — `README.md`, `CONTRIBUTING.md`, `api.yml` |
| Portal UI | `quantum-dashboard` — `README.md`, `CONTRIBUTING.md` |

The rule in force here is the second one above: link to those, do not restate
them. This site's job is the view *across* the components — architecture,
identity, accounting, the job lifecycle, the gateway↔backend contract — plus the
exact interfaces in `reference/`, which is the one place where restating the
detail is correct because it is written to be authoritative.

Every repository in the stack carries a `CLAUDE.md` in this same shape, so the
same is true read from the other side. If you find yourself needing a fact that
is not in one of them, that is a gap worth fixing in the repository that owns the
fact — not a reason to go looking for a central document.
