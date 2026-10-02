# Administration

The **Admin** section of the sidebar. Every page here is scoped by the backend to
what the caller may reach, so the same list serves a platform admin, an
organization manager and a principal investigator, showing three different sets
of rows.

Lists show 25 rows per page throughout.

## Reservations and Reservations Calendar

Two views of the same bookings: a table and a calendar. They are administrative
views — everything the caller may reach — which is why they sit here rather than
above the Admin header, and they are open to **project admins and above**.

The table gives one row per reservation with its project, who made it, the slot
it sits in, the day, the window and the description. The calendar gives the same
information as occupancy, which is the better shape for answering "is next
Tuesday free".

Both calendars — this one and the slots calendar — carry a **Jump to date**
field. Slots are planned months ahead, and stepping a week per click told you
nothing about where you were going; picking or typing a date moves the view to
the week containing it and refetches, exactly as Back and Next do.

**What a reservation costs.** Booking one debits the project's budget for the
whole window immediately, at reservation time — not when jobs run inside it. That
is why a member of the reserving project can submit during the window even with
no budget left: it was already paid for. Cancelling refunds it in full if more
than 24 hours ahead, and 25% otherwise. A reservation cannot be placed outside
its project's validity window.

Which in turn is why the [billing reports](#billing-reports) count reserved hours
and job hours separately, and exclude jobs that ran inside a reservation from the
job hours: that time is already on the invoice once.

## Projects

The projects the caller administers: all of them for a platform admin, the
organization's for a manager, the administered ones for a PI. Creating and
editing a project is platform-admin and organization-manager work; a PI can edit
the projects they administer.

A project's page is tabbed:

| Tab | Contents |
|---|---|
| Project details | Name, organization, budget, validity window, free-queue flag |
| Project Users | Membership, with add and remove — and a bulk path for adding many at once |
| Project Reservations | The project's bookings, with **Add reservation** |
| Budget Transactions | The ledger: every debit and credit against this project's budget, with its cause |
| Jobs | The project's jobs, with the same Circuit and Results buttons as My Jobs |
| Tags | Assigning tags from the vocabulary (see [Tags](#tags)) |

**Budget Transactions is the authoritative answer to "where did the budget go".**
It is a ledger, not a summary: reservations, refunds, job charges and manual
adjustments each appear as an entry. A billing report aggregates the same
underlying facts for accounting; the ledger is what to read when a single number
looks wrong.

## Users

Platform admins and organization managers. Create a user, set their
organization, and toggle the flags that are not mirrored from the identity
provider. There is a bulk creation path for onboarding a cohort at once.

Two labels differ from the field behind them, deliberately:

- **Username** is the identity field (`users.email`), so that deployments whose
  identities are not e-mail addresses are not lying to their users.
- **Pulse access** is sweep-submission access, backed by `users.pulla_user`, and
  the toggle is visible to platform administrators only. A new user starts with
  whatever the identity provider gave them; from then on this is where it is
  granted and revoked, and it is what the machine actually checks when a sweep is
  submitted. See
  [platform roles](../architecture/identity.md#platform-roles-how-the-backend-derives-them).

Roles themselves are not granted here. They come from the identity provider.

## All Jobs

Every job on the platform, for platform admins: the same columns as
[My Jobs](day-to-day.md#my-jobs) without the user filter. This is the page for
"what has the machine been doing", and the Circuit and Results buttons make it
the page for reproducing a user's complaint.

## Allocated slots and Slots Calendar

Present only where the deployment is **slot-constrained** — that is, where
reservations must sit inside a pre-allocated slot. The backend advertises this
through its public `GET /config`, and where reservations are unconstrained these
two items disappear rather than showing an empty concept.

A slot allocates a window of machine time to an organization; reservations are
then placed inside it by that organization. The calendar is usually the view you
want, for the same reason as the reservation calendar, and it carries the same
**Jump to date** field.

## Organizations

Platform admins. An organization owns projects and users, and may reference a
parent organization — which groups it for slot eligibility without merging its
accounting. Counts and consumption stay strict: a parent does **not** absorb its
children's jobs, reservations or budget. An organization's page also lists its
users, projects and jobs.

## Billing reports

For platform admins, and for organization managers and auditors — who see their
own organization, enforced by the backend. Not PIs: a project's consumption is
the budget ledger, not an accounting report.

The page produces the figures a *consuntivo* is built from, computed by the
backend rather than by exporting the database and aggregating it elsewhere.

1. **Choose a period** — start and end date.
2. **Narrow it** — per-project and per-organization checkboxes, with select-all
   and clear. **Enable organization reports** adds the organization-level tables.
3. **Generate reports.**
4. **Export** — CSV per table, or **Full HTML report** for all of them in the
   layout the report has always had.

Two display options change what you read, not what was computed:

- **Hide technical columns** drops `free_queue`, `organization_id` and
  `project_id` — the columns that matter for reconciliation and clutter a report
  meant to be read.
- **Round durations to the second** is for a readable report. Off, durations
  carry three decimals — thousandths of a second, which is the granularity
  billing actually runs at. Hours are derived from whichever of the two is
  showing, so the columns of a row can never disagree.

The tables produced are reservations and jobs (each by project and by
organization), slots by organization, and a reservation-utilization table
comparing time booked against time used — which is the one that tells you
whether the reserved hours were worth reserving.

## Tags

A controlled vocabulary of labels for projects, maintained by platform admins:
create, rename, delete. The point of the vocabulary being admin-defined is that
project administrators pick from it rather than inventing terms by typo, so the
labels stay aggregatable.

Assignment happens on a project's **Tags** tab and is open to platform admins,
organization managers, and the administrators of that project. A PI who cannot
submit the project form at all can still tag their own project — which is why
assignment is a control of its own rather than a field on the project form.
Saving replaces the whole set, so clearing the selection clears the tags.

Two refusals carry information you cannot get anywhere else, and are shown
verbatim rather than as a generic error:

- creating a tag whose name is taken names the existing one;
- deleting a tag still assigned to projects says how many carry it. Remove it
  from them first — the vocabulary will not drop a label out from under the
  projects using it.

Tags show as chips in the projects list and on a project's page, and are exposed
per-tag project counts on the [metrics endpoint](../reference/quantum-api-http.md#get-metricsjobs).

## Logs

Platform admins. The action log: who changed what, and when. Read-only by
design — an audit trail that can be edited is not one.

## Announcements

Platform admins. Operational messaging with a validity window: an announcement
appears on every user's landing page while it is current and disappears on its
own afterwards. This is the channel for a maintenance window; a notification is
the channel for something addressed to one person.
