# Licensing

Every component of the stack — and this documentation — is licensed under the
**European Union Public Licence v. 1.2 (EUPL-1.2)**.

## Why EUPL-1.2

EUPL-1.2 is an OSI- and FSF-approved copyleft licence maintained by the European
Commission. Two properties made it the right choice here:

- **It is a European public-sector licence.** The stack was built with public
  research funding in an EU context, and the EUPL exists precisely for that case: it
  is available in all official EU languages, each version equally valid, and it names
  EU law and jurisdiction rather than requiring a choice of a foreign legal system.
- **It is compatible with the major copyleft licences.** Through the EUPL
  compatibility list, code under **GPL v2 and v3, LGPL, AGPL, MPL, CeCILL** and
  several others can be combined with this stack without a licence conflict — the
  combined work is distributed under the compatible licence. In practice that means
  reusing this code does not force a project to abandon the licence it already uses.

## What it means in practice

- **You may** use, study, modify, run and distribute the software, including
  commercially.
- **If you distribute** a modified version, or make it available over a network as a
  service, you must make the corresponding source available under EUPL-1.2 or a
  compatible licence. The EUPL's copyleft covers network distribution, so running a
  modified fork as a public service is a distribution.
- **You must** keep the copyright and licence notices intact.
- **No warranty.** See the licence text.

This is a summary, not legal advice. The authoritative text is the `LICENSE` file in
each repository.

## Per-repository licences

| Repository | Licence |
|---|---|
| QC Gateway | EUPL-1.2 |
| quantum-api | EUPL-1.2 |
| quantum-dashboard | EUPL-1.2 |
| portable-deployment | EUPL-1.2 — applies to the installer, configuration, documentation and test notebook in that repository; each component it deploys carries its own licence |
| this documentation | EUPL-1.2 |

## Third-party components

The stack runs on, but does not redistribute, a number of independently licensed
components: PostgreSQL, Redis, RustFS, Keycloak, nginx, and the vendor's own control
software. Their licences apply to them. The Python and Node dependency trees are
declared in `requirements.txt` and `package.json` respectively, with their own
licences.

The vendor client SDKs users point at a deployment (`qiskit-iqm` and friends) are
third-party software and are not part of this stack.
