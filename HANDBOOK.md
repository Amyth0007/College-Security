# College Visitor Management System (VMS) — Handbook

## Project Overview

A MERN-stack web application for managing visitors at a college campus. Security guards register visitors and issue QR codes; visitors present QR codes for check-in/check-out; admins oversee the entire system via a dashboard.

---

## Current Architecture

### Backend

- **Runtime:** Node.js + Express.js (v5)
- **Database:** MongoDB via Mongoose
- **Auth:** JWT (stateless), bcryptjs for password hashing
- **Validation:** express-validator
- **Security:** Helmet, CORS
- **Logging:** Morgan (dev mode)

### Frontend

*Not yet implemented.*

### Folder Structure

```
VMS/
├── backend/
│   ├── src/
│   │   ├── config/         # DB connection
│   │   ├── constants/      # Roles, statuses, transitions
│   │   ├── controllers/    # Route handlers (thin)
│   │   ├── middleware/      # Auth, validation, error handler
│   │   ├── models/         # Mongoose schemas
│   │   ├── routes/         # Express routers
│   │   ├── services/       # Business logic
│   │   ├── utils/          # Helpers (response, seed)
│   │   ├── validations/    # express-validator rule sets
│   │   └── server.js       # Entry point
│   ├── .env
│   ├── .gitignore
│   └── package.json
├── frontend/               # (Phase 2)
└── HANDBOOK.md
```

---

## Database Schema

### Users Collection

| Field     | Type    | Details                          |
|-----------|---------|----------------------------------|
| name      | String  | Required, trimmed                |
| email     | String  | Required, unique, lowercase      |
| password  | String  | Hashed (bcrypt 12 rounds), `select: false` |
| role      | String  | `admin` or `guard`               |
| isActive  | Boolean | Default `true`                   |
| timestamps| -       | `createdAt`, `updatedAt` auto    |

**Indexes:** Unique on `email`.

### Visitors Collection

| Field          | Type    | Details                          |
|----------------|---------|----------------------------------|
| name           | String  | Required, trimmed                |
| mobile         | String  | Required, trimmed                |
| visitorType    | String  | `parent`, `vendor`, `official`, `student`, `other` |
| photo          | String  | Base64/URL, optional (default null) |
| purpose        | String  | Trimmed, default empty           |
| hostName       | String  | Who to visit                     |
| hostDepartment | String  | Department of host               |
| timestamps     | -       | `createdAt`, `updatedAt` auto    |

**Indexes:** Text index on `name` and `mobile`.

### Visits Collection

| Field          | Type     | Details                          |
|----------------|----------|----------------------------------|
| visitor        | ObjectId | Ref → Visitor (required)         |
| registeredBy   | ObjectId | Ref → User (guard who registered)|
| status         | String   | `registered` → `checked_in` → `completed` |
| qrCode         | String   | Base64 PNG data URL              |
| qrToken        | String   | Unique, crypto random hex (64 chars) |
| checkInTime    | Date     | Set on check-in                  |
| checkOutTime   | Date     | Set on check-out                 |
| purpose        | String   | Visit purpose                    |
| hostName       | String   | Who to visit                     |
| hostDepartment | String   | Department                       |
| timestamps     | -       | `createdAt`, `updatedAt` auto    |

**Indexes:** Unique on `qrToken`, index on `status`, index on `createdAt` desc.

---

## Implemented Features

- [x] Project setup (Express, Mongoose, env config)
- [x] User model with password hashing
- [x] JWT Authentication (login)
- [x] Role-based authorization middleware
- [x] Auth routes (login, me, user CRUD)
- [x] Input validation (express-validator)
- [x] Centralized error handling
- [x] Admin seed script
- [x] Consistent JSON response format
- [x] Visitor model + CRUD
- [x] Visit model + lifecycle (state transitions enforced)
- [x] QR code generation (crypto token + qrcode lib)
- [x] Check-In via QR
- [x] Check-Out via QR
- [x] Today's visits API
- [x] Visitor search API
- [x] Visit history per visitor
- [x] Reusable numbered QR passes (admin-managed, history per card)
- [x] Frontend (Phase 2)

---

## API Documentation

### Base URL: `http://localhost:5000/api`

### Health Check

| Method | Route         | Auth | Purpose          |
|--------|---------------|------|------------------|
| GET    | `/api/health` | No   | Server status    |

---

### Auth Routes (`/api/auth`)

#### POST `/api/auth/login`

Login and receive JWT token.

**Auth:** None

**Body:**
```json
{ "email": "admin@college.com", "password": "Admin@123" }
```

**Success 200:**
```json
{
  "success": true,
  "message": "Login successful",
  "data": {
    "user": { "_id": "...", "name": "Admin", "email": "admin@college.com", "role": "admin" },
    "token": "eyJhbG..."
  }
}
```

---

#### GET `/api/auth/me`

Get current authenticated user.

**Auth:** Bearer token (any role)

**Success 200:**
```json
{
  "success": true,
  "message": "Current user",
  "data": { "_id": "...", "name": "...", "email": "...", "role": "..." }
}
```

---

#### POST `/api/auth/users`

Create a new user (guard or admin).

**Auth:** Bearer token (Admin only)

**Body:**
```json
{ "name": "Guard One", "email": "guard1@college.com", "password": "Guard@123", "role": "guard" }
```

**Success 201:**
```json
{ "success": true, "message": "User created successfully", "data": { ... } }
```

---

#### GET `/api/auth/users`

List all users.

**Auth:** Bearer token (Admin only)

**Success 200:**
```json
{ "success": true, "message": "Users fetched", "data": [ ... ] }
```

---

#### PATCH `/api/auth/users/:id/toggle`

Activate or deactivate a user.

**Auth:** Bearer token (Admin only)

**Success 200:**
```json
{ "success": true, "message": "User status updated", "data": { ... } }
```

### Visitor Routes (`/api/visitors`)

| Method | Route                     | Auth          | Purpose               |
|--------|---------------------------|---------------|------------------------|
| GET    | `/api/visitors`           | Admin         | List all visitors      |
| GET    | `/api/visitors/search?q=` | Admin, Guard  | Search by name/mobile  |
| GET    | `/api/visitors/:id`       | Admin, Guard  | Get visitor details    |
| PUT    | `/api/visitors/:id`       | Admin, Guard  | Update visitor         |

---

### Visit Routes (`/api/visits`)

#### POST `/api/visits/register`

Register a visitor and generate QR code (Requires auth).

**Auth:** Bearer token (Admin, Guard)

**Body:**
```json
{
  "name": "Rahul Sharma",
  "mobile": "9876543210",
  "visitorType": "parent",
  "purpose": "Meet HOD",
  "hostName": "Dr. Gupta",
  "hostDepartment": "Computer Science"
}
```

**Success 201:** Returns visit with visitor data, QR code (base64), and qrToken.

---

#### POST `/api/visits/public-register`

Public self-registration for visitors (No auth required). Generates a QR code directly.

**Auth:** None

**Body:** (Same schema as `/api/visits/register`)

**Success 201:** Returns visit with visitor data, QR code (base64), and qrToken. `registeredBy` field remains null.

---

#### POST `/api/visits/scan`

Scan QR → get visit info before acting.

**Auth:** Bearer token | **Body:** `{ "qrToken": "..." }`

---

#### POST `/api/visits/check-in`

Check in visitor (status must be `registered`).

**Auth:** Bearer token | **Body:** `{ "qrToken": "..." }`

---

#### POST `/api/visits/check-out`

Check out visitor (status must be `checked_in`). Sets status to `completed`.

**Auth:** Bearer token | **Body:** `{ "qrToken": "..." }`

---

| Method | Route                          | Auth          | Purpose                |
|--------|--------------------------------|---------------|-------------------------|
| GET    | `/api/visits/today`            | Admin, Guard  | Today's visits          |
| GET    | `/api/visits`                  | Admin         | All visits (filterable) |
| GET    | `/api/visits/visitor/:visitorId`| Admin, Guard  | Visitor's visit history |
| GET    | `/api/visits/:id`              | Admin, Guard  | Single visit details    |

---

## Decisions Log

| # | Decision | Reason |
|---|----------|--------|
| 1 | JWT over sessions | Stateless auth, simpler frontend integration, no server-side session store needed |
| 2 | bcryptjs over bcrypt | Pure JS — no native build dependencies, works everywhere |
| 3 | Service layer pattern | Keeps controllers thin; business logic is testable and reusable |
| 4 | express-validator | Declarative validation in route definitions; cleaner than manual checks |
| 5 | `select: false` on password | Never leak password hashes in normal queries |
| 6 | Centralized error handler | Single place to format Mongoose and generic errors consistently |
| 7 | Crypto random QR tokens | `crypto.randomBytes(32).hex` — unpredictable, no visitor data exposed in QR |
| 8 | QR as base64 data URL | No file storage needed; embed directly in API response and frontend `<img>` |
| 9 | Visitor reuse by mobile | If same mobile registers again, reuse visitor record — prevents duplicates |
| 10 | Check-out = completed | Simplified from 4-step to 3-step: registered → checked_in → completed |
| 12 | Reusable physical passes | Visitors without a phone get a numbered card. QR encodes a pass token, not a visit. Each assignment creates a Visit so history is preserved. Checkout frees the card. |

---

## Pending Tasks

- [x] Frontend setup with React (Phase 2 — Milestone 3)
- [x] Login page
- [x] Guard Dashboard
- [x] Admin Dashboard (Fully implemented)
- [x] Register Visitor page
- [x] QR Scanner page
- [x] Today's Visitors page (in Guard Dashboard)
- [x] Visitor Search + Details pages

---

## Known Issues

- None yet.

---

## Future Improvements

- Photo upload for visitors (multer/cloudinary)
- Email/SMS notifications
- Visitor analytics dashboard
- Export visit reports (CSV/PDF)
- Rate limiting on login endpoint

---

## File Changes

### Milestone 1 — Project Setup & Authentication

**Files created:**

| File | Purpose |
|------|---------|
| `backend/package.json` | Dependencies and scripts |
| `backend/.env` | Environment variables |
| `backend/.gitignore` | Ignore node_modules and .env |
| `backend/src/server.js` | Express entry point |
| `backend/src/config/db.js` | MongoDB connection |
| `backend/src/constants/index.js` | Roles, statuses, transitions |
| `backend/src/models/User.js` | User schema with bcrypt |
| `backend/src/middleware/auth.js` | JWT protect + role authorize |
| `backend/src/middleware/errorHandler.js` | Global error handler |
| `backend/src/middleware/validate.js` | Express-validator result check |
| `backend/src/validations/authValidation.js` | Login & user creation rules |
| `backend/src/services/authService.js` | Auth business logic |
| `backend/src/controllers/authController.js` | Auth route handlers |
| `backend/src/routes/authRoutes.js` | Auth route definitions |
| `backend/src/utils/response.js` | Standardised JSON helpers |
| `backend/src/utils/seed.js` | Default admin seeder |
| `HANDBOOK.md` | This file |

### Milestone 2 — Visitor CRUD, Visit Lifecycle, QR, Check-In/Out

**Files created:**

| File | Purpose |
|------|---------|
| `backend/src/models/Visitor.js` | Visitor schema (name, mobile, type, photo) |
| `backend/src/models/Visit.js` | Visit schema (status, QR, timestamps) |
| `backend/src/services/visitorService.js` | Visitor CRUD + search logic |
| `backend/src/services/visitService.js` | Register, check-in, check-out, queries |
| `backend/src/validations/visitValidation.js` | Validation rules for visits |
| `backend/src/controllers/visitorController.js` | Visitor route handlers |
| `backend/src/controllers/visitController.js` | Visit route handlers |
| `backend/src/routes/visitorRoutes.js` | Visitor route definitions |
| `backend/src/routes/visitRoutes.js` | Visit route definitions |

**Files modified:**

| File | Change |
|------|--------|
| `backend/src/server.js` | Mounted `/api/visitors` and `/api/visits` routes |
| `backend/src/models/User.js` | Fixed pre-save hook for Mongoose v9 (removed `next()`) |
| `backend/src/models/Visit.js` | Removed duplicate qrToken index |
| `HANDBOOK.md` | Updated with Milestone 2 docs |

### Milestone 3 — Frontend Setup (React)

**Files created:**

| File | Purpose |
|------|---------|
| `frontend/src/index.css` | Global CSS for minimal UI |
| `frontend/src/services/api.js` | Axios interceptors for API calls |
| `frontend/src/context/AuthContext.jsx` | React context for user auth state |
| `frontend/src/hooks/useAuth.js` | Custom hook for auth |
| `frontend/src/layouts/MainLayout.jsx` | Application shell and navbar |
| `frontend/src/components/ProtectedRoute.jsx` | Auth routing wrapper |
| `frontend/src/pages/Login.jsx` | Login form page |
| `frontend/src/pages/AdminDashboard.jsx` | Admin dashboard (Placeholder) |
| `frontend/src/pages/GuardDashboard.jsx` | Guard dashboard showing today's visits |
| `frontend/src/pages/RegisterVisitor.jsx` | Visitor registration and QR display |
| `frontend/src/pages/QrScanner.jsx` | QR scanning for Check-In / Check-Out |

**Files modified:**

| File | Change |
|------|--------|
| `frontend/src/main.jsx` | Wrapped with AuthProvider |
| `frontend/src/App.jsx` | Defined React Router routes and protection |
| `HANDBOOK.md` | Updated with Milestone 3 docs |

### Milestone 4 — Admin Dashboard & Visitor Search

**Files created:**

| File | Purpose |
|------|---------|
| `frontend/src/pages/VisitorSearch.jsx` | Search visitor by name/mobile |
| `frontend/src/pages/VisitorDetails.jsx` | View visitor info & visit history |
| `frontend/src/pages/PublicRegister.jsx` | Public self-registration page |

**Files modified:**

| File | Change |
|------|--------|
| `backend/src/models/Visit.js` | Made `registeredBy` optional to support self-registered visits |
| `backend/src/controllers/visitController.js` | Added `registerPublicVisit` logic |
| `backend/src/routes/visitRoutes.js` | Mounted public `/public-register` endpoint |
| `frontend/src/pages/AdminDashboard.jsx` | Fully implemented Stats, Visits Log, Filter, and Guard Management |
| `frontend/src/pages/Login.jsx` | Added self-registration Link below the form |
| `frontend/src/layouts/MainLayout.jsx` | Added Search Visitor navigation link |
| `frontend/src/App.jsx` | Defined routing for VisitorSearch, VisitorDetails, and PublicRegister |
| `HANDBOOK.md` | Updated with Milestone 4 progress and Public route details |

---

### Milestone 5 — Mobile Responsiveness & UX Polish

**Key Features Implemented:**
1. **Render Server Spin-up (Auto-Warmup)**:
   - Implemented a focus-based auto-ping to `/api/health` in `api.js`. When any user focuses or changes an input field in any form, the client automatically triggers a throttled ping to the backend. This wakes up Render's free tier instance before the user clicks Submit.
2. **Comprehensive Mobile Responsiveness**:
   - **Main Navigation**: Implemented a responsive mobile drawer navigation with a custom hamburger menu.
   - **Tables to Cards**: Replaced desktop-oriented HTML tables on Guard Dashboard, Admin Dashboard, Visitor Search, and Visitor Details with responsive card layouts on screens under 768px wide.
   - **Scan QR Quick Button**: Displayed a persistent "Scan Entry/Exit QR Code" button directly on the Guard Dashboard screen for mobile viewports.
   - **QR Scanner Custom CSS**: Applied styling rules directly onto the `html5-qrcode` library's custom camera request, start/stop scan buttons, and options select menus.
3. **Password Security Toggle**:
   - Integrated a password visibility toggle (using clean inline SVG Eye and Eye-Slash icons) next to the password fields in both `Login.jsx` (Staff Login) and `AdminDashboard.jsx` (Add New Guard form).
   - Added disabled states and loading indicators for all API post actions.
4. **Form Requirement Standardisation**:
   - Set all visitor registration fields to `required` (including Purpose of Visit) except for the host fields.

**Files modified:**
- `frontend/src/services/api.js` (Implemented focus-based backend warmup trigger)
- `frontend/src/layouts/MainLayout.jsx` (Added responsive hamburger toggle drawer & overlay)
- `frontend/src/index.css` (Added card-table responsive rules, custom qr-reader component button style overrides)
- `frontend/src/pages/Login.jsx` (Added password eye visibility toggle and loading state)
- `frontend/src/pages/AdminDashboard.jsx` (Added guard password eye visibility toggle and loading state)
- `frontend/src/pages/GuardDashboard.jsx` (Added mobile-only quick Scan QR action button)
- `frontend/src/pages/RegisterVisitor.jsx` & `PublicRegister.jsx` (Form required updates)
- `frontend/public/_redirects` (Added Netlify redirect rule for Single Page Application routing support)
- `HANDBOOK.md` (Documented UX/mobile responsiveness milestone updates)

### Milestone 6 — Reusable QR Passes

Physical numbered cards (QR-01, QR-02, …) for visitors without a phone. Admin generates and prints them. Guards assign a free pass at registration. Scanner reads the pass token and acts on the **current** visit. Checkout frees the card. Every assignment is stored as a Visit, so admin can see who used a pass, current holder, and use count.

**Files created:**
- `backend/src/models/Pass.js`
- `backend/src/services/passService.js`
- `backend/src/controllers/passController.js`
- `backend/src/routes/passRoutes.js`
- `backend/src/validations/passValidation.js`
- `frontend/src/pages/PassManagement.jsx`
- `frontend/src/pages/PassDetails.jsx`

**Files modified:**
- Visit model (`pass` ref), visit service (assign + resolve scan by pass), server routes
- Register visitor (Phone QR vs Physical pass), dashboards, scanner, visitor details
- Admin nav: QR Passes, Register Visitor, Scan QR

