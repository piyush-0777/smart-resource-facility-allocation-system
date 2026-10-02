# API Reference — WorkFlow

Complete contract for the Express backend. This is the agreement between the two
developers: **the frontend is built against this file, and the backend implements it.**
If anything here changes, both files change in the same commit.

| Field | Value |
| --- | --- |
| Base URL (dev) | `http://localhost:4000/api` |
| Base URL (client env) | `VITE_API_URL=http://localhost:4000/api` |
| Transport | HTTPS (JSON request/response) |
| Content type | `application/json` (`multipart/form-data` only for uploads, `text/csv` for exports) |
| Auth | `Authorization: Bearer <jwt>` |
| Time format | ISO 8601 UTC, e.g. `2026-10-05T09:00:00.000Z` |
| Timezone | All timestamps stored and compared in UTC; converted to local only in the UI |
| Version | `v1` (unversioned path, breaking changes ⇒ new deploy + doc header bump) |

---

## 1. Conventions

### 1.1 Authentication
All endpoints except the ones marked **public** require a JWT:

```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

Token payload: `{ sub: <user_id>, role: "user" | "admin", iat, exp }` — 7-day expiry.

### 1.2 Roles
| Role value | Who |
| --- | --- |
| `user` | Standard user (default for every signup) |
| `admin` | Administrator / facility manager |

### 1.3 Success envelope
Successful responses return the resource **directly** (no wrapper), except collections
and delete-style actions:

```jsonc
// single resource
{ "id": "…", "name": "Board Room A", … }

// collection
{
  "data": [ … ],
  "pagination": { "page": 1, "limit": 20, "total": 50, "totalPages": 3 }
}
```

Actions that return nothing meaningful: `204 No Content`.

### 1.4 Error envelope
Every non-2xx response:

```json
{
  "error": {
    "code": "BOOKING_CONFLICT",
    "message": "Asset is already booked for that window.",
    "details": { "conflict": { "start_time": "…", "end_time": "…" } },
    "requestId": "b3f1c0de-…"
  }
}
```

`message` is human-readable and safe to show in a toast. `details` is machine-readable and
optional. `requestId` appears in server logs for debugging.

### 1.5 Error codes

| Code | HTTP | When |
| --- | --- | --- |
| `VALIDATION_ERROR` | 422 | Body/query failed schema; `details.fields` maps field → message |
| `UNAUTHORIZED` | 401 | Missing, malformed, expired, or tampered token |
| `INVALID_CREDENTIALS` | 401 | Wrong email/password (identical for unknown email) |
| `FORBIDDEN` | 403 | Authenticated but role insufficient, or not the resource owner |
| `NOT_FOUND` | 404 | Resource does not exist (or is not visible to this user) |
| `METHOD_NOT_ALLOWED` | 405 | Wrong verb for the path |
| `EMAIL_TAKEN` | 409 | Signup with an existing email |
| `BOOKING_CONFLICT` | 409 | Overlaps an existing approved/active booking |
| `ASSET_UNAVAILABLE` | 409 | Asset is under maintenance |
| `INVALID_TRANSITION` | 409 | Illegal booking state-machine jump |
| `UNSUPPORTED_FILE_TYPE` | 415 | Upload MIME not in the allow-list |
| `PAYLOAD_TOO_LARGE` | 413 | Upload > 5 MB |
| `RATE_LIMITED` | 429 | Too many requests; includes `Retry-After` header |
| `INTERNAL_ERROR` | 500 | Unexpected failure — details never exposed |

### 1.6 Pagination
Query params on all list endpoints:

| Param | Default | Notes |
| --- | --- | --- |
| `page` | `1` | 1-based |
| `limit` | `20` | Max `100` |

```json
"pagination": { "page": 1, "limit": 20, "total": 50, "totalPages": 3 }
```

### 1.7 Filtering & sorting
- Query params are allow-listed; unknown values are **stripped, not rejected** (never 500).
- Dates use `?from=2026-10-01&to=2026-10-31` (inclusive) unless stated otherwise.

### 1.8 Rate limits

| Route | Limit |
| --- | --- |
| `/api/auth/login`, `/api/auth/signup` | 10 / 15 min / IP |
| `POST /api/bookings` | 20 / hour / user |
| `POST /api/issues` | 10 / day / user |
| Everything else | 300 / 15 min / IP |

---

## 2. Endpoint Summary

### Auth — `/api/auth`
| Method | Path | Role | Purpose |
| --- | --- | --- | --- |
| POST | `/auth/signup` | public | Create account (role always `user`) |
| POST | `/auth/login` | public | Issue JWT |
| POST | `/auth/refresh` | user | Rotate token |
| POST | `/auth/logout` | user | Client-side token clear (stateless, always 204) |
| GET | `/auth/me` | user | Current profile |
| PATCH | `/auth/me` | user | Update own name/department |
| POST | `/auth/change-password` | user | Change password with old-password check |

### Catalog — `/api/assets`, `/api/categories`
| Method | Path | Role | Purpose |
| --- | --- | --- | --- |
| GET | `/categories` | public | Category list for filters |
| GET | `/assets` | user | Catalog with search/filter/pagination |
| GET | `/assets/:id` | user | Asset detail + next upcoming bookings |
| GET | `/assets/:id/availability` | user | Busy windows for a date range |
| GET | `/assets/:id/bookings.csv` | admin | **Export** booking history (CSV) |
| POST | `/assets` | admin | Create asset (multipart, optional image) |
| PATCH | `/assets/:id` | admin | Edit fields / set `status` |
| DELETE | `/assets/:id` | admin | Soft-delete (archived) |

### Bookings — `/api/bookings`
| Method | Path | Role | Purpose |
| --- | --- | --- | --- |
| POST | `/bookings` | user | Create booking (overlap-checked) |
| GET | `/bookings/mine` | user | My bookings, filterable by status |
| GET | `/bookings/:id` | user | Booking detail (owner or admin) |
| POST | `/bookings/:id/cancel` | user | Cancel own pending/approved booking |
| POST | `/bookings/:id/start` | user/admin | `approved → active` |
| POST | `/bookings/:id/complete` | user/admin | `active → completed` |
| POST | `/bookings/:id/transition` | user/admin | Generic state-machine call |

### Admin — `/api/admin`
| Method | Path | Role | Purpose |
| --- | --- | --- | --- |
| GET | `/admin/metrics` | admin | Dashboard counters + utilization |
| GET | `/admin/requests` | admin | Pending request queue |
| POST | `/admin/requests/:id/approve` | admin | Approve (re-runs conflict check) |
| POST | `/admin/requests/:id/reject` | admin | Reject with reason |
| GET | `/admin/users` | admin | User list |
| POST | `/admin/users/:id/role` | admin | Promote / demote |

### Issues — `/api/issues`
| Method | Path | Role | Purpose |
| --- | --- | --- | --- |
| POST | `/issues` | user | Report a problem with an asset |
| GET | `/admin/issues` | admin | Issue queue |
| PATCH | `/admin/issues/:id` | admin | Resolve / escalate |

### Notifications — `/api/notifications`
| Method | Path | Role | Purpose |
| --- | --- | --- | --- |
| GET | `/notifications` | user | Own notifications |
| PATCH | `/notifications/:id/read` | user | Mark one read |
| POST | `/notifications/read-all` | user | Mark all read |
| GET | `/notifications/unread-count` | user | Badge counter |

---

## 3. Auth

### `POST /api/auth/signup` · public
```json
// Request
{
  "email": "user@college.edu",
  "password": "Str0ngPass!",
  "full_name": "Aarav Sharma",
  "department": "CSE"
}
```
```json
// 201 Created
{
  "token": "eyJhbGciOi…",
  "expires_in": 604800,
  "user": {
    "id": "uuid",
    "email": "user@college.edu",
    "full_name": "Aarav Sharma",
    "department": "CSE",
    "role": "user",
    "penalty_points": 0,
    "created_at": "2026-10-02T09:00:00Z"
  }
}
```
Validation: `email` valid + unique, `password` ≥ 8 chars with one letter and one digit,
`full_name` 2–120 chars. **Any `role` field in the body is stripped** — a signup can never
create an admin (`TEST_PLAN` A-2).
Errors: `422 VALIDATION_ERROR`, `409 EMAIL_TAKEN`.

### `POST /api/auth/login` · public
```json
{ "email": "user@college.edu", "password": "Str0ngPass!" }
```
→ `200` same body shape as signup. Errors: `401 INVALID_CREDENTIALS`,
`429 RATE_LIMITED`.

### `POST /api/auth/refresh` · user
→ `200 { "token": "…", "expires_in": 604800, "user": { … } }`. Previous token invalidated.

### `GET /api/auth/me` · user
→ `200 { "id": "…", "email": "…", "full_name": "…", "department": "…", "role": "user",
"penalty_points": 0, "created_at": "…" }`. Never returns `password_hash`.

### `POST /api/auth/change-password` · user
```json
{ "current_password": "Str0ngPass!", "new_password": "Even5tronger!" }
```
→ `204`. Errors: `422`, `401 INVALID_CREDENTIALS` if `current_password` is wrong.

---

## 4. Catalog

### `GET /api/categories` · public
```json
{
  "data": [
    { "id": 1, "name": "Meeting Room", "icon": "door-open", "asset_count": 12 },
    { "id": 2, "name": "AV Equipment", "icon": "projector", "asset_count": 18 }
  ]
}
```

### `GET /api/assets` · user
Query params:

| Param | Type | Example | Notes |
| --- | --- | --- | --- |
| `q` | string | `projector` | Searches name + description, case-insensitive |
| `category` | int | `2` | Category id |
| `status` | enum | `available` | `available` \| `booked` \| `maintenance` |
| `location` | string | `Block A` | Partial match |
| `sort` | enum | `name_asc` | `name_asc` \| `name_desc` \| `newest` \| `utilization_desc` |
| `page`, `limit` | int | `1`, `20` | See §1.6 |

```json
// 200 OK
{
  "data": [
    {
      "id": "uuid",
      "name": "Board Room A",
      "description": "12-seat room with 4K display and video conferencing.",
      "category": { "id": 1, "name": "Meeting Room", "icon": "door-open" },
      "location": "Block A · 3rd floor",
      "image_url": "https://…supabase.co/storage/v1/object/public/assets/board-room-a.png",
      "capacity": 12,
      "status": "available",
      "requires_approval": true,
      "tags": ["4k-display", "video-conf"],
      "created_at": "2026-10-01T10:00:00Z"
    }
  ],
  "pagination": { "page": 1, "limit": 20, "total": 50, "totalPages": 3 }
}
```
Errors: `401 UNAUTHORIZED`. Invalid enum values are stripped (default filter applied).

### `GET /api/assets/:id` · user
Same object as above plus:
```json
"upcoming_bookings": [
  { "id": "uuid", "start_time": "2026-10-05T09:00:00Z", "end_time": "2026-10-05T10:30:00Z",
    "status": "approved", "user": { "id": "uuid", "full_name": "Aarav Sharma" } }
]
```
Errors: `404 NOT_FOUND`.

### `GET /api/assets/:id/availability` · user
Query: `from` (ISO date, required), `to` (ISO date, required). Max range 31 days.

```json
// 200 OK
{
  "asset_id": "uuid",
  "from": "2026-10-01T00:00:00Z",
  "to": "2026-10-31T23:59:59Z",
  "busy": [
    { "booking_id": "uuid", "start_time": "2026-10-05T09:00:00Z",
      "end_time": "2026-10-05T10:30:00Z", "status": "approved" }
  ]
}
```
Only `approved` and `active` windows are returned — pending requests do not block, so the
calendar does not show phantom conflicts. Errors: `422` (missing/bad range, range > 31 days),
`404`.

### `GET /api/assets/:id/bookings.csv` · admin
Bonus export. Returns `text/csv` with `Content-Disposition: attachment`.
Query: optional `from`, `to`, `status`.

```csv
booking_id,asset,requester,email,start_time,end_time,status,approved_by,is_overdue
b1f2,Board Room A,Aarav Sharma,user@college.edu,2026-10-05T09:00:00Z,2026-10-05T10:30:00Z,completed,Admin,false
```
Errors: `403 FORBIDDEN` (standard user — this is the graded "admin only" export),
`404`.

### `POST /api/assets` · admin
`multipart/form-data`. Image optional; if absent the asset is created without one.

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `name` | string | ✔ | ≤ 120 chars |
| `description` | string | | ≤ 2000 chars |
| `category_id` | int | ✔ | |
| `location` | string | ✔ | ≤ 160 chars |
| `capacity` | int | | ≥ 1 |
| `image` | file | | `image/jpeg` \| `image/png` \| `image/webp`, ≤ 5 MB |
| `tags` | csv | | e.g. `4k-display,video-conf` |

```json
// 201 Created — same asset object as GET /assets
```
Errors: `403`, `422`, `413 PAYLOAD_TOO_LARGE`, `415 UNSUPPORTED_FILE_TYPE`.
Uploads go to Multer's temp dir, then to Supabase Storage; the response `image_url` is the
public object URL.

### `PATCH /api/assets/:id` · admin
Partial update — same fields as create. Setting `status: "maintenance"` immediately blocks
new bookings **and** is reflected in the catalog.

```json
{ "status": "maintenance" }
```
Errors: `403`, `404`, `422` (invalid enum).

### `DELETE /api/assets/:id` · admin
Soft delete (`archived_at` set). Historical bookings are preserved. → `204`.

---

## 5. Bookings *(the critical part)*

### `POST /api/bookings` · user
```json
{
  "asset_id": "uuid",
  "start_time": "2026-10-05T14:00:00Z",
  "end_time": "2026-10-05T15:00:00Z",
  "purpose": "Client demo with the design team"
}
```
```json
// 201 Created
{
  "id": "uuid",
  "asset": { "id": "uuid", "name": "Board Room A", "location": "Block A · 3rd floor" },
  "user": { "id": "uuid", "full_name": "Aarav Sharma" },
  "start_time": "2026-10-05T14:00:00Z",
  "end_time": "2026-10-05T15:00:00Z",
  "status": "pending",
  "purpose": "Client demo with the design team",
  "is_overdue": false,
  "created_at": "2026-10-02T12:00:00Z"
}
```

**Conflict response** — the single most important error in the API:
```json
// 409 Conflict
{
  "error": {
    "code": "BOOKING_CONFLICT",
    "message": "Board Room A is already booked for part of that window.",
    "details": {
      "conflict": {
        "booking_id": "uuid",
        "start_time": "2026-10-05T14:30:00Z",
        "end_time": "2026-10-05T16:00:00Z",
        "status": "approved"
      }
    },
    "requestId": "b3f1c0de-…"
  }
}
```

**Rules** (mirrors `TEST_PLAN` §7):
- Overlap predicate: `existing.start_time < requested.end_time AND existing.end_time > requested.start_time` (half-open `[start, end)`, so abutting windows are fine).
- Only `approved` and `active` bookings block. **Pending does not block** — multiple users may be pending for the same slot; the conflict is resolved at approval.
- `end_time > start_time`; `end_time <= start_time + 14 days`; window not in the past.
- Asset must exist and **not** be `maintenance` (`409 ASSET_UNAVAILABLE`).
- If `asset.requires_approval === false`, the booking is created directly as `approved` and the conflict check still applies.
- Server time is authoritative; client clocks are never compared.

Errors: `401`, `403`, `404`, `409 BOOKING_CONFLICT`, `409 ASSET_UNAVAILABLE`,
`422 VALIDATION_ERROR`, `429 RATE_LIMITED`.

### `GET /api/bookings/mine` · user
Query: `status` (optional, one or comma-separated), `from`, `to`, `page`, `limit`.
Default sort: newest first. Returns the same booking objects as above, with
`asset` and `approved_by` expanded. Only the caller's own bookings are ever returned.

### `GET /api/bookings/:id` · user
Owner or admin. `403` for anyone else (`TEST_PLAN` T-7 / S-9).

### `POST /api/bookings/:id/cancel` · user
```json
// Request (reason optional for the owner, required when an admin force-cancels an active booking)
{ "reason": "Meeting got moved online" }
```
`pending → cancelled` and `approved → cancelled` for the owner. An admin may also
force-cancel `active` bookings, which frees the asset and notifies the owner. → `200` with
the updated booking.

Errors: `403` (not the owner), `404`, `409 INVALID_TRANSITION` (already completed/rejected).

### `POST /api/bookings/:id/start` · user/admin
`approved → active`. Sets `started_at`. An admin or the owner of an approved booking that
has reached its `start_time` may start it. The cron job also does this automatically.

### `POST /api/bookings/:id/complete` · user/admin
`active → completed`. Sets `completed_at` and releases the asset.
Returns `200` with the updated booking.

### `POST /api/bookings/:id/transition` · user/admin
Generic form of the three action endpoints — useful for the admin queue's generic controls.
```json
{ "to": "approved" }        // or: rejected | cancelled | active | completed
{ "to": "rejected", "reason": "Room already reserved by the department" }
```
Rules: the target must be legal from the current status (`TEST_PLAN` §8), and role rules
still apply (only an admin can `approve`/`reject`). Illegal jumps →
`409 INVALID_TRANSITION`, with `details.allowed` listing the legal next states.

**State machine** (server-authoritative):
```
PENDING ──approve──▶ APPROVED ──start──▶ ACTIVE ──complete──▶ COMPLETED
   │                     │                  │
   ├──reject──▶ REJECTED └──cancel──▶ CANCELLED ◀──admin force── ACTIVE
   └──cancel──▶ CANCELLED
```

| From | Allowed `to` | Who |
| --- | --- | --- |
| `pending` | `approved`, `rejected`, `cancelled` | admin / owner(cancel only) |
| `approved` | `active`, `cancelled` | admin or system |
| `active` | `completed`, `cancelled` | admin or owner |
| `completed`, `rejected`, `cancelled` | — terminal | — |

---

## 6. Admin

### `GET /api/admin/metrics` · admin
```json
{
  "total_assets": 52,
  "available_assets": 41,
  "assets_under_maintenance": 3,
  "pending_requests": 6,
  "active_bookings": 9,
  "overdue_bookings": 2,
  "bookings_today": 14,
  "total_bookings": 128,
  "open_issues": 4,
  "utilization": [
    { "asset_id": "uuid", "asset_name": "Board Room A", "bookings_30d": 21, "hours_30d": 42.5,
      "utilization_pct": 8.8 }
  ],
  "utilization_by_category": [
    { "category": "Meeting Room", "bookings_30d": 48 }
  ]
}
```
Numbers are computed with direct SQL aggregates, so they always match the underlying rows.

### `GET /api/admin/requests` · admin
Pending queue. Query: `page`, `limit`, `sort=oldest_first` (default — FIFO).
Each row is enriched for the queue UI:
```json
{
  "data": [
    {
      "id": "uuid", "status": "pending",
      "start_time": "2026-10-05T14:00:00Z", "end_time": "2026-10-05T15:00:00Z",
      "purpose": "Client demo with the design team",
      "user": { "id": "uuid", "full_name": "Aarav Sharma", "email": "user@college.edu",
                "department": "CSE", "penalty_points": 0 },
      "asset": { "id": "uuid", "name": "Board Room A", "category": { "id": 1, "name": "Meeting Room" } },
      "conflict_warning": {
        "has_pending_overlap": 2,
        "note": "Two other users are also pending for this window."
      },
      "created_at": "2026-10-02T12:00:00Z"
    }
  ],
  "pagination": { "page": 1, "limit": 20, "total": 6, "totalPages": 1 }
}
```

### `POST /api/admin/requests/:id/approve` · admin
No body required. → `200` with the updated booking (`status: "approved"`,
`approved_by`, `decided_at`). Side effects, in order: row lock → re-check conflicts inside
the lock → update → insert `notifications` row → realtime broadcast → Nodemailer email.

If the window became unavailable while the request sat pending:
`409 BOOKING_CONFLICT` (with `details.conflict` naming the blocking booking). The request
stays `pending`.

### `POST /api/admin/requests/:id/reject` · admin
```json
{ "reason": "Room is reserved for the annual review that day" }
```
`reason` is **required** (1–500 chars). → `200` with `status: "rejected"`. Notifies the
user in-app + email with the reason.

### `GET /api/admin/users` · admin
Query: `q`, `role`, `page`, `limit`. Returns profiles without any password material.
```json
{ "data": [ { "id": "uuid", "full_name": "Aarav Sharma", "email": "user@college.edu",
              "role": "user", "penalty_points": 1, "created_at": "…" } ],
  "pagination": { "page": 1, "limit": 20, "total": 21, "totalPages": 2 } }
```

### `POST /api/admin/users/:id/role` · admin
```json
{ "role": "admin" }   // or "user" to demote
```
→ `200` with the updated profile. A user cannot change their own role (prevents accidental
lockout); `403` in that case.

---

## 7. Issues

### `POST /api/issues` · user
```json
{
  "asset_id": "uuid",
  "booking_id": "uuid",        // optional
  "description": "Projector bulb flickering on the left lens.",
  "severity": "medium"         // low | medium | high | critical
}
```
→ `201` with the created issue (`resolved: false`).

### `GET /api/admin/issues` · admin
Query: `resolved` (`true`/`false`), `severity`, `page`, `limit`. Default: open issues first,
newest first. Each row expands `asset`, `reporter`, and `booking` when present.

### `PATCH /api/admin/issues/:id` · admin
```json
{ "resolved": true, "severity": "high" }
```
→ `200` with the updated issue. Notifies the reporter when resolved.

---

## 8. Notifications

### `GET /api/notifications` · user
Query: `unread_only`, `page`, `limit`.
```json
{
  "data": [
    {
      "id": "uuid",
      "type": "approved",              // approved | rejected | overdue | reminder | resolved
      "title": "Booking approved",
      "body": "Board Room A is reserved for you on 5 Oct, 2:00–3:00 PM.",
      "booking_id": "uuid",
      "read_at": null,
      "created_at": "2026-10-02T12:05:00Z"
    }
  ],
  "pagination": { "page": 1, "limit": 20, "total": 3, "totalPages": 1 }
}
```
Only the caller's notifications are ever returned. Triggers a client toast when the
notification arrives via realtime.

### `PATCH /api/notifications/:id/read` · user → `200` with the notification (`read_at` set).
### `POST /api/notifications/read-all` · user → `204`.
### `GET /api/notifications/unread-count` · user → `200 { "count": 2 }` (drives the sidebar bell badge).

---

## 9. Realtime Events (bonus)

Supabase Realtime, channel `booking_events`, scoped to the authenticated user's id.

```json
{ "type": "notification", "payload": { /* the notification object */ } }
{ "type": "booking_updated", "payload": { "id": "uuid", "status": "approved", "asset": { … } } }
{ "type": "asset_maintenance", "payload": { "asset_id": "uuid", "status": "maintenance" } }
```

The realtime channel is a **notification mechanism only** — the REST response remains the
source of truth. If the socket drops, the client polls `/notifications` every 60 s.

---

## 10. Health & Meta

| Method | Path | Role | Purpose |
| --- | --- | --- | --- |
| GET | `/api/health` | public | `{ "status": "ok", "uptime_s": 123, "db": "reachable" }` |
| GET | `/api/meta/statuses` | public | Enum lists for the UI (booking/asset statuses, transitions) |

`/api/meta/statuses` lets the frontend render badges and the state machine UI without
duplicating the enum lists in client code.

---

## 11. Frontend Integration Notes (`client/src/lib/api.js`)

A single fetch wrapper is the only place that touches `fetch`:

```js
export async function api(path, { method = 'GET', body, token, raw } = {}) {
  const res = await fetch(`${import.meta.env.VITE_API_URL}${path}`, {
    method,
    headers: {
      ...(body && !(body instanceof FormData) ? { 'Content-Type': 'application/json' } : {}),
      ...(token ? { Authorization: `Bearer ${token}` } : {}),
    },
    body: body instanceof FormData ? body : body ? JSON.stringify(body) : undefined,
  });
  if (raw) return res;                       // CSV export
  const payload = res.status === 204 ? null : await res.json();
  if (!res.ok) throw Object.assign(new Error(payload?.error?.message ?? 'Request failed'), {
    status: res.status, code: payload?.error?.code, details: payload?.error?.details,
  });
  return payload;
}
```

Client-side rules that mirror the server (never a substitute for it):
- Bookings use an optimistic Redux insert, rolled back on `409 BOOKING_CONFLICT` with a modal showing `details.conflict`.
- Route guards (`<RequireAdmin>`) read `auth.role`; the server still re-checks every admin call and returns `403`.
- Toasts render `error.message`; `error.code` decides the tone (`BOOKING_CONFLICT`/`ASSET_UNAVAILABLE` → warning, `FORBIDDEN`/`UNAUTHORIZED` → redirect to login).
- Availability is polled for the visible month only, and never cached beyond a 30 s window.

---

## 12. Change Log

| Date | Change | Author |
| --- | --- | --- |
| 2026-10-02 | Initial contract for the MVP (v1) | Team |

Keep this table updated — if a parameter is added or a status code changes, log it here in
the same commit as the code.