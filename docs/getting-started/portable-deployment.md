# One-command stack

The **portable-deployment** project
([Lagrange-stack-portable](https://github.com/LINKS-Foundation-CPE/Lagrange-stack-portable) on GitHub) stands up the entire stack on a single Linux
host: QC Gateway, quantum-api, quantum-dashboard, a dedicated Keycloak,
PostgreSQL, Redis and RustFS — on a bare IP address with a self-signed
certificate. No FQDN, no DNS, no certificate authority.

!!! warning "Development and integration only"
    This deployment ships development credentials, permissive CORS and redirect
    settings, and an anonymous-read object store. **Never expose it to an
    untrusted network.** It exists so you can evaluate, develop against, and
    integration-test the stack — not to run a service.

## Install

```bash
git clone https://github.com/LINKS-Foundation-CPE/Lagrange-stack-portable portable-deployment
cd portable-deployment
./install.sh --ip <host-ip> \
             --machine-url https://<mock-or-machine> \
             --token <machine-service-token>
```

`--ip` is the address **clients will use** — a VM floating IP, a LAN address, or
`127.0.0.1` for laptop-local use. It is not autodetected, because autodetection
is wrong behind NAT and a wrong value produces a certificate that fails
validation everywhere.

The installer is **idempotent**: re-running it keeps existing secrets, `.env`
files and certificates. That makes it safe to re-run after changing one flag, and
it is the normal way to repair a partially-failed install.

What it does, in order:

1. checks prerequisites and that the ports it needs are free;
2. clones each component at the revision pinned in `components.env` (override
   `GIT_BASE`, or any individual `*_REPO`, to clone from a fork or a mirror);
3. generates the shared self-signed certificate and any missing secrets;
4. writes each component's `.env`;
5. starts the identity provider and bootstraps its realm, clients, roles and test
   users;
6. starts the gateway stack, then the backend, then the dashboard;
7. seeds an organization, a project and test users **through the real API**, so
   vault accounting, budget events and role derivation all behave exactly as they
   would in production;
8. runs smoke tests.

## What you get

| Port | Service | Notes |
|---|---|---|
| 8000 | QC Gateway | point the vendor client's server URL here; API-identical to the machine |
| 8080 | quantum-dashboard | Vite dev server, hot-reloading the checked-out source |
| 8500 | quantum-api | REST backend, native TLS |
| 8843 | Keycloak | realm `cortex`, admin console under `/auth/admin` |
| 8901 | RustFS console | |
| 8940 | Job results portal | static viewer linked from job records |
| 9000 | RustFS S3 API | artifact links point here |

**Seeded state** — an organization with a 100-hour vault, a project with a
10-hour budget drawn from it, a platform-admin user, and a regular user with the
project as their default. Generated passwords are written to files inside the
`keycloak-deployment` checkout; the installer prints the paths.

**Architecture** — one nginx `tls-proxy` terminates HTTPS for the gateway stack
with the shared certificate; quantum-api and Keycloak terminate their own TLS
with the same certificate; everything speaks plain HTTP inside the Compose
networks. This matters: **no service's configuration diverges from a production
deployment**, so what you exercise here is the real code path with real TLS
termination in front of it, not a special "dev mode".

## First steps after install

**Accept the certificate.** Once per browser, visit each of
`https://<ip>:8843`, `https://<ip>:8500` and `https://<ip>:8080` and accept the
self-signed certificate. Until you do, the dashboard login hangs with no error —
the browser is silently refusing the cross-origin request to the backend.

**Log in** to the dashboard at `https://<ip>:8080` as the seeded platform admin.
You should see organizations, projects, users, slots and logs in the sidebar. Log
in as the regular user and you should see only *My Projects* and *My Jobs*.

**Submit a circuit** — see [Running your first circuit](first-circuit.md). The
repository also ships a ready-made notebook (`test/01_hello_world_dev.ipynb`)
that fetches the certificate, obtains a token by password grant and runs a GHZ
circuit with plain `qiskit-iqm`.

## Day-2 operations

```bash
./teardown.sh              # stop everything, keep data volumes
./teardown.sh --volumes    # also drop databases, realm and Redis
./teardown.sh --purge      # volumes + secrets + certificates + env files
./install.sh --ip <ip>     # re-create; existing secrets and envs are reused
```

Logs are plain container logs — `docker logs <container>`. The containers that
matter are the gateway proxy, the job reporter, the backend, Keycloak, the TLS
proxy and RustFS; the installer prints their names.

To change a component's revision, edit `components.env` and re-run the installer.
To develop against a local checkout, point the relevant `*_REPO` at it (or edit
the cloned tree in place — the dashboard runs a hot-reloading dev server, and the
backend and gateway are bind-mounted, so most changes need only a container
restart).

## Adapting it for your own deployment

The installer is a readable shell script and is the most useful reference for
"what does this stack actually need to be configured with". Three things are worth
lifting from it when building your own deployment:

- **The `.env` files it generates.** They are a working, minimal configuration for
  each component, with the cross-references (backend URL in the gateway, CORS
  origin in the backend, issuer and realm everywhere) already consistent. Compare
  against the [configuration reference](../reference/configuration.md).
- **The realm bootstrap.** `keycloak-deployment/bootstrap-dev-realm.sh` creates
  exactly the realm objects the stack expects — realm, two public clients, four
  realm roles, users whose username is their e-mail. See
  [Identity provider](../components/identity-provider.md).
- **The seed script.** `quantum-api/scripts/dev-seed.sh` drives the real API to
  create an organization, its vault, a project funded from the vault, and
  memberships. It is a good template for provisioning a new deployment, and a
  good example of the API's semantics.

## Optional: HPC scheduler variant

The installer has an opt-in `--with-slurm` flag that additionally stands up a
containerised SLURM cluster and switches the gateway to an HPC-oriented site
plugin, for work on integrating the stack with a batch scheduler. This path is
**under active development** and depends on components not yet part of a release:
treat it as a preview, not as a supported deployment. The default install has
nothing to do with SLURM.
