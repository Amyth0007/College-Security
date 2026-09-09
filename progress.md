# VMS — Progress / Context Snapshot

Last updated: 2026-09-01

This file is the working snapshot for the next round of changes. Full API and milestone history lives in `HANDBOOK.md`. Use this file to see **where the app is today**.

---

## What this project is

College **Visitor Management System** (MERN). Guards (or visitors themselves) register a visit and get a QR pass. Staff scan that QR to check in and check out. Admins see logs, stats, and manage guard accounts.

Two roles: `admin` and `guard`.

---

## Stack (as implemented)

| Layer | Tech |
|-------|------|
| Backend | Node.js, Express 5, MongoDB / Mongoose 9 |
| Auth | JWT + bcryptjs, role middleware |
| QR | `crypto.randomBytes(32)` token + `qrcode` PNG data URL |
| Frontend | React 19, Vite 8, React Router 7, Axios |
| Scanner | `html5-qrcode` |
| Deploy notes | Frontend on Netlify (`frontend/public/_redirects` SPA rule). Backend health ping in `api.js` to wake a Render free-tier API. Local API is `http://localhost:5000/api`. |

---

## Run locally

```bash
# backend
cd backend && npm run dev
# optional: npm run seed   → default admin from .env

# frontend
cd frontend && npm run dev
```

- API: `http://localhost:5000`
- Health: `GET /api/health`
- App root `/` redirects to public register `/register`
- Staff login: `/login`

---

## Domain model

Four collections:

1. **User** — name, email, hashed password, `admin` | `guard`, `isActive`
2. **Visitor** — person identity: name, mobile, type (`parent` | `vendor` | `official` | `student` | `other`), optional photo. Reused by **mobile** on later visits.
3. **Visit** — one campus visit: visitor ref, optional `registeredBy`, optional `pass`, status, QR, purpose/host, check-in/out times. **Each pass use is a visit row** — history is never overwritten.
4. **Pass** — reusable physical QR card: `QR-01`, secret `qrToken` (`PASS:…`), status `available` | `in_use` | `disabled`, `currentVisit`, `useCount`.

**Visit lifecycle in practice (3 steps):**

`registered` → `checked_in` → `completed`

Check-out sets status to `completed`, writes `checkOutTime`, and if a pass was assigned it becomes `available` again.

Phone QR = one-time visit token. Physical pass QR always encodes the **pass** token; scan looks up the pass, then its **current** visit.

**Visit lifecycle in practice (3 steps):**

`registered` → `checked_in` → `completed`

Check-out sets status to `completed` and writes `checkOutTime` in one step.

**Quirk:** `backend/src/constants/index.js` still defines a 4-step map including unused `checked_out`. Runtime code does **not** use that extra step.

---

## Backend map

```
backend/src/
  server.js                 Express app, CORS, helmet, JSON 5mb, routes, 404, error handler
  config/db.js              Mongo connect
  constants/index.js        roles, statuses, visitor types
  models/                   User, Visitor, Visit, Pass
  middleware/               protect/authorize, validate, errorHandler
  validations/              auth + visit/search + pass rules
  services/                 business logic (thin controllers)
  controllers/              HTTP adapters
  routes/                   /api/auth, /api/visitors, /api/visits, /api/passes
  utils/                    response helpers, seed.js
```

### Auth (`/api/auth`)

| Method | Path | Who |
|--------|------|-----|
| POST | `/login` | public |
| GET | `/me` | any logged-in |
| POST | `/users` | admin — create guard/admin |
| GET | `/users` | admin |
| PATCH | `/users/:id/toggle` | admin — activate/deactivate |

### Visitors (`/api/visitors`) — all authenticated

| Method | Path | Who |
|--------|------|-----|
| GET | `/search?q=` | admin, guard |
| GET | `/` | admin |
| GET | `/:id` | admin, guard |
| PUT | `/:id` | admin, guard |

### Visits (`/api/visits`)

| Method | Path | Who |
|--------|------|-----|
| POST | `/public-register` | **public** |
| POST | `/register` | admin, guard |
| POST | `/scan` | logged-in |
| POST | `/check-in` | logged-in |
| POST | `/check-out` | logged-in |
| GET | `/today` | admin, guard |
| GET | `/` | admin (filterable) |
| GET | `/visitor/:visitorId` | admin, guard |
| GET | `/:id` | admin, guard |

Staff register accepts optional `passId`. Scan/check-in/out resolve a **pass token** to the pass’s current visit, otherwise a one-time visit token.

### Passes (`/api/passes`)

| Method | Path | Who |
|--------|------|-----|
| GET | `/available` | admin, guard |
| GET | `/` | admin |
| POST | `/generate` | admin |
| GET | `/:id` | admin |
| GET | `/:id/history` | admin |
| PATCH | `/:id/toggle` | admin |
| POST | `/:id/release` | admin |

---

## Frontend map

```
frontend/src/
  App.jsx                   routes
  main.jsx                  AuthProvider
  context/AuthContext.jsx   JWT in localStorage
  hooks/useAuth.js
  services/api.js           Axios + JWT header + focus-in health warmup
  layouts/MainLayout.jsx    header, mobile hamburger drawer
  components/ProtectedRoute.jsx
  pages/
    Login.jsx
    PublicRegister.jsx      /register — no auth
    RegisterVisitor.jsx     staff register — phone QR or physical pass
    QrScanner.jsx           camera scan → check-in / check-out (visit or pass)
    GuardDashboard.jsx      today's visits + mobile Scan QR button
    AdminDashboard.jsx      stats, visit log/filter, add/toggle guards
    VisitorSearch.jsx
    VisitorDetails.jsx
    PassManagement.jsx      generate, print, status, disable/release
    PassDetails.jsx         usage history per pass
```

### Routes

| Path | Access |
|------|--------|
| `/` | → `/register` |
| `/register` | public self-register |
| `/login` | public |
| `/admin` | admin |
| `/guard` | guard |
| `/register-visitor` | admin, guard |
| `/qr-scanner` | admin, guard |
| `/search-visitor` | admin, guard |
| `/visitor/:id` | admin, guard |
| `/passes` | admin |
| `/passes/:id` | admin |

Admin nav: Dashboard, Register Visitor, Scan QR, QR Passes, Search Visitor.  
Guard nav: Dashboard, Register Visitor, Scan QR, Search Visitor.

---

## What is already done

- [x] Backend setup, JWT auth, admin seed, user CRUD for admins
- [x] Visitor + Visit models, QR generation, check-in/out
- [x] Public self-registration (`registeredBy` optional)
- [x] Today's visits, search, visit history
- [x] React app: login, guard + admin dashboards, register, scanner, search/details
- [x] Mobile layout (drawer, tables → cards under 768px)
- [x] Password show/hide on login and add-guard
- [x] Render warmup ping on form focus; Netlify SPA redirects
- [x] Reusable numbered QR passes (admin generate/print, guard assign, scan, full usage history)

---

## Not done / backlog (from handbook)

- Photo upload via storage (multer/cloudinary) — photo field exists as string only
- Email/SMS notifications
- Analytics / export CSV-PDF
- Rate limit on login
- `checked_out` constant vs 3-step runtime — leftover, not wired

---

## Ready for your next changes

Baseline is **Milestone 6** — reusable physical QR passes with visit history per card.
