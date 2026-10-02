# Decisions

Architecture Decision Records for WorkFlow. Newest last. Each entry: context → decision →
consequences.

---

## D-001 — Supabase Postgres as the only datastore
**Date:** 2026-10-02 · **Status:** Accepted
**Context:** The brief bans static JSON and requires real overlap queries; Supabase also
gives Storage (asset images) and Realtime (notifications) for free.
**Decision:** Postgres via Supabase. All reads/writes go through the Express API using the
service-role key; the browser holds only the anon key.
**Consequences:** No dual-source-of-truth bug. RLS is currently unused (single trusted
backend) — documented as the migration path if a client ever queries Supabase directly.

## D-002 — Exclusion constraint instead of application-only overlap checks
**Date:** 2026-10-02 · **Status:** Accepted
**Context:** "SELECT for conflicts, then INSERT" is vulnerable to a TOCTOU race: two admins
can both see a free slot and both insert.
**Decision:** Keep the readable pre-check for good error messages, but add
`EXCLUDE USING gist (asset_id WITH =, slot WITH '[)' tstzrange) WHERE status IN ('approved','active')`,
plus `SELECT … FOR UPDATE` around approval.
**Consequences:** Double booking becomes physically impossible; requires `btree_gist`.
Requires a small migration step the frontend devs shouldn't have to reason about.

## D-003 — Half-open intervals `[)`
**Date:** 2026-10-02 · **Status:** Accepted
**Context:** If a booking ends at 11:00, should one starting at 11:00 conflict?
**Decision:** No. Intervals are `[start, end)`, so abutting bookings are allowed.
**Consequences:** Matches how humans schedule rooms. Documented in `TEST_PLAN.md` (B-5) so
the behaviour is intentional rather than accidental.

## D-004 — Custom JWT auth rather than Supabase Auth
**Date:** 2026-10-02 · **Status:** Accepted
**Context:** Supabase Auth would work, but the demo benefits from an auth flow we can
explain line-by-line, and we need `profiles.role` governed by our own API.
**Decision:** Supabase Auth only for identity; a `profiles` table mirrors `auth.users`, and
our API issues HS256 JWTs (`jsonwebtoken` + `bcrypt`).
**Consequences:** Full control over the RBAC story; we own token rotation and password
reset. Trade-off: refresh/recovery flows are ours to build.

## D-005 — Server-enforced RBAC, client guards as UX only
**Date:** 2026-10-02 · **Status:** Accepted
**Context:** The brief explicitly grades "users should not access the admin dashboard via
URL manipulation" — a client-side redirect cannot satisfy that.
**Decision:** Every protected route declares `auth` + `requireRole(...)`; React Router has
`<RequireAuth>`/`<RequireAdmin>` purely for navigation comfort.
**Consequences:** Tests must exercise RBAC over HTTP (TEST_PLAN §5). Role can never be
self-assigned because signup strips unknown body fields.

## D-006 — Booking status as an enum + a transition map
**Date:** 2026-10-02 · **Status:** Accepted
**Context:** Status strings scattered across controllers drift and allow illegal jumps
(`approved → completed`).
**Decision:** Postgres enum `booking_status` plus one transition map in
`booking.service.js`; `transition()` validates before writing.
**Consequences:** Illegal jumps fail loudly with `409`. Adding a status means editing the
enum, the map, and the UI badge together.

## D-007 — Redux Toolkit for cross-cutting state, not for all server data
**Date:** 2026-10-02 · **Status:** Accepted
**Context:** An 8-hour build doesn't warrant RTK Query's caching layer, but booking and
notification state genuinely crosses routes (catalog → my bookings → toast).
**Decision:** Four slices — `auth`, `bookings`, `notifications`, `ui` (theme, toasts,
sidebar). Assets are fetched by page into local component state with an optimistic insert
for newly created bookings.
**Consequences:** Minimal boilerplate, fast onboarding for the second developer, no
cache-invalidation puzzles. Migrating to RTK Query later is straightforward if the API
surface grows.

## D-008 — Real Apple-style glassmorphism, class-based dark mode
**Date:** 2026-10-02 · **Status:** Accepted
**Context:** Brief demands a clean, consistent, responsive UI; the team wants the premium
"liquid glass" look with both light and dark themes.
**Decision:** Custom Tailwind glass utilities (blur + saturate + inset highlight) over a
drifting gradient mesh; `.dark` class on `<html>` swaps CSS custom properties;
`darkMode: 'class'`. Theme = light / dark / system, persisted.
**Consequences:** Distinctive presentation. Costs: blur is GPU-expensive, so blur radius is
capped (36 px) and never stacked more than ~3 deep; an inline script in `index.html`
prevents a white flash before hydration.

## D-009 — Nodemailer for approval/rejection/overdue mail
**Date:** 2026-10-02 · **Status:** Accepted
**Context:** Users should learn the outcome without watching the app.
**Decision:** Nodemailer over SMTP (dev: Ethereal or console transport). Mail failure never
fails the booking transaction — it is fire-and-forget after commit.
**Consequences:** Bonus-feature credibility for near-zero cost. Requires SMTP env vars on
the host; without them the app logs a warning and continues.

## D-010 — Multer to a temp dir, then Supabase Storage
**Date:** 2026-10-02 · **Status:** Accepted
**Context:** Asset images are needed for the catalog to look credible; direct-to-storage
uploads complicate progress and validation in a timed build.
**Decision:** Multer disk storage with MIME allow-list and a 5 MB cap; the controller then
uploads to Supabase Storage and stores the public URL.
**Consequences:** Server-generated UUID filenames kill path traversal. The `uploads/`
directory is gitignored and ephemeral (not durable storage).

## D-011 — node-cron for the state machine's time-based edges
**Date:** 2026-10-02 · **Status:** Accepted
**Context:** Auto-starting approved bookings and flagging overdue ones needs a scheduler.
**Decision:** In-process `node-cron` (1 min auto-start, 5 min overdue). Each job is
idempotent and individually try/caught.
**Consequences:** Fine for a single instance. Documented: a second replica would double-run
jobs, so jobs must move to a queue/managed scheduler before horizontal scaling.

## D-012 — Vitest + Supertest with a real test database
**Date:** 2026-10-02 · **Status:** Accepted
**Context:** The most important behaviour (no overlapping approved bookings) is a database
constraint; mocking Postgres would test nothing.
**Decision:** Vitest (shares Vite's config) + Supertest against the Express app and a
disposable Supabase test schema, reset before the suite.
**Consequences:** Slower than pure unit tests but catches the race conditions that actually
matter. Parallel workers are limited to avoid fixture collisions.

## D-013 — AI assistance is a scaffold, not an author
**Date:** 2026-10-02 · **Status:** Accepted
**Context:** The brief warns that the team must explain its own code during the
presentation.
**Decision:** Use tooling to scaffold files and boilerplate fast, but the booking
overlap logic, the state machine, and the RBAC guard are written and understood by the
team, and are the first things rehearsed in the presentation.
**Consequences:** "Understand your code" is treated as a build constraint: if we can't
explain a block, we delete it rather than ship it.