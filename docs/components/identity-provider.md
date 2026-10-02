# Identity provider

The stack does not implement authentication. It expects an **OIDC provider** and
validates its tokens; every deployment so far uses **Keycloak**, and the
`keycloak-deployment` repository holds a working configuration plus a realm
bootstrap script.

This page is the contract: what the provider must offer for the rest of the stack
to work.

## Token requirements

Both the gateway and the backend validate access tokens themselves, against the
provider's JWKS endpoint. A token must therefore:

- be a **JWT signed with RS256**, with a `kid` resolvable from the realm's JWKS
  endpoint;
- carry a **stable `sub`** claim — the backend records it on first login and
  matches on it thereafter;
- carry an **`email`** claim, which is the identity the platform keys on
  (`users.email`) and the string the gateway reports as a job's `username`;
- carry realm roles in **`realm_access.roles`**.

Access-token lifetime is a realm setting. Short lifetimes (the Keycloak default of
five minutes) are fine — the SDK refreshes, and the platform holds no session
state — but they are a common source of confusing `401`s in scripts.

## Realm objects to create

| Object | Requirement |
|---|---|
| **Realm** | Any name. Referenced as `KEYCLOAK_REALM` in the backend and inside `KEYCLOAK_ISSUER` / `KEYCLOAK_JWKS_URL` in the gateway. Conventionally `cortex`. |
| **Realm role `cortex_user`** | Baseline machine access: circuit submission and job inspection. |
| **Realm role `pulla_user`** | Sweep / pulse-level submission ("Pulse access"). Seeds the platform's own record of the grant when a user is first registered; afterwards the platform decides. |
| **Realm role `platform-admin`** | Full platform administration → `admin` + `impersonator`. |
| **Realm role `platform-ro-admin`** | Read-only platform administration → `readOnlyAdmin`. |
| **Public client for the dashboard** | Authorization-code (standard) flow; redirect URIs and web origins covering the dashboard origin. |
| **Public client for the machine path** | Used to obtain a token for the vendor SDK. Direct access grants are useful for scripted tests. One client can serve both roles; separating them keeps redirect URIs tidy. |
| **Users** | Username **is** the e-mail address, with `email` set and `emailVerified` true. |

The role names are conventional, not hardcoded in the gateway core: they appear in
the vendor plugin's route table, which maps `(path, method) → [roles]`. Renaming
them means editing that table and the realm. The one name the backend knows is the
Pulse-access role, and it is configurable via `PULSE_ACCESS_OIDC_ROLE`.

Note that `pulla_user` is not a gate at the gateway: the sweep route asks for
`cortex_user` like every other route, and pulse access is decided by the backend
from `users.pulla_user`. The realm role seeds that flag at first registration.

## Bootstrapping a realm

`keycloak-deployment/bootstrap-dev-realm.sh` creates all of the above idempotently
against a running Keycloak, using `kcadm.sh` inside the container. It creates the
realm, the four roles, two public clients, and a set of test users with generated
passwords written to a gitignored file.

Sketch of what it does, in case you are adapting it to a different provider or
doing it by hand:

```bash
kcadm.sh config credentials --server <url> --realm master --user admin --password <pw>

kcadm.sh create realms -s realm=cortex -s enabled=true

for role in cortex_user pulla_user platform-admin platform-ro-admin; do
  kcadm.sh create roles -r cortex -s name=$role
done

kcadm.sh create clients -r cortex \
  -s clientId=dashboard -s publicClient=true \
  -s standardFlowEnabled=true -s directAccessGrantsEnabled=true \
  -s 'redirectUris=["https://dashboard.example.org/*"]' \
  -s 'webOrigins=["https://dashboard.example.org"]'

kcadm.sh create users -r cortex \
  -s username=user@example.org -s enabled=true \
  -s email=user@example.org -s emailVerified=true
kcadm.sh set-password -r cortex --username user@example.org --new-password <pw>
kcadm.sh add-roles -r cortex --uusername user@example.org --rolename cortex_user
```

!!! warning "The development script is a development script"
    The shipped bootstrap creates **public clients with `redirectUris=["*"]` and
    `webOrigins=["*"]`** and generates test-user passwords. That is deliberate for
    a throwaway environment on a bare IP with no DNS. For anything else, scope
    redirect URIs and web origins to your actual origins.

## Bootstrapping platform administration

There is no seeded superuser and no database bootstrap. The first administrator is
created by granting `platform-admin` in the realm: on their next token exchange
the backend auto-provisions the user row and derives the `admin` role. From there,
everything else is done through the dashboard or the API.

This also means **revoking `platform-admin` in the realm revokes it in the
platform**, at the next login. Organization- and project-scoped roles are the
opposite: they live in the backend database and are managed through the dashboard.

## Federation

Nothing in the stack cares how the provider authenticates people. Production
deployments federate institutional identity — SAML from a university identity
provider, an OIDC provider, or a social/lab account broker — and Keycloak maps the
result into realm roles. The stack only ever sees the resulting token.

Two consequences worth planning for:

- **Auto-provisioning is a feature here.** Federating an entire institution means
  any member can log in and get an empty account; they can do nothing until an
  administrator places them in a project. This is how a deployment onboards a
  large user base without pre-registration.
- **The e-mail claim must be stable and unique.** It is the identity the platform
  keys on, and job attribution follows it. If your federation can change a user's
  e-mail, plan for the fact that the platform will treat the new address as a
  different identity.

## Deployment notes

The `keycloak-deployment` repository runs Keycloak with a PostgreSQL backend and a
volume-backup sidecar. Two settings matter to the rest of the stack:

- **Relative path.** Deployments so far serve Keycloak under `/auth` on the same
  host as the machine API (`KC_HTTP_RELATIVE_PATH=/auth`), which keeps issuer URLs
  and CORS simple. If you serve it elsewhere, the `/auth` segment disappears from
  `KEYCLOAK_BASE_URL`, `KEYCLOAK_ISSUER` and `VITE_KEYCLOAK_URL` — a very common
  cause of `401`s where the token is fine but the issuer does not match.
- **Proxy mode.** Behind a TLS-terminating proxy, Keycloak needs `KC_PROXY=edge`
  (or the current equivalent) and forwarded-header handling, or it will generate
  redirect URLs on the wrong scheme.
