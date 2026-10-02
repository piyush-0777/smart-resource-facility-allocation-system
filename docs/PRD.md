# Product Requirements Document — WorkFlow

**Smart Resource & Facility Allocation System**
Odoo × LDCE Hackathon — Offline Round

| Field | Value |
| --- | --- |
| Product name | WorkFlow |
| Domain | Enterprise Productivity & Resource Management |
| Team size | 2 developers |
| Timebox | 8 hours |
| Status | MVP specification (no implementation yet) |

---

## 1. Problem Context

Shared physical resources in campuses and mid-sized offices — conference rooms, high-end
testing devices, company vehicles, projectors — are managed through fragmented WhatsApp
groups, static Excel sheets, and paper registers.

The consequences of that fragmentation are concrete and recurring:

- **Double bookings** — two teams claim the same room at 10:00 because nobody sees the other request.
- **Untracked asset damage** — no record of who had a device last, so damage is unattributed.
- **Utilization bottlenecks** — assets sit idle while popular ones are oversubscribed.
- **No accountability** — overdue returns are never chased because nothing tracks end-time.
- **Slow approvals** — a request sits in a personal inbox instead of a queue.

## 2. Objective

Build **WorkFlow**, a centralized, role-based web application that manages the end-to-end
lifecycle of shared resources:

1. Discover assets that are available right now.
2. Request them for a date/time range.
3. Prevent overlaps before they are ever committed.
4. Approve or reject through a strict workflow.
5. Track maintenance, overdue returns, and utilization.

WorkFlow is the single source of truth that replaces the spreadsheet.

## 3. Target Audience & Roles

| Role | Description | Primary jobs |
| --- | --- | --- |
| **Standard User** (Employee / Student) | Books and returns assets | Browse catalog, check availability calendars, create bookings, report issues |
| **Administrator** (Facility Manager) | Owns the inventory | Approve/reject requests, add inventory, mark assets "Under Maintenance", view utilization analytics |

Authorization is enforced on the server. Hiding a button in the UI is not access control.

## 4. Core Functional Requirements (MVP)

### FR-1 Role-Based Access Control (RBAC)
- JWT-based auth (signup, login, logout, refresh).
- Passwords hashed with bcrypt (12 rounds).
- Every API route declares a required role: `public`, `user`, `admin`.
- Admin routes reject `user` tokens with `403` regardless of what the client sends.
- No role can be self-assigned at signup — role is `user` by default and only an admin can change it.

### FR-2 Resource Catalog
- Grid/list of assets with: name, category, image, location, status, capacity/tags.
- Live status filter: `Available`, `Booked`, `Under Maintenance`.
- Search by name and category; filter by category and status.
- Pagination or infinite scroll (≥50 seeded assets).

### FR-3 Conflict-Free Booking Engine *(Critical)*
- User picks **start time/date** and **end time/date**.
- **The backend must explicitly validate and reject any booking that overlaps an already
  approved booking for that specific asset.**
- Overlap rule: `existing.start < requested.end AND existing.end > requested.start`.
- Pending bookings do not block; only `approved` and `active` windows block.
- Validation is enforced twice — a Postgres exclusion constraint (defence in depth against
  races) and an explicit pre-check in the service layer so the user gets a readable error.
- Rejections return `409 Conflict` with the conflicting booking's window.

### FR-4 State Machine (Workflow)
A booking follows a strict, one-directional status flow:

```
                ┌──────────────► Rejected (terminal)
                │
Pending ──approve──► Approved ──start──► Active ──complete──► Completed
   │                       │
   └───reject──────────────┘

(any non-terminal, not started) ──cancel by owner──► Cancelled
Admin may move Approved/Active ──► Cancelled (force release).
```

Legal transitions:

| From | Allowed `to` | Who |
| --- | --- | --- |
| `pending` | `approved`, `rejected`, `cancelled` | admin / owner |
| `approved` | `active`, `cancelled` | admin (or system cron for auto-start) |
| `active` | `completed`, `cancelled` | admin |
| `completed`, `rejected`, `cancelled` | — terminal | — |

Illegal transitions return `409` and are never silently applied.

### FR-5 Admin Dashboard
Single-page view with:
- Metric cards: **Total Assets**, **Pending Requests**, **Active Bookings**, (bonus) **Overdue**.
- Pending requests list with inline **Approve** / **Reject** (with reason).
- Quick actions: add asset, mark under maintenance.
- Utilization analytics (bonus): bookings per asset, top categories.

### FR-6 Issue Reporting
Standard users report an issue against an asset ("Projector bulb broken").
Admins see an issue queue with severity and can mark it resolved.

## 5. Advanced / Bonus Features (To Stand Out)

| # | Feature | Description | Priority |
| --- | --- | --- | --- |
| B-1 | Penalty / Overdue logic | Auto-flag users who haven't returned past end-time; show an overdue badge and log a `penalty_points` increment | High |
| B-2 | Real-time notifications | Alert the user in the UI when an admin approves their request (WebSocket or Supabase Realtime subscription) | High |
| B-3 | Data export | Admin exports booking history of an asset as CSV/PDF | Medium |
| B-4 | Email notifications | Nodemailer sends approval/rejection/overdue emails (SMTP) | Medium |
| B-5 | Asset images | Multer upload of asset photos, stored in Supabase Storage | Medium |
| B-6 | Usage heatmap | Day-of-week × hour utilization grid per asset | Low |

## 6. User Journeys

### 6.1 Standard User — book an asset
1. Sign up / log in → redirected to `/catalog`.
2. Filter catalog by category/status → open asset detail.
3. Pick a date + start/end time on the availability calendar.
4. Live preview shows which existing bookings overlap.
5. Submit → `POST /api/bookings` → status `pending`, success toast.
6. Navigate to **My Bookings**; badge shows pending count.
7. Admin approves → real-time toast + email → status `approved`.
8. User starts the booking (or cron auto-starts) → `active`.
9. At end-time, user marks `completed`, or the system flags **overdue** after grace period.

### 6.2 Administrator — approve a request
1. Log in → `/admin` shows Pending Requests = N.
2. Open a request: requester, asset, window, and any overlap warning.
3. **Approve** → conflict check re-runs atomically → `approved`, user notified (UI + email).
4. **Reject** → reason required → `rejected`, user notified.
5. Optionally mark the asset **Under Maintenance** (blocks new bookings immediately).

### 6.3 Overdue flow (bonus)
1. A cron/scheduled job runs every 5 minutes.
2. Bookings with `status = active` and `end_time < now() - GRACE_MINUTES` are flagged `overdue = true`.
3. The user's dashboard shows a red overdue banner and +penalty points.
4. Admin dashboard surfaces an **Overdue** metric card.

## 7. Out of Scope (for the 8-hour build)

- Mobile native apps; QR/RFID check-in hardware.
- Recurrence rules beyond "weekly, N times".
- Multi-tenant organizations / SSO.
- Real-time video or chat.

## 8. Success Criteria

| Metric | Target |
| --- | --- |
| Overlapping approved bookings for the same asset | **0** (verified by SQL count + concurrency test) |
| Unauthorized admin access | **0** (verified by automated RBAC tests) |
| Core MVP flows demo-able end-to-end | 100% (catalog, booking, approval, dashboard) |
| Seed data | ≥ 20 users, ≥ 50 assets, ≥ 100 bookings across statuses |
| Pitch | Covers stack reasoning, flow diagram, user journey, edge cases |

## 9. Requirement Traceability

| Brief requirement | Implementation area |
| --- | --- |
| RBAC, no URL manipulation | `server/src/middleware/auth.js`, `server/src/middleware/rbac.js` |
| Resource catalog + live status | `client/src/features/catalog`, `GET /api/assets` |
| Conflict-free booking | `server/src/services/booking.service.js`, Postgres `EXCLUDE USING gist` |
| State machine | `server/src/services/booking.service.js` transition map |
| Admin dashboard | `client/src/features/admin`, `GET /api/admin/metrics` |
| Penalty/overdue | `server/src/jobs/overdue.job.js` |
| Real-time notifications | `server/src/realtime`, Supabase Realtime channel |
| Data export CSV/PDF | `GET /api/assets/:id/bookings.csv` |
| Dynamic data (no static JSON) | Supabase Postgres — every core read/write goes through the API |