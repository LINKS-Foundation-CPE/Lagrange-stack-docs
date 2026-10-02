# Running your first circuit

This is the acceptance test for the whole stack. It exercises the identity
provider, the gateway, the backend's authorization, the machine, the object store,
the reporter and the billing ledger — in that order. If it works end to end,
the installation is complete.

## The one thing that changes for users

Nothing about their code. The gateway preserves the vendor API on the wire, so an
existing script keeps working; the only change is the server URL and the token.

```python
from iqm.qiskit_iqm import IQMProvider

provider = IQMProvider("https://<gateway>")   # was: the machine's own URL
backend  = provider.get_backend()
```

The token is read from the environment (`IQM_TOKEN` for `iqm-client`-based SDKs),
and it is a **standard OIDC access token** from your identity provider — not a
platform token, not an API key.

## Obtain a token

For interactive use, whatever OIDC flow your SDK or CLI supports. For scripts and
tests, the password grant against a client with direct access grants enabled:

```bash
export IQM_TOKEN=$(curl -s \
  "https://<idp>/realms/<realm>/protocol/openid-connect/token" \
  -d grant_type=password -d client_id=<client> \
  -d username=<user> --data-urlencode password='<password>' \
  | python3 -c 'import json,sys;print(json.load(sys.stdin)["access_token"])')
```

Access tokens are short-lived — five minutes in a default realm. Re-fetch rather
than debugging a `401` that is really an expiry.

!!! note "Self-signed certificates"
    On a development deployment, also
    `export REQUESTS_CA_BUNDLE=/path/to/cert.pem`. In a notebook set it with
    `os.environ[...]` **before** constructing the provider — a kernel started
    earlier will not have inherited it.

## Submit

```python
import os
from qiskit import QuantumCircuit, transpile
from iqm.qiskit_iqm import IQMProvider

os.environ.setdefault("REQUESTS_CA_BUNDLE", "/path/to/cert.pem")  # dev only

backend = IQMProvider("https://<gateway>").get_backend()

qc = QuantumCircuit(3, 3)
qc.h(0)
qc.cx(0, 1)
qc.cx(0, 2)
qc.measure_all()

job = backend.run(transpile(qc, backend=backend), shots=1000)
print(job.job_id())
print(job.result().get_counts())
```

Two things worth knowing when the circuit itself fails:

- **Let the transpiler map the circuit.** Two-qubit gates must respect the
  machine's coupling topology. A hand-placed `cx` between two qubits that are not
  coupled fails at compilation on the machine, which surfaces as
  `compilation_started → failed` — a machine-side error, not a platform one.
- **Choosing a project.** If your identity has a default project, submissions bill
  to it. To bill a different project you belong to, pass the project name in the
  submission metadata your SDK provides (the gateway reads it and forwards it to
  the backend as `project_name`).

## Verify the whole chain

Check these in order. Each failure points at a different component.

### 1. The submission was authorized

A `403` here comes from the backend, and the message says why: not a member of the
project, no default project, project expired, machine reserved to another project,
no budget. See [authorization](../architecture/accounting.md#authorization-of-a-submission)
for the full decision tree.

A `429` means you hit the per-user concurrency limit, not a permissions problem.

### 2. The job exists in the accounting system

Log in to the dashboard as the submitting user and open **My Jobs**. The job
should appear with its vendor job id, status, project and submission time.

If the job ran on the machine but is missing here, the initial `POST /jobReport`
did not land. Look at the backend's audit log (`/api/logs`, platform admin only)
for a `reportingError` entry — "user not found" and "project not found" are the
two usual causes, and both mean the identity or the project name did not match
what the backend knows.

### 3. Artifacts were captured

The job record links to the results portal. Following it should show the
submitted circuit, the results, the machine timeline and — when the vendor
publishes one for the job's calibration set — a **calibration metrics** card:
readout fidelity, single- and two-qubit gate fidelities, and coherence times. If
the link is dead, the reporter could not reach the object store, or
`JOBS_PORTAL_URL` / `MINIO_SERVER_URL` point somewhere the browser cannot
resolve.

!!! note "Which readout fidelity the card shows"
    An IQM calibration set carries more than one single-shot-readout series per
    qubit — the readout calibration of the `measure` gate, and a separate
    fidelity characterisation — and they report different numbers for the same
    qubit. The card shows the `measure` series, which is the one the monitoring
    dashboards plot, and names the task in the section heading if an artifact
    carries only the other one. A card whose numbers seem not to match a
    dashboard is worth checking against that heading first.

### 4. The project was billed

Open the project in the dashboard and look at its transactions, or:

```bash
curl -sk "https://<api>/api/projects/1/transactions" \
     -H "Authorization: Bearer $PLATFORM_TOKEN"
```

There should be a negative entry whose description names your job id, and the
project's remaining budget should have dropped by the same amount. That amount is
`execution_end − execution_start` in milliseconds, as reported by the machine.

**If the job is terminal but there is no ledger entry**, work through the
exemptions in order — they are all legitimate reasons not to bill:

- the project has `free_queue` set;
- the job is flagged `usedReservation` (it ran inside a reservation, which was
  paid for when it was booked);
- the terminal report carried no execution window, so there was nothing to bill;
- the job had already been finalised once — the terminal report is idempotent and
  a repeat is a no-op.

If none of those apply, the terminal `PUT /jobReport/{jobid}` never arrived: check
that the reporter process is running and that it can reach the backend.

## What you have proven

At this point every contract in the stack has been exercised at least once:

| Contract | Proven by |
|---|---|
| OIDC token validation (gateway) | the submission was not rejected with `401` |
| Route table and RBAC | the submission reached the machine |
| `POST /jobAuthorizer` | the submission was authorized rather than `403` |
| Concurrency counters | no spurious `429`, and the counter was released |
| Vendor proxying | the machine returned a job id and results |
| Payload capture | the submitted circuit is in the object store |
| `POST /jobReport` | the job appears in the dashboard |
| Reporter reconciliation | the job reached a terminal status with artifacts |
| `PUT /jobReport/{jobid}` and billing | the ledger entry exists |

If something did not work, [Troubleshooting](troubleshooting.md) maps symptoms to
causes.
