# Architecture — WorkFlow

## 1. Overview

Three-tier single-repo application:

```
┌───────────────────────────────────────────────────────────────┐
│  client/  (React 18 + Vite + Redux Toolkit + React Router)     │
│  Tailwind CSS glass UI · lucide-react icons · light/dark       │
└───────────────┬───────────────────────────────┬───────────────┘
                │ REST (fetch + JWT)            │ Realtime (SSE/WS)
┌───────────────▼───────────────────────────────▼───────────────┐
│  server/  (Node.js + Express)                                 │
│  routes → controllers → services → repositories                │
│  jsonwebtoken · bcrypt · multer · nodemailer · node-cron       │
└───────────────────────────────┬───────────────────────────────┘
                                │  @supabase/supabase-js (service key)
┌───────────────────────────────▼───────────────────────────────┐
│  Supabase — Postgres (source of truth) · Storage (images)     │
│  Realtime (booking events) · Auth-compatible email via SMTP    │
└───────────────────────────────────────────────────────────────┘
```

- `client/` — SPA, port `5173`
- `server/` — API, port `4000`
- `docs/` — these documents

The API is the only writer. The client never talks to Supabase directly except for
realtime subscriptions and image CDN URLs (RLS applies to storage objects).

## 2. Tech Stack & Reasoning

| Layer | Choice | Why |
| --- | --- | --- |
| Frontend | React 18 + Vite | Instant HMR matters in an 8-hour build; Vite dev server starts in ~200 ms vs webpack's 20 s |
| State | Redux Toolkit | Booking flow and admin queue are shared across routes; RTK's `createSlice` + RTK Query-style reducers keep booking/notification state predictable without prop drilling |
| Routing | React Router v6 | Declarative nested routes map cleanly to RBAC route guards (`<RequireAdmin>`) |
| Styling | Tailwind CSS | Design-system consistency with no hand-written CSS; glass utility classes composed from theme tokens |
| Icons | lucide-react | Consistent stroke weight, tree-shakeable, matches the light aesthetic |
| Backend | Node.js + Express | Same language as the client → shared validation schemas and types, no context switch |
| Database | **Supabase (Postgres)** | Real SQL database (brief explicitly bans static JSON), plus free Storage, Realtime, and dashboard auth |
| Auth | jsonwebtoken + bcrypt | Stateless JWT fits the "cannot reach admin URL" requirement without server sessions |
| Email | nodemailer | SMTP (Gmail app password or Ethereal in dev); approval/rejection/overdue notifications |
| Uploads | multer | Multipart asset photos → Supabase Storage |
| Scheduling | node-cron | Marks overdue active bookings, auto-completes finished windows |
| Tests | Vitest + Supertest | Same runner as Vite; Supertest exercises real HTTP + real SQL overlap rules |

**Why Postgres/Supabase specifically:** the hard requirement — *reject overlapping approved
bookings* — is a database problem, not an application problem. Postgres gives us
`EXCLUDE USING gist` range constraints that make double-booking **physically impossible**
even under concurrent requests, which a "SELECT then INSERT" check in JavaScript cannot
guarantee.

## 3. Repository Layout

```
.
├── client/
│   ├── public/
│   ├── src/
│   │   ├── app/            # store.js, router.jsx, provider
│   │   ├── components/
│   │   │   ├── glass/      # GlassPanel, GlassButton, GlassInput, GlassModal
│   │   │   ├── layout/     # AppShell, Sidebar, Topbar, ThemeToggle
│   │   │   └── ui/         # Badge, Skeleton, EmptyState, Toast
│   │   ├── features/
│   │   │   ├── auth/       # authSlice, LoginPage, RegisterPage, guards
│   │   │   ├── catalog/    # AssetGrid, AssetCard, AssetDetail, filters
│   │   │   ├── bookings/   # BookingForm, BookingWizard, MyBookings
│   │   │   ├── admin/      # DashboardPage, RequestsQueue, AssetManager, Metrics
│   │   │   └── realtime/   # notification listener
│   │   ├── lib/            # api.js (fetch wrapper), theme.js, csv.js
│   │   ├── styles/         # tailwind.css with glass tokens, .dark variant
│   │   └── store/          # bookingSlice, uiSlice, notificationSlice
│   └── vite.config.js
├── server/
│   ├── src/
│   │   ├── index.js        # bootstrap + cron
│   │   ├── app.js          # express app (exported for tests)
│   │   ├── config/         # env.js, supabase.js, mailer.js, cron.js
│   │   ├── middleware/     # auth.js, rbac.js, error.js, validate.js, upload.js
│   │   ├── modules/        # auth | assets | bookings | admin | issues
│   │   ├── services/       # booking.service.js, overlap.js, notifications.js
│   │   └── jobs/           # overdue.job.js
│   ├── uploads/            # multer temp dir
│   └── tests/
└── docs/
```

Module-per-feature inside both apps; `routes → controllers → services → repositories`.
Services own all business rules (state machine, overlap) so controllers stay thin and the
rules are unit-testable without HTTP.

## 4. Data Model (Postgres / Supabase)

```sql
-- ENUMs
create type user_role       as enum ('user', 'admin');
create type asset_status    as enum ('available', 'booked', 'maintenance');
create type booking_status  as enum ('pending','approved','active',
                                     'completed','rejected','cancelled');
create type issue_severity  as enum ('low','medium','high','critical');

create table profiles (            -- mirrors auth.users
  id          uuid primary key references auth.users on delete cascade,
  full_name   text not null,
  role        user_role not null default 'user',
  department  text,
  penalty_points int not null default 0,
  created_at  timestamptz not null default now()
);

create table categories (
  id   serial primary key,
  name text unique not null,
  icon text not null default 'box'
);

create table assets (
  id            uuid primary key default gen_random_uuid(),
  name          text not null,
  description   text,
  category_id   int references categories(id),
  location      text not null,
  image_url     text,
  capacity      int,
  status        asset_status not null default 'available',
  requires_approval boolean not null default true,
  created_by    uuid references profiles(id),
  created_at    timestamptz not null default now()
);

create table bookings (
  id            uuid primary key default gen_random_uuid(),
  asset_id      uuid not null references assets(id) on delete cascade,
  user_id       uuid not null references profiles(id) on delete cascade,
  start_time    timestamptz not null,
  end_time      timestamptz not null,
  status        booking_status not null default 'pending',
  purpose       text not null,
  approved_by   uuid references profiles(id),
  decided_at    timestamptz,
  reject_reason text,
  started_at    timestamptz,
  completed_at  timestamptz,
  is_overdue    boolean not null default false,
  created_at    timestamptz not null default now(),
  constraint valid_window check (end_time > start_time),
  constraint max_duration  check (end_time <= start_time + interval '14 days')
);

-- ★ The double-booking guarantee
create extension if not exists btree_gist;
alter table bookings add column slot tstzrange
  generated always as (tstzrange(start_time, end_time, '[)')) stored;

alter table bookings add constraint no_overlapping_approved_bookings
  exclude using gist (asset_id with =, slot with &&)
  where (status in ('approved','active'));

create table issues (
  id          uuid primary key default gen_random_uuid(),
  asset_id    uuid not null references assets(id) on delete cascade,
  booking_id  uuid references bookings(id) on delete set null,
  user_id     uuid not null references profiles(id),
  description text not null,
  severity    issue_severity not null default 'medium',
  resolved    boolean not null default false,
  created_at  timestamptz not null default now()
);

create table notifications (
  id         uuid primary key default gen_random_uuid(),
  user_id    uuid not null references profiles(id) on delete cascade,
  type       text not null,      -- approved | rejected | overdue | reminder
  title      text not null,
  body       text,
  booking_id uuid references bookings(id) on delete cascade,
  read_at    timestamptz,
  created_at timestamptz not null default now()
);

create table audit_log (          -- accountability trail
  id          uuid primary key default gen_random_uuid(),
  actor_id    uuid references profiles(id),
  action      text not null,
  entity      text not null,
  entity_id   uuid,
  meta        jsonb,
  created_at  timestamptz not null default now()
);
```

Key indexes: `bookings(asset_id, start_time)`, `bookings(user_id, status)`,
`bookings(status) where is_overdue`, `assets(category_id)`, GIN on `audit_log.meta`.

## 5. Data Flow — Creating a Conflict-Free Booking

```
 User (React form)
   │  POST /api/bookings  { asset_id, start_time, end_time, purpose }
   │  + Authorization: Bearer <jwt>
   ▼
 Express: validate.js (shape) → auth.js (verify JWT, load profile)
   ▼
 auth.js → rbac.js (role === 'user' | 'admin')
   ▼
 booking.controller → booking.service.createBooking()
   │
   ├─ 1. Guard: end_time > start_time, within 14 days, not in the past
   ├─ 2. Guard: asset exists AND status <> 'maintenance'
   ├─ 3. PRE-CHECK (readable error):
   │      select * from bookings
   │       where asset_id = $1
   │         and status in ('approved','active')
   │         and start_time < $3 and end_time > $2
   │      → if any row: 409 CONFLICT + conflicting window
   ├─ 4. INSERT status='pending'
   │      (exclusion constraint still applies as safety net — but pending
   │       doesn't overlap-block, so two users can both be pending)
   ▼
 Supabase Postgres
   │  (later, admin approves)
   ▼
 booking.service.transition(id, 'approved')
   ├─ reload row with SELECT ... FOR UPDATE  (row lock)
   ├─ assert legal transition from state machine map
   ├─ re-check conflicts INSIDE the lock
   ├─ UPDATE status='approved'  → exclusion constraint is the final arbiter;
   │                             violation ⇒ 409 with the blocking booking id
   ├─ insert notification + audit_log
   └─ notifications.send() → nodemailer email + realtime broadcast
   ▼
 Client: realtime subscription fires → toast + badge refresh (RTK slice)
```

ASCII state machine used by `booking.service`:

```
PENDING ──approve──▶ APPROVED ──start──▶ ACTIVE ──complete──▶ COMPLETED
   │                     │                  │
   ├──reject──▶ REJECTED └──cancel──▶ CANCELLED ◀──admin force── ACTIVE
   └──cancel──▶ CANCELLED
```

## 6. API Surface

Full request/response contract lives in **`docs/API.md`** — the table below is the summary.
All routes under `/api`. Protected routes require `Authorization: Bearer <token>`.

| Method | Path | Role | Purpose |
| --- | --- | --- | --- |
| POST | `/api/auth/signup` | public | Create account (role always `user`) |
| POST | `/api/auth/login` | public | Issue JWT (7 d) |
| POST | `/api/auth/refresh` | user | Rotate token |
| GET | `/api/auth/me` | user | Current profile |
| POST | `/api/auth/change-role` | admin | Promote/demote |
| GET | `/api/categories` | public | Category list for filters |
| GET | `/api/assets` | user | Catalog + `?q&category&status&page` |
| GET | `/api/assets/:id` | user | Detail + upcoming bookings |
| GET | `/api/assets/:id/availability` | user | Busy windows for a date range |
| POST | `/api/assets` | admin | Create asset (multer `image`) |
| PATCH | `/api/assets/:id` | admin | Edit / set `status`, incl. `maintenance` |
| GET | `/api/assets/:id/bookings.csv` | admin | Data export |
| POST | `/api/bookings` | user | Create (overlap-checked) |
| GET | `/api/bookings/mine` | user | My bookings |
| POST | `/api/bookings/:id/cancel` | user | Cancel own pending/approved |
| POST | `/api/bookings/:id/transition` | user/admin | State machine (role-scoped) |
| GET | `/api/admin/metrics` | admin | Dashboard counters + utilization |
| GET | `/api/admin/requests` | admin | Pending queue |
| POST | `/api/admin/requests/:id/approve` | admin | Approve |
| POST | `/api/admin/requests/:id/reject` | admin | Reject with reason |
| POST | `/api/issues` | user | Report asset issue |
| GET | `/api/admin/issues` | admin | Issue queue |
| GET | `/api/notifications` | user | Notification history |
| POST | `/api/notifications/:id/read` | user | Mark read |

Errors are uniform:
```json
{ "error": { "code": "BOOKING_CONFLICT", "message": "Asset already booked 10:00–11:30",
  "conflict": { "start_time": "...", "end_time": "..." } } }
```

## 7. RBAC Enforcement

```
request → auth.js (verify JWT) → rbac('admin') → handler
                  │
                  └── 401 no/invalid token
                                    403 valid token, insufficient role
```

- `auth.js` — verifies signature + expiry, loads profile, attaches `req.user`.
- `rbac.js` — factory `requireRole('admin')`; compares `req.user.role`.
- React Router mirrors this with `<RequireAuth>` / `<RequireAdmin>` for UX only.
- Manual URL tampering (`/admin` as a user) hits the server guard → `403`, plus the
  client guard bounces to `/catalog`. Both layers tested.

## 8. Realtime & Notifications

- Supabase Realtime channel `booking_events`, filtered by `user_id`.
- `booking.service` inserts into `notifications` then broadcasts.
- Client middleware (Redux listener) subscribes when a user is logged in and dispatches
  `notificationReceived` → toast renders with lucide icon, badge count increments.
- Fallback: client polls `/api/notifications` every 60 s if the socket drops.

## 9. Background Jobs (`node-cron`)

| Schedule | Job |
| --- | --- |
| every 1 min | auto-start approved bookings whose `start_time <= now()` |
| every 5 min | flag overdue: `status='active' AND end_time < now() - grace` → `is_overdue=true`, notify user, `penalty_points += 1` |
| hourly | maintenance expiry reminder emails |

Jobs are idempotent and wrapped in try/catch so one failure never kills the process.

## 10. Environment Variables (`server/.env`)

```
PORT=4000
NODE_ENV=development
SUPABASE_URL=
SUPABASE_SERVICE_ROLE_KEY=
JWT_SECRET=
JWT_EXPIRES_IN=7d
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=
SMTP_PASS=
MAIL_FROM="WorkFlow <no-reply@workflow.app>"
CLIENT_ORIGIN=http://localhost:5173
GRACE_MINUTES=30
UPLOAD_DIR=uploads
```

Client (`.env`): `VITE_API_URL=http://localhost:4000/api`,
`VITE_SUPABASE_URL`, `VITE_SUPABASE_ANON_KEY` (realtime only).

## 11. Deployment Shape

- **Client**: Vite build → static host (Vercel/Netlify) with SPA rewrite to `index.html`.
- **Server**: Node process on Render/Railway with `npm start`; env vars from dashboard.
- **Database**: Supabase cloud project; `supabase/migrations/001_init.sql` applied via CLI.
- CORS restricted to `CLIENT_ORIGIN`.

## 12. Performance & Scale Notes

- Index-backed catalog queries; cursor pagination beyond page 1.
- Availability fetched as a single range query per asset per month view.
- Realtime is fire-and-forget — the REST response is always the source of truth.
- At hackathon scale (< 10k bookings) the exclusion constraint + composite index is more
  than sufficient; no caching layer or read replica needed.