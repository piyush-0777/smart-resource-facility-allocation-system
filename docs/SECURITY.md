# Security — WorkFlow

## 1. Threat Model

| # | Threat | Impact | Mitigation |
| --- | --- | --- | --- |
| T-1 | Privilege escalation by editing the URL (`/admin` as a standard user) | Full admin access | Server-side `requireRole('admin')` on every admin route; role never read from the request body at signup |
| T-2 | Forged / tampered JWT | Impersonation | HS256 signature + `exp` verified server-side; secret never shipped to the client |
| T-3 | Double booking via concurrent requests | Data integrity | Postgres `EXCLUDE USING gist` constraint + `SELECT … FOR UPDATE` |
| T-4 | SQL injection | Data breach | Parameterized Supabase queries only; no string-built SQL anywhere |
| T-5 | Malicious upload (`.svg` with script, oversized file, wrong MIME) | Stored XSS / DoS | `multer` diskStorage, allow-list `image/jpeg|png|webp`, 5 MB cap, random filename, re-encoded through Supabase Storage; SVG never accepted |
| T-6 | Stored XSS via asset name / purpose text | Session theft | React escapes by default; no `dangerouslySetInnerHTML`; length caps + sanitizing strip on input |
| T-7 | IDOR on `/api/bookings/:id` | Read/cancel others' bookings | Ownership check in the service layer (`booking.user_id === req.user.id` unless admin) |
| T-8 | Secrets committed to git | Full compromise | `.env` gitignored; `.env.example` committed; `git secrets` pre-commit hook |
| T-9 | Abuse of booking creation | Denial of service | Rate limit auth + write routes; max 7-day window; per-user active-booking cap |
| T-10 | CSRF on state-changing requests | Forged approvals | JWT in the `Authorization` header (not cookies) + strict CORS allowlist |

Explicitly out of scope for the MVP: multi-tenant isolation, SSO, encryption at rest
(handled by the Supabase provider), audit-log tamper-proofing.

## 2. Authentication

- Passwords hashed with **bcrypt, cost factor 12**. Never logged, never returned by any
  endpoint (`select *` is forbidden on `profiles`; a fixed column list is used instead).
- Login responses are uniform: `401 { code: "INVALID_CREDENTIALS" }` whether the email
  exists or not — no user enumeration.
- JWT payload: `{ sub, role, iat, exp }`, 7-day expiry. Client stores it in
  `localStorage` (XSS exposure is accepted and mitigated by T-6 mitigations; an
  httpOnly refresh cookie is the documented next step).
- Logout clears client state; refresh rotation invalidates the previous token.
- Optional: 2FA is out of scope; rate limiting protects the login route instead.

## 3. Authorization (RBAC)

Single choke point, applied per route:

```js
router.get('/api/admin/requests', auth, requireRole('admin'), handler)
```

- `auth` — verifies the token, loads the profile, sets `req.user`.
- `requireRole(...)` — compares `req.user.role` against the allowed set; otherwise `403`.
- Client-side route guards (`<RequireAdmin>`) are **UX only** and never trusted.
- Data-level checks sit in services, not just routes (ownership, e.g. cancelling a booking).
- Default deny: a route without an explicit guard is a bug caught by an automated test that
  walks the route table and asserts every route declares a policy.

## 4. Input Validation

- Every request body / query / param passes a schema (zod or express-validator) in
  `validate.js` before a controller runs.
- Enumerated fields are allow-listed (`status`, `severity`), never trusted as free strings.
- `purpose` ≤ 500 chars, `description` ≤ 2000, `name` ≤ 120.
- Timestamps must parse to a valid ISO date; the server is the only clock authority
  (client times are never compared against the DB).
- Unknown fields are stripped (mass-assignment protection) so a user cannot post
  `{ "role": "admin" }` or `{ "user_id": "…" }`.

## 5. Overlap / Integrity Controls

1. **Pre-check** in `booking.service` → readable `409 BOOKING_CONFLICT` with the window.
2. **Row lock** (`SELECT … FOR UPDATE`) around approval so two admins can't approve
   overlapping windows concurrently.
3. **Database constraint** `EXCLUDE USING gist (asset_id WITH =, slot WITH &&) WHERE status IN ('approved','active')` — the final, non-bypassable arbiter.
4. `CHECK (end_time > start_time)` and `CHECK (end_time <= start_time + interval '14 days')`.
5. All booking writes are recorded in `audit_log` with actor, entity, and metadata.

## 6. File Uploads (multer)

```js
const ALLOWED = new Set(['image/jpeg', 'image/png', 'image/webp']);
multer({
  storage: multer.diskStorage({ destination: UPLOAD_DIR, filename: cryptoRandom }),
  limits: { fileSize: 5 * 1024 * 1024, files: 1 },
  fileFilter: (req, file, cb) =>
    ALLOWED.has(file.mimetype) ? cb(null, true) : cb(new Error('UNSUPPORTED_FILE_TYPE')),
})
```

- Filenames are server-generated (UUID + extension); the client's filename is discarded.
- Stored outside the web root; served only through an authenticated endpoint or a
  signed Supabase Storage URL.
- SVG and HTML are rejected (script-capable formats).
- Uploads dir is gitignored and cleared on deploy.

## 7. Transport, CORS & Headers

- HTTPS enforced in production (proxy terminates TLS; `X-Forwarded-Proto` honoured).
- `helmet` sets CSP (`default-src 'self'`, `img-src 'self' data: https://*.supabase.co`),
  `X-Content-Type-Options: nosniff`, `Referrer-Policy: no-referrer`.
- CORS: `origin: process.env.CLIENT_ORIGIN` only, `credentials: false`. No wildcard.
- Cookies, if ever introduced: `httpOnly`, `secure`, `sameSite: 'strict'`.

## 8. Rate Limiting

| Route | Limit |
| --- | --- |
| `/api/auth/login`, `/signup` | 10 / 15 min / IP |
| `/api/bookings` (POST) | 20 / hour / user |
| `/api/issues` (POST) | 10 / day / user |
| Global | 300 / 15 min / IP |

Returns `429` with `Retry-After`. Rate limiter store is in-memory (single instance) —
Redis would be the production note.

## 9. Secrets & Environment

- `server/.env` and `client/.env.local` are gitignored; `.env.example` documents keys
  with empty values.
- `JWT_SECRET` ≥ 32 random chars, generated per environment; never committed, never
  reused across staging/production.
- Supabase **service role key** is server-only (all DB access is server-side, so no RLS
  policy is required for the MVP — noted as the migration path if a client ever talks
  to Supabase directly). The client gets only the anon key, used for Realtime and image
  URLs.
- Nodemailer credentials are environment-only; emails log to console in development.

## 10. Error & Logging Hygiene

- Central `error.js` handler: unexpected errors become a generic `500` with a request id;
  stack traces never reach the client.
- `morgan` request logging plus structured app logs; JWTs, passwords, tokens, and email
  bodies are filtered out of logs.
- Validation errors return field-level messages only — no database internals.

## 11. Verification Checklist

- [ ] Standard user calling any `/api/admin/*` gets `403` (automated test).
- [ ] Signing up with `{ "role": "admin" }` yields a `user` (automated test).
- [ ] Invalid / expired / tampered JWT → `401`.
- [ ] Two concurrent approvals for the same slot → exactly one `200`, one `409`.
- [ ] Booking `end_time <= start_time` → `422`.
- [ ] Uploading `text/html` or a 6 MB file → `415`/`413`.
- [ ] `?status=admin'` style enum injection → stripped, not executed.
- [ ] No secret appears in the built client bundle (`grep` the dist output).