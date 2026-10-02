# Memory — WorkFlow

Durable context for future sessions. Update this file whenever something non-obvious is
decided or discovered. Details live in the other docs; this file is the "what you need to
know before touching anything" note.

## What we are building
WorkFlow — a Smart Resource & Facility Allocation System for the Odoo × LDCE hackathon
(offline round, 8-hour timebox, 2 developers). Users browse shared assets (rooms, AV gear,
vehicles, lab devices), request them for a date/time window, and admins approve/reject.
Replaces WhatsApp groups and Excel sheets. Full spec: `docs/PRD.md`.

## Locked stack (do not substitute)
| Layer | Choice |
| --- | --- |
| Frontend | React 18, Redux Toolkit, React Router v6, Tailwind CSS, lucide-react |
| Build | Vite |
| Backend | Node.js + Express |
| Database | Supabase (Postgres) — real DB, no static JSON |
| Auth | jsonwebtoken + bcrypt, server-enforced RBAC |
| Email | Nodemailer |
| Uploads | Multer → Supabase Storage |
| Scheduling | node-cron |
| Tests | Vitest + Supertest |

## Non-negotiables from the brief
1. **Backend must reject overlapping bookings** for the same asset. Enforced by
   `EXCLUDE USING gist`, not just an application check.
2. **RBAC on the server.** A standard user hitting `/api/admin/*` must get `403`. Client-side
   route guards are cosmetic only.
3. **No static JSON for core data.** Everything goes through the API and Postgres.
4. **Consistent UI** with light/dark modes — we ship a real glass design system, not ad-hoc
   classes.
5. Team must be able to **explain the booking validation code** in the presentation.

## Mental model / gotchas
- Booking windows are **half-open `[start, end)`** — a booking ending at 11:00 does *not*
  conflict with one starting at 11:00. Intentional (D-003).
- Only `approved` and `active` bookings block a slot. Two users *can* both be `pending` for
  the same window; the fight is resolved at approval time, where exactly one wins.
- Status flow: `pending → approved → active → completed`, with `rejected` / `cancelled` as
  side exits. All transitions validated in `booking.service.js`; illegal jumps return `409`.
- Two developers split the work: **one owns server + DB** (schema, overlap logic, RLS), the
  **other owns client + UI** (components, slices, routing). Agree on the API surface in
  `docs/ARCHITECTURE.md` §6 before writing code — it is the contract.
- Glass UI cost: blur is GPU-expensive. Cap blur at 36 px and never stack more than ~3
  blurred layers. Dark mode is *not* an inversion — surfaces get darker and *more*
  transparent, blur increases.
- Theme is applied as a `.dark` class on `<html>` with an inline pre-hydration script to
  avoid a white flash.

## Where things live
- `docs/PRD.md` — requirements, user journeys, success criteria, traceability
- `docs/ARCHITECTURE.md` — stack reasoning, ERD, overlap algorithm, API surface, RBAC flow
- `docs/API.md` — **the frontend/backend contract**: every endpoint, request/response JSON,
  error codes, pagination, rate limits, realtime events, and a change log
- `docs/DESIGN.md` — glass design system, tokens, components, theming, a11y
- `docs/SECURITY.md` — threat model, auth, validation, uploads, verification checklist
- `docs/TEST_PLAN.md` — test IDs per feature, fixtures, manual E2E script
- `docs/DECISIONS.md` — ADRs; read before changing the stack

## Current state
Documentation complete; **no implementation started**. `client/` and `server/` contain only
`.gitkeep`. Next actions when coding begins:

1. Create the Supabase project and apply the schema from `ARCHITECTURE.md` §4
   (`supabase/migrations/001_init.sql`).
2. Agree on `docs/API.md` between both developers — the backend implements it, the frontend
   consumes it. Sign off before writing routes or components.
3. Seed fixtures matching `TEST_PLAN.md` §3 (≥ 50 assets, ≥ 120 bookings, 1 admin, 3 users).
3. Server: config → middleware (`auth`, `rbac`, `validate`, `upload`) → `booking.service`
   with the overlap logic → routes → `app.js`. Implement strictly against `docs/API.md`.
4. Client: glass primitives → auth slice + guards → catalog → booking wizard → admin dashboard.
5. Fill in the demo journey from `TEST_PLAN.md` §13 and rehearse it.

## Open questions
- Should `requires_approval = false` assets auto-approve (book instantly)? Cheap to add via
  the same transition map, but keep it out of the MVP demo path.
- CSV vs PDF export: CSV first, PDF only if time remains.
- Whether to seed a fixed demo dataset or generate it per run — fixed is better for a
  repeatable presentation.