# Lagrange technical documentation

Technical documentation for the **Lagrange** quantum-computer management stack —
[QC Gateway](https://github.com/LINKS-Foundation-CPE/QC-Gateway),
[quantum-api](https://github.com/LINKS-Foundation-CPE/Lagrange-API) and
[quantum-dashboard](https://github.com/LINKS-Foundation-CPE/Lagrange-Dashboard): architecture, deployment, API reference and extension guides. Developed in the context of the
**QTech Piemonte** strategic initiative.

<a href="https://linksfoundation.com"><img src="docs/assets/logo-links.png" alt="LINKS Foundation" height="60"></a>&nbsp;&nbsp;&nbsp;<a href="https://www.polito.it"><img src="docs/assets/logo-polito.png" alt="Politecnico di Torino" height="60"></a>&nbsp;&nbsp;&nbsp;<a href="https://www.inrim.it"><img src="docs/assets/logo-inrim.jpg" alt="INRIM" height="60"></a>

---

Built with [MkDocs](https://www.mkdocs.org/) and
[Material for MkDocs](https://squidfunk.github.io/mkdocs-material/).

Published at **<https://links-foundation-cpe.github.io/Lagrange-stack-docs/>**.

## What belongs here

**Software documentation**: how the stack is built, how the components talk to each
other, how to deploy and test it, what the APIs are, how to extend it. Written to
be self-contained — a reader with no access to private material should be able to
stand the stack up and run a circuit through it.

**Not here:**

- *Deployment-specific material* — host names, network topology, credentials,
  backup targets, machine operating procedures. That stays in the private
  documentation.
- *End-user documentation for the LINKS deployment* — how to use the Lagrange
  machine as a researcher. That lives in the public
  [Lagrange user documentation](https://pages.linksfoundation.com/links-iqm-spark/docs/).
- *Per-repository detail* — each component's `README.md`, `CONTRIBUTING.md`,
  `CLAUDE.md` and (for the gateway) `docs/` remain the authoritative source for
  their own internals. This site links to them rather than duplicating them.

## Local preview

```bash
python3 -m venv .venv && . .venv/bin/activate
pip install -r requirements.txt
mkdocs serve            # http://127.0.0.1:8000
mkdocs build --strict    # what CI runs; fails on broken links
```

`--strict` is what the pipeline enforces: a broken internal link, or a page not
listed in `nav`, fails the build.

## Branches

| Branch | Purpose |
|---|---|
| `staging` | Where documentation is written; merge requests target it |
| `main` | Reviewed content; deploys GitLab Pages |
| `public` | Orphan branch, cherry-picked from `main`; pushed to GitHub |

See [CONTRIBUTING.md](CONTRIBUTING.md) for the workflow and
[CLAUDE.md](CLAUDE.md) for the publishing procedure.

## Licence

EUPL-1.2 — see [LICENSE](LICENSE), consistently with the rest of the stack.
