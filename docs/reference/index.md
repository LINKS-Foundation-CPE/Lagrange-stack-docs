# Reference

Exact interfaces, for integrating with the stack rather than reading about it.

<div class="grid cards" markdown>

-   **[quantum-api HTTP API](quantum-api-http.md)**

    Every endpoint: authentication, roles, request and response shapes, list
    conventions, error model. The machine-readable source is `api.yml` in the
    repository, served as Swagger UI at `/api-docs`.

-   **[Gateway ↔ backend contract](machine-endpoints.md)**

    `/jobAuthorizer`, `/jobReport` and `/userRoles` in detail — what each call
    carries, what it decides, what it trusts, and how to harden it.

-   **[Configuration reference](configuration.md)**

    Every environment variable of every component, with defaults and the
    consequences of getting it wrong.

-   **[Database schema](database-schema.md)**

    Tables, columns, relationships and the invariants that are not expressed in
    the schema.

</div>

## Conventions used across the stack

**Time.** All timestamps on the wire are ISO 8601. The backend parses submission
and execution timestamps as **UTC** explicitly, so a report without a zone offset
is interpreted as UTC, not as server-local time. Send offsets.

**Budgets and durations.** Always **QPU-milliseconds**, except for an
organization's `initial_budget`, which is in hours. `BIGINT` columns
(`projects.remaining_budget`, `budget_events.value`) are serialised as **JSON
strings**, not numbers, because they can exceed the safe integer range in
JavaScript. Parse accordingly.

**Identity.** The user identity is a single string, stored in `users.email` and
reported by the gateway as a job's `username`. Whether it must be an e-mail
address is a per-deployment decision frozen at first boot — see
[`USERNAME_FORMAT`](configuration.md#username_format).

**Errors.** Every error response is `{"message": "<human-readable reason>"}`. The
messages are meant to be shown to users and are the primary diagnostic for refused
submissions — do not swallow them.

**Job identity.** `jobid` is the identifier the *machine* assigned, and it is the
key every component uses to talk about a job. The backend's own `jobs.id` is
internal.
