# Getting started

You can have the whole stack running and a circuit billed to a project in about
fifteen minutes, most of which is Docker pulling images.

## What you need before anything else

**A quantum backend to point at.** The gateway proxies to a vendor API; it does
not simulate one. You need either:

- a **mock / simulated device** exposing the vendor API (IQM ships a mock station
  configuration; this is what development against the stack normally uses), or
- access to a **real machine**, in which case you also need its service token.

Everything else in the stack — identity provider, databases, object store — is
brought up for you by the deployment paths below.

**A Linux host with Docker.** Docker Engine with the Compose v2 plugin
(`docker compose version` must work), plus `git`, `curl`, `openssl` and
`python3`. Around 5 GB of disk for images and volumes, and outbound network
access for image pulls. Tested on Ubuntu 24.04; nothing is distribution-specific.

**A client environment.** Python with the vendor client SDK — for IQM,
`qiskit-iqm` (or `cirq-iqm`, or Qrisp). This can be the same host or your laptop.

## Two paths

<div class="grid cards" markdown>

-   :material-flash: **One-command stack**

    A single installer brings up gateway, backend, dashboard, identity provider,
    Postgres, Redis and object store on one host, on a bare IP with a
    self-signed certificate — no DNS, no CA, no FQDN. It seeds an organization,
    a project and test users, and runs smoke tests.

    Use this to evaluate the stack, to develop against it, or as a reference for
    how the pieces are wired.

    [→ One-command stack](portable-deployment.md)

-   :material-tools: **Component by component**

    Deploy each service yourself: create the identity-provider realm, run the
    backend and its migrations, build the dashboard, configure the gateway,
    wire them together.

    Use this when you are integrating into existing infrastructure — your own
    Keycloak, your own Postgres, your own reverse proxy and certificates — or
    when you need to understand exactly what each component requires.

    [→ Component by component](manual-deployment.md)

</div>

Both paths converge on [Running your first circuit](first-circuit.md), which is
also the acceptance test: if a circuit runs and its QPU milliseconds appear as a
ledger entry against a project, the whole stack is working end to end.

## What "working" means

A complete installation demonstrates all five of these. It is worth checking them
in order, because each one depends on the previous:

1. **A user can log in to the dashboard** — identity provider reachable, token
   exchange working, user row auto-provisioned.
2. **An administrator can create an organization and a project with budget** —
   backend and database healthy, vault accounting correct.
3. **An unmodified vendor SDK can submit a circuit through the gateway** — the
   gateway authenticates the OIDC token, the backend authorizes the submission,
   the machine accepts it.
4. **The job appears in the dashboard with artifact links** — the reporter is
   reconciling, the object store is reachable, the results pointer resolves.
5. **The project's budget went down by the job's execution time, with a ledger
   entry** — the terminal report reached the backend and billed.

If you get to step 3 and stop, you have an authenticating proxy. Steps 4 and 5
are what make it a *managed* service, and they are the ones that fail quietly —
check them explicitly.

## Ports and hostnames

Neither path requires DNS. The one-command stack deliberately confines every
service to the 8000–9000 range, which is convenient when only a narrow port range
is reachable on a VM:

| Port | Service |
|---|---|
| 8000 | QC Gateway — point the vendor client's server URL here |
| 8080 | quantum-dashboard |
| 8500 | quantum-api |
| 8843 | Identity provider (Keycloak) |
| 8901 | Object store console |
| 8940 | Job results portal |
| 9000 | Object store S3 endpoint |

A production deployment normally puts all of these behind one TLS-terminating
reverse proxy on port 443 with per-service virtual hosts instead. Nothing in the
stack depends on which arrangement you choose — every component learns its own
public URL from configuration.

## A word on self-signed certificates

The one-command path issues a single self-signed certificate covering the host's
IP, `127.0.0.1`, `localhost` and `host.docker.internal`, and uses it for every
TLS listener. That means:

- **In a browser**, you must visit each HTTPS origin once and accept the
  certificate before the dashboard can talk to the backend. Skipping this
  produces a silent hang at login, which is by far the most common
  "it doesn't work" report.
- **In Python**, point `REQUESTS_CA_BUNDLE` at the certificate file. It must be
  set *in the process that makes the request*: a notebook kernel started before
  you exported the variable will not see it, so set it via `os.environ` before
  constructing the provider.

None of this applies to a deployment with a real certificate.
