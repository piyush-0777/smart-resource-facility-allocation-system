# Test Plan — WorkFlow

## 1. Scope

Covers the MVP core: authentication & RBAC, the resource catalog, the conflict-free
booking engine, the booking state machine, the admin dashboard, notifications, and bonus
overdue logic.

Out of scope for the 8-hour build: visual regression at pixel level, load testing beyond
a smoke check, Safari/Firefox full matrix (Chromium + responsive widths only).

## 2. Test Types

| Type | Tool | Purpose |
| --- | --- | --- |
| **Unit** | Vitest | Overlap predicate, state-machine transition map, penalty calculation, date utils |
| **Integration / API** | Vitest + Supertest | Real HTTP against the Express app and a real (test) Postgres database |
| **Component** | Vitest + React Testing Library | Booking form validation, status badges, role-gated rendering |
| **E2E (manual)** | Chrome + checklist | The full journey from login to completed booking |
| **Static** | ESLint + `tsc --noEmit`-equivalent (if typed) | Catches dead imports and unused slices before the demo |

No mocks for the booking rules: the whole point of Postgres is the constraint, so the
tests run against a real database (`supabase db reset` per run, or a `test` schema).

Request/response shapes and error codes referenced by the cases below are specified in
`docs/API.md` — the cases assert against that contract.

## 3. Test Data & Fixtures

`server/tests/fixtures.js` seeds deterministically:

- `users`: 1 admin, 3 standard users (`admin@workflow.test`, `user@workflow.test`, …).
- `categories`: Meeting Room, AV Equipment, Vehicles, Lab Devices, Projectors.
- `assets`: ~50 across categories, including 2 in `maintenance` and 1 pre-booked today.
- `bookings`: ~120 spanning every status, plus overlapping pairs and an overdue one.
- `issues`: open + resolved samples.

Ids are fixed (`user_admin`, `asset_room_a`, …) so tests never depend on ordering.

## 4. Authentication

| ID | Case | Expected |
| --- | --- | --- |
| A-1 | Signup with valid payload | `201`, `role = 'user'`, no password in response |
| A-2 | Signup with `{ role: 'admin' }` in body | `201`, `role` still `user` (mass assignment stripped) |
| A-3 | Signup with weak password (< 8 chars) | `422 VALIDATION_ERROR` |
| A-4 | Signup with duplicate email | `409 EMAIL_TAKEN` |
| A-5 | Login with correct credentials | `200` + JWT with `sub` and `role` |
| A-6 | Login with wrong password / unknown email | `401 INVALID_CREDENTIALS` (identical body both times) |
| A-7 | Request with malformed / tampered JWT | `401` |
| A-8 | Request with expired JWT | `401` |
| A-9 | Password hashing | bcrypt, 12 rounds; `profiles` table stores no plaintext |
| A-10 | Login rate limit | 11th attempt in 15 min → `429` |

## 5. RBAC (explicitly graded by the brief)

| ID | Case | Expected |
| --- | --- | --- |
| R-1 | `GET /api/admin/metrics` as **admin** | `200` |
| R-2 | `GET /api/admin/metrics` as **user** | `403` — user cannot reach the admin dashboard by URL |
| R-3 | `POST /api/admin/requests/:id/approve` as **user** | `403`, booking still `pending` |
| R-4 | `POST /api/assets` as **user** | `403` |
| R-5 | `PATCH /api/assets/:id` set `maintenance` as **user** | `403` |
| R-6 | `GET /api/assets/:id/bookings.csv` as **user** | `403` (admin-only export) |
| R-7 | Client-side: navigate to `/admin` as user | Redirected to `/catalog`; server call still `403` |
| R-8 | Every registered route declares a guard | Automated route-table walk passes |

## 6. Catalog

| ID | Case | Expected |
| --- | --- | --- |
| C-1 | `GET /api/assets` unauthenticated | `401` (catalog needs a session) |
| C-2 | Filter by `category` and `status` | Only matching assets returned |
| C-3 | Search `?q=projector` | Matches name/description case-insensitively |
| C-4 | Invalid `status=banana` | Stripped → default filter, no 500 |
| C-5 | Pagination | `page=1&limit=10` returns 10 + `total` |
| C-6 | `GET /api/assets/:id/availability?from&to` | Busy windows returned for the range |
| C-7 | Admin `POST /api/assets` with an image | `201`, `image_url` set, file in Supabase Storage |
| C-8 | Admin marks asset `maintenance` | Status changes; asset disappears from available results |
| C-9 | Invalid asset UUID | `404` (not `500`) |

## 7. Conflict-Free Booking Engine *(critical)*

| ID | Case | Expected |
| --- | --- | --- |
| B-1 | Book a free slot | `201`, status `pending` |
| B-2 | Book overlapping an **approved** booking (partial overlap) | `409 BOOKING_CONFLICT` + conflicting window in the body |
| B-3 | Book fully containing an approved booking | `409` |
| B-4 | Book fully inside an approved booking | `409` |
| B-5 | Book **abutting** an approved booking (ends exactly at start) | `201` — `[)` half-open intervals do not conflict |
| B-6 | Book overlapping a **pending** booking | `201` — pending does not block |
| B-7 | Two users book the same slot concurrently (10 parallel requests) | Both may be `pending`; on approval exactly **one** succeeds, one `409` |
| B-8 | Two admins approve overlapping requests concurrently | Same as B-7 — enforced by `EXCLUDE USING gist` + row lock |
| B-9 | `end_time <= start_time` | `422` |
| B-10 | Booking in the past | `422` |
| B-11 | Window > 14 days | `422` |
| B-12 | Booking an asset in `maintenance` | `409 ASSET_UNAVAILABLE` |
| B-13 | Booking a non-existent asset | `404` |
| B-14 | Purpose missing / 600 chars | `422` |
| B-15 | SQL check after the suite | `select count(*) from bookings where a.asset_id=b.asset_id and a.id<>b.id and a.slot && b.slot and a.status in ('approved','active') and b.status in ('approved','active')` → **0** |

## 8. State Machine

| ID | Case | Expected |
| --- | --- | --- |
| S-1 | `pending → approved` by admin | `200`, `approved_by` and `decided_at` set |
| S-2 | `pending → rejected` with reason | `200`, reason stored |
| S-3 | `approved → active` | `200`, `started_at` set |
| S-4 | `active → completed` | `200`, `completed_at` set, asset released |
| S-5 | `approved → completed` (skip) | `409 INVALID_TRANSITION` |
| S-6 | `completed → active` (terminal) | `409` |
| S-7 | `rejected → approved` | `409` — rejection is terminal |
| S-8 | User cancels own `pending` booking | `200`, status `cancelled` |
| S-9 | User cancels **another user's** booking | `403` (ownership) |
| S-10 | User completes someone else's active booking | `403` |
| S-11 | Admin force-cancels an `active` booking | `200`, asset freed, user notified |
| S-12 | Transition on a non-existent id | `404` |
| S-13 | Approving a request for a now-maintenance asset | `409` (block at decision time too) |
| S-14 | After completion, the same slot is bookable | `201` |

## 9. Admin Dashboard

| ID | Case | Expected |
| --- | --- | --- |
| D-1 | Metrics for seeded data | Total Assets / Pending / Active counts match direct SQL counts |
| D-2 | Utilization breakdown | Non-zero buckets for seeded bookings, sums to total |
| D-3 | Pending queue pagination | Page 2 returns different ids |
| D-4 | Approve from the queue | Row disappears, Pending metric decrements |
| D-5 | Reject without a reason | `422` |
| D-6 | Empty state (no pending requests) | EmptyState component, not a blank panel |

## 10. Notifications & Email

| ID | Case | Expected |
| --- | --- | --- |
| N-1 | Approval → `notifications` row for the booking owner | Row exists, `read_at` null |
| N-2 | Approval → email attempted | `sendMail` called (mocked transport in tests, real SMTP in dev/staging log) |
| N-3 | Rejection → notification + email containing the reason | Reason present in the body |
| N-4 | Realtime broadcast on approval | Client store receives the event; toast appears |
| N-5 | `GET /api/notifications` returns only the caller's | No cross-user leakage |
| N-6 | `POST /api/notifications/:id/read` | `read_at` set; badge count decreases |

## 11. Overdue / Penalty (bonus)

| ID | Case | Expected |
| --- | --- | --- |
| O-1 | Active booking past end-time + grace | `is_overdue = true`, user notified |
| O-2 | Booking inside the grace period | Not flagged |
| O-3 | Overdue increments `penalty_points` | By exactly 1 |
| O-4 | Job re-runs twice | Idempotent — no double penalty, no duplicate notification |
| O-5 | Booking completed before end-time | Never flagged overdue |
| O-6 | Approving a booking whose window already passed | `422` |

## 12. Uploads

| ID | Case | Expected |
| --- | --- | --- |
| F-1 | Valid 1 MB PNG | `201`, stored, correct `image_url` |
| F-2 | `image/svg+xml` | `415 UNSUPPORTED_FILE_TYPE` |
| F-3 | 6 MB file | `413` |
| F-4 | Two files in one request | `400` (files: 1) |
| F-5 | Filename with `../` or spaces | Stored under a server-generated UUID name only |

## 13. Manual E2E Script (demo rehearsal)

1. Register `demo@workflow.test` → lands on catalog, empty state visible.
2. Filter to *Meeting Room*, open **Board Room A**, see today's busy windows.
3. Try to book 10:00–11:00 when 10:30–11:30 is approved → conflict modal shown.
4. Book 14:00–15:00 → success toast, visible under My Bookings as **Pending**.
5. Log out; log in as admin → dashboard Pending = 1.
6. Approve it → toast + metric drops; the user account shows a real-time toast.
7. Try navigating to `/admin` as the user → bounced to catalog; direct API call returns `403`.
8. Start the booking → Active; mark the asset `maintenance` → new bookings for it are blocked.
9. Export booking history CSV → opens in a spreadsheet.
10. Switch light ⇄ dark; toggle to system; confirm no white-flash and readable contrast.

## 14. Execution

```bash
# server
cd server && npm test            # unit + integration, resets test DB first
npm run test:watch

# client
cd client && npm test            # Vitest + RTL
npm run lint
```

## 15. Entry / Exit Criteria

**Entry:** API routes implemented, migrations applied, fixtures loading.

**Exit:** all P0 cases (A-*, R-*, B-*, S-*) green; B-15 returns 0; no `console.log`
secrets; manual E2E script completed end-to-end without a blocker; coverage ≥ 80 % lines on
`server/src/services/**`. Known issues logged in `MEMORY.md` rather than discovered live.