# Day-to-day use

What every signed-in account has, in sidebar order. Nothing here needs an
administrative role.

## Signing in

The portal delegates login to the identity provider: the first visit redirects
there, and after authenticating you land back on the dashboard. Behind that, the
OIDC token is exchanged **once** for a platform token, which is what every
subsequent API call carries and what the interface reads your roles from. You do
not see this, but it explains two things: signing out of the identity provider
elsewhere does not immediately empty the portal, and a session that has expired
sends you back to login rather than showing errors.

## Dashboard

The landing page carries the **active announcements** — the operators'
maintenance windows and notices, shown while they are current — and the
**reservation calendar** for the coming weeks, so the first thing anyone sees is
whether the machine is available.

## My Projects

One row per project you belong to:

| Column | What it is |
|---|---|
| Name | The project name. This is the string the SDK sends as `project`. |
| Organization | The organization the project belongs to. |
| Remaining budget | QPU time left, as a duration. A free-queue project shows its budget but does not consume it. |
| Start, End | The project's validity window. Outside it, submissions are refused. |

If you are the administrator of a project, that row also carries **Add users**,
which is the fastest path to granting a colleague access — the same form as the
Admin section's membership management, pre-filled with that project.

The project name matters beyond this page: it is what
[authorization](../architecture/job-lifecycle.md) matches a submission against,
and what a job is billed to.

## My Jobs

Your own jobs, 25 rows to a page and sortable by any column:

| Column | Notes |
|---|---|
| Id, Job id | The platform's own row id, and the machine's job identifier — the second is the one to quote in a support request. |
| Project, User | Attribution. On this page the user is always you. |
| Job type | `circuit` or `sweep`. |
| Status | As last reported by the gateway. |
| Execution start, Execution end | The executed window. Their difference is what billing counts. |
| Submitted | When the job was accepted. |
| **Circuit**, **Results** | Buttons to the stored artifacts. |

**Circuit** is the payload as submitted; **Results** is what came back. Both open
the object store directly, so a link keeps working outside the portal — which is
what makes them worth pasting into a notebook or a ticket. A job that has not
produced the artifact shows a muted dash rather than a dead button, so "not yet"
and "broken" look different.

Statuses come from the machine through the gateway. A job that never reached
execution keeps empty timestamps and contributes nothing to billing.

## Machine Status

A card per monitor — the frontend API, the dashboard, the cryostat, the backend,
whatever the deployment watches — each showing its current state and its uptime.
The cards are rendered by the deployment's status page and simply displayed
here, so this panel cannot drift from the status site itself. It refreshes on a
timer.

There is an **Open in new tab** link to the full status page, which carries the
incident history and per-monitor detail that a card cannot.

## Machine Metrics

The deployment's monitoring overview, embedded: calibration figures — readout
and gate fidelities — cryostat temperature, and machine occupancy over recent
days.

That page is built as a single-screen wall display, so it fills the panel rather
than scrolling inside it. On a short window the portal page scrolls instead. As
with the status panel, **Open in new tab** gets the page on its own, which is
also where its own navigation works.

!!! note
    Both panels appear only where the deployment has configured their URLs
    (`VITE_MACHINE_STATUS_URL`, `VITE_MACHINE_METRICS_URL`). A deployment
    without a monitoring stack simply does not have them, rather than showing
    an empty frame.

## Documentation

A link, not a panel: it opens this documentation site in a new tab. A separate
site with its own navigation is a poor fit for a frame — the same reason the
status page is shown as cards instead.

## Profile

Reached from the account menu in the top bar. Two things live here.

**Your default project.** Pick one and save it, and submissions that do not name
a project are attributed to it. This is worth setting once: the SDK's project
field is easy to forget, and without a default a nameless submission is refused
— *unless* it lands inside a reservation of a project you belong to, which is
authorized on the strength of the reservation alone. Only projects that are
currently valid are offered; if your saved default has since expired, the page
says so and asks you to choose another.

**A machine token.** *Get token* returns the access token of your current
session, which is what the vendor SDK presents to the gateway. Copy it into the
SDK's configuration. It expires with the session, so fetch a fresh one when the
SDK starts reporting authentication failures — it is a credential for the
duration, so treat it like one.

## Notifications

The bell in the top bar carries a count of unread notifications and opens the
recent ones; each can be marked read from there. The notifications page also has
**Mark all as read**, which reports how many of how many succeeded rather than
failing silently — a notification that is not addressed to you cannot be marked,
and the count says so instead of the list quietly not refreshing.

Announcements are the other half of the same story and are not the same thing:
an announcement is broadcast by the operators and appears on the landing page
while it is current; a notification is addressed to you.
