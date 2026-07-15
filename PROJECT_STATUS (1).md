# Ardent OS — Project Status

## Purpose
Replacement for ZenMaid for Ardent Living (cleaning business), built by Simputic.

## Architecture

**Frontend:** React 19 + Vite + Tailwind v4 + React Router (HashRouter) + Recharts + lucide-react

**Current data flow (as of this build):**
Pages import data directly from `src/data/mockData.js`. There is no state
management library (no Redux) — components use local `useState` for UI state
(filters, search, calendar view) and read shared arrays straight from the
mock data module.

**Target data flow:**
Airtable becomes the single source of truth. No business data should live
inside React source — the frontend only displays data returned by the API layer.

**Security requirement (non-negotiable):**
The Airtable connection is accessed only through a secure server-side API
layer. Airtable tokens are never present in browser code or in any
`VITE_`-prefixed environment variable, since Vite inlines those into the
client bundle at build time.

**UI requirement:** the current UI/design language is preserved exactly
during the data-layer migration. This is a backend swap, not a redesign.

## Data Source

Airtable tables:
- Clients
- Staff
- Job Appointments
- Job Assignments
- Quality Checks
- Invoices

**Relationships:**
```
Clients
  ↓
Job Appointments
  ↓ (primary: Cleaners link field)
Staff
  ↑ (fallback only: Job Assignments)
```
- One Job Appointment = one customer visit/job.
- **Primary cleaner relationship:** `Job Appointments.Cleaners` is a real Airtable
  linked-record field pointing straight at Staff. This is the relationship used
  for appointment cleaner badges, cleaner colours, staff schedules, today's/upcoming
  jobs, and the appointment detail panel. It requires no ID-matching — Airtable
  gives us the linked Staff record directly.
- **Job Assignments is a fallback path only**, used only when a Job Appointment
  has no linked Cleaners value and a cleaner must be recovered some other way.
  The base was originally modelled around a ZenMaid export, so Job Assignments
  and Job Appointments both carry duplicated flat fields (Customer ID,
  Appointment Date, Start Time, etc.) rather than a real link between them.
  There is currently no linked-record field or shared ID connecting Job
  Assignments to Job Appointments.
- **Fallback composite match** (used only when Cleaners is empty):
  `Job Assignments.Customer ID + Appointment Date + Appointment Start Time`
  ↔ `Job Appointments.Customer ID + Appointment Date + Start Time`.
  When this fallback is used: cleaner IDs recovered this way are deduplicated,
  no duplicate appointment records are created, and unmatched or ambiguous
  matches (e.g. more than one Job Appointment sharing the same composite key)
  are flagged in logs rather than silently guessing a cleaner.
- Cleaner colours come from Staff data or a stable frontend mapping — never regenerated per render.
- Subscription ID is never used as the unique appointment link.
- Date/time fields used: `Appointment Date`, `Start Time`, `End Time`. The
  "Latest Appointment…" fields and the unlabelled Staff single-select field
  are ignored — their meaning hasn't been confirmed.

## Current Status

- UI ~95% complete (Calendar, Dashboard, Staff complete; Clients nearly
  complete; Quality Control scaffold complete)
- `npm run build` and `npm run lint` both pass clean
- Backend: none yet — all data currently mocked in `src/data/mockData.js`
- `mockData.js` is retained (not deleted) during migration, to allow
  switching between mock and live data via `VITE_USE_MOCK_DATA`

## Data Abstraction Layer (in progress)

A data-access layer sits between pages and their data source, so pages never
import `mockData.js` or an Airtable client directly. It reads the
`VITE_USE_MOCK_DATA` flag to decide whether to return mock data or call the
live API layer. `VITE_USE_MOCK_DATA` only ever holds `true`/`false` — no
secrets travel through it.

## Next Development — Phase 1: Airtable Integration (Calendar data flow)

Priority: calendar/scheduling is the most important operational feature, so
it is connected first — not Clients.

Phase 1 connects two tables directly:
1. Staff
2. Job Appointments (using the `Cleaners` linked-record field as the primary
   source of which cleaner(s) are on a job)

Job Assignments is **not** a required Phase 1 endpoint. It is only queried as
a fallback, per-record, when a Job Appointment has no `Cleaners` value and a
cleaner must be recovered via the composite match described above. Fallback
usage is logged, deduplicated, and never guesses when ambiguous.

Everything else (Clients, Quality Checks, Invoices) stays on mock data until
later phases.

## Deployment

- GitHub → Netlify
- Airtable access via a server-side API layer (see integration plan) —
  approach to be finalized before Phase 1 code is written

## Phase 2 — Backend Cleanup (after live frontend is stable)

Priority in Phase 2 is optimization, not stability — the reverse of Phase 1.
Planned schema work:
- Add a proper linked-record field on Job Assignments pointing directly at
  Job Appointments, replacing the composite Customer ID + Date + Start Time
  fallback match entirely.
- Once that link exists, remove reliance on composite matching in code.
- Review and archive redundant fields inherited from the original ZenMaid
  export (duplicated flat fields on Job Assignments/Job Appointments that
  exist only because no real link was in place).

## Future

- Authentication
- Role permissions
- Offline capability
- Reporting
- Invoice integration
- Remaining Airtable table connections (Clients, Quality Checks, Invoices)
