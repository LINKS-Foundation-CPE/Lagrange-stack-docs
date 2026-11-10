# Deploying Sqed

This page turns a working Lagrange deployment (gateway, quantum-api, dashboard
and an identity provider, see [Getting started](../getting-started/index.md))
into one that takes its submissions from a SLURM cluster. It describes the
**HPC-only** configuration: SLURM is the only way to submit, and the portal is a
job viewer. That is the configuration Sqed is tested in.

Versions: QC Gateway, quantum-api and the dashboard **2026.11.0** or later, and
the prolog/epilog at the same release. Older gateways look token records up by
the token itself, not by its digest, and do not authenticate these scripts'
tokens.

## 1. Redis between the cluster and the gateway

The prolog writes token records to **the Redis the gateway already uses**
(`REDIS_HOST` in the gateway's configuration). Make it reachable from wherever
SLURM runs the prolog. Only the first node of each allocation talks to Redis,
once at job start and once at job end. Compute nodes and users never do.

- **Redis 7.4 or later.** The records use per-field TTLs (`HPEXPIRE`).
- Give the prolog its own ACL user with the narrowest grant that works: write on
  `qpu:*` and `licenses:hpc:*`, read-write on `licenses:in_use`, and `EVAL`. The
  gateway only ever reads `qpu:*`.

## 2. The cluster: license and prolog/epilog

Define a SLURM license that stands for the quantum computer, named `qpu` by
default. Either a local license in `slurm.conf` (`Licenses=qpu:4`), or a remote
one in the accounting database, so that several clusters share it:

```sh
sacctmgr add resource name=qpu server=<name> type=license count=4 \
    cluster=<cluster> allowed=100
```

Jobs then request it as `-L qpu:1`, or `-L qpu@<name>:1` for a remote license.
How many licenses exist is how many jobs may hold the machine at once.

Then install the scripts on every node, and on wherever the prolog runs, as the
[prolog/epilog README](https://github.com/LINKS-Foundation-CPE/Lagrange-prolog-epilog#install)
describes. In short:

- copy the scripts to `/opt/sqed`, the configuration to `/etc/sqed`;
- add the drop-ins to `Prolog=` and `Epilog=` in `slurm.conf`, and set
  `TaskProlog=` (or call it from an existing one);
- in `/etc/sqed/qpu-token.conf` set at least `REDIS_HOST`, the Redis
  credentials, and `MAX_WALLTIME_SECONDS` to the `MaxTime` of the partition that
  carries the license. That bounds how long a token outlives a job whose epilog
  never ran.

The README covers multi-node jobs, which by default get the token on the first
node only, and the scripts' security properties.

## 3. quantum-api: cluster identities, job-portal-only

Two [deployment policy flags](../reference/configuration.md#deployment-policy-flags):

```ini
USERNAME_FORMAT=any     # identities are cluster user names, not e-mail addresses
JOB_PORTAL_ONLY=true    # the portal lists jobs; SLURM owns access and accounting
```

!!! warning "`USERNAME_FORMAT` is frozen at first boot"
    quantum-api stores it in the database the first time it starts and refuses
    to start later with a different value. Set it **before** the first start. To
    change it on an existing database, see
    [`USERNAME_FORMAT`](../reference/configuration.md#username_format).

With `JOB_PORTAL_ONLY=true`:

- users, projects, organizations, slots, reservations, reports and the rest of
  the management API are not mounted, and neither is `/jobAuthorizer`;
- a job report still records the job and its timing, but **nothing is billed**:
  accounting is SLURM's;
- a report naming a project the backend has never seen **creates it** (a free,
  zero-budget row), because the project is the SLURM account and there is no API
  left to create it with. It is created in the reporting user's organization,
  or in the only organization if the deployment has just one.

The dashboard reads the flag from `GET /config` and shows only the job views.

## 4. Identities

The gateway reports each job under the **SLURM user name** of the job. The
backend must already know a user with exactly that identity, or it refuses the
report (`user … not found`). The job still runs, but it is not recorded.

A user becomes known the first time they log in to the portal. With
`USERNAME_FORMAT=any` the identity is the token's `email` claim if there is one,
and otherwise its `preferred_username`. So each cluster user needs an
identity-provider account:

- whose **username is their cluster user name**, and
- that carries **no e-mail address**, or the e-mail would become the identity
  instead.

Have them log in to the portal once. That registers them, and it is also how
they will look at their jobs. If the realm requires e-mail, first name and last
name, relax its user profile for these accounts; Keycloak refuses to log in an
account missing a required attribute.

## 5. The gateway: `sqed` plugins

```ini
AUTH_PLUGINS=sqed          # per-job tokens only
STRICT_PREFIX_MODE=true    # a token without the sqed_ prefix is refused outright
SITE_PLUGIN=sqed           # SLURM-admitted jobs skip the portal's authorization
```

| Variable | Default | Purpose |
|---|---|---|
| `SQED_TOKEN_PREFIX` | `sqed_` | The prefix the auth plugin claims. Must match `TOKEN_PREFIX` in the prolog's configuration. |
| `SQED_DEFAULT_ROLES` | `cortex_user` | Roles every job's principal gets: SLURM vouches for the account, not for privileges. Add `pulla_user` to let jobs submit pulse-level sweeps. |
| `SQED_ROLES_LOOKUP_URL` | — | Optional per-user roles, **added** to the defaults: quantum-api's `GET /userRoles/<username>`, e.g. `http://quantum-api:8500/userRoles`. Fails open: an unreachable backend costs extra roles, not submissions. |
| `SQED_ROLES_CACHE_TTL` | `60` | Seconds a lookup result, including an empty one, is cached. |
| `SQED_ROLES_VERIFY_TLS` | `true` | TLS verification for an `https://` lookup URL. |

Optionally, make each job's concurrent-shot allowance follow the licenses it
holds, instead of the flat `MAX_CONCURRENT_SHOTS`:

```ini
POLICY_PLUGIN=shot_budget
SHOTS_PER_LICENSE=625000   # allowance = min(licenses × this, MAX_CONCURRENT_SHOTS)
```

Sweeps keep the `MAX_CONCURRENT_SWEEPS` limit either way.

The reporter needs no change. It reports Sqed jobs like any other, with the
SLURM account as the project.

## 6. Submitting from a job

Inside a job holding the license, `IQM_TOKEN` is already set, and the IQM client
picks it up from the environment. The user only points the client at the gateway:

```bash
#!/bin/bash
#SBATCH --account=<account>
#SBATCH --licenses=qpu:1
#SBATCH --time=00:30:00

export IQM_SERVER_URL=https://<gateway>
python my_experiment.py
```

```python
import os
from iqm.qiskit_iqm import IQMProvider

backend = IQMProvider(os.environ["IQM_SERVER_URL"]).get_backend()  # IQM_TOKEN from the environment
```

The token works from the moment the job starts until it ends, and only then.

## 7. Checking it

```sh
sbatch -A <account> -L qpu:1 --wrap 'env | grep IQM_TOKEN'   # the token reaches the job
redis-cli --scan --pattern 'qpu:*'                           # while it runs: qpu:token:<64 hex>, qpu:job:<id>
```

While a job runs there is one `qpu:token:<sha256>` hash and one `qpu:job:<id>`
per job, and never a key containing a token. After the job, both are gone. In
the gateway's log a submission shows `Authenticated user via sqed (prefix)`, and
in the portal the job appears under the user, with the account as its project.

## Upgrading from before 2026.11.0

2026.11.0 changed where records live, from `qpu:token:<token>` to
`qpu:token:<sha256(token)>`. Upgrade the gateway and the prolog/epilog
**together, with no licensed job running**: a record written in the old layout is
not found by the new gateway, and the reverse is also true.

## Trying it without a cluster

The [one-command installer](../getting-started/portable-deployment.md#optional-hpc-scheduler-variant)
has an opt-in variant that adds a containerised SLURM cluster with the
prolog/epilog installed and the settings above applied.
