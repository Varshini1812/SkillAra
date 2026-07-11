# SkillAra — Development Status & Technical Reference

**Last updated:** July 2026  
**Status:** Active development — ready for GitHub, not production-hardened

This document is the single source of truth for what has been built, how the system works, and what remains.

---

## Table of contents

1. [Product overview](#1-product-overview)
2. [Architecture](#2-architecture)
3. [Repository layout](#3-repository-layout)
4. [Multi-tenancy model](#4-multi-tenancy-model)
5. [Authentication & authorization](#5-authentication--authorization)
6. [Development completed](#6-development-completed)
7. [Admin panel features](#7-admin-panel-features)
8. [Backend API overview](#8-backend-api-overview)
9. [Data models](#9-data-models)
10. [Local development guide](#10-local-development-guide)
11. [Environment configuration](#11-environment-configuration)
12. [Known limitations & roadmap](#12-known-limitations--roadmap)
13. [Changelog summary](#13-changelog-summary)

---

## 1. Product overview

SkillAra is a **multi-tenant SaaS learning platform**. Each customer organization (tenant) operates in an isolated workspace identified by a subdomain:

```
acmebootcamp.skillara.com   → Acme Bootcamp workspace
codexacademy.skillara.com   → Codex Academy workspace
```

Three user-facing applications share one API:

| Application | Users | Purpose |
|-------------|-------|---------|
| **Admin Panel** (super) | Super Admin | Create/manage organizations, plans, platform settings |
| **Admin Panel** (tenant) | Tenant Admin | Manage users, courses, org branding |
| **Client App** | Students | Enroll, learn, take quizzes, use AI tutor |

---

## 2. Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         Browser                                  │
├──────────────────┬──────────────────────┬───────────────────────┤
│  Admin Panel     │  Admin Panel         │  Client App           │
│  localhost:5174  │  {sub}.localhost:5174│  {sub}.localhost:5173 │
│  (super admin)   │  (tenant admin)      │  (students)           │
└────────┬─────────┴──────────┬───────────┴───────────┬───────────┘
         │                    │                       │
         │    Vite dev proxy /api → localhost:5000     │
         └────────────────────┼───────────────────────┘
                              ▼
                 ┌────────────────────────┐
                 │   SkillAra_server      │
                 │   Express 5 + MongoDB  │
                 │   Port 5000            │
                 └────────────────────────┘
```

### Request flow (tenant-scoped)

1. Frontend resolves subdomain from hostname (`acmebootcamp.localhost`)
2. API requests include `Authorization: Bearer <token>` and optionally `X-Tenant-Subdomain`
3. `tenant-context` middleware resolves tenant from host, header, body, or query
4. `auth` middleware validates JWT and attaches `req.user`
5. Role guards enforce `SUPER_ADMIN`, `TENANT_ADMIN`, `TUTOR`, or `STUDENT`

---

## 3. Repository layout

```
SkillAra/
├── README.md                    # Project overview & quick start
├── docs/
│   └── DEVELOPMENT.md           # This document
│
├── SkillAra_server/
│   ├── server.js                # Express entry point
│   ├── config/db.js             # MongoDB connection
│   ├── controllers/             # Route handlers
│   ├── middlewares/             # auth, tenant-context, plan limits, upload
│   ├── models/                  # Mongoose schemas
│   ├── routes/                  # API route definitions
│   ├── services/                # JWT, password, security, AI
│   └── utils/                   # CORS, tenant resolution, seed scripts
│
├── SkillAra_adminpanel/
│   └── src/
│       ├── api/                 # Axios client + admin API helpers
│       ├── components/          # UI, create-tenant wizard, edit panel
│       ├── context/             # AuthContext
│       ├── pages/               # Login, Tenants, Users, Roles, etc.
│       └── utils/               # tenant, roles, logo helpers
│
└── SkillAra_client/
    └── src/
        ├── api/                 # Client API
        ├── context/             # AuthContext
        └── pages/               # Login, workspace finder
```

---

## 4. Multi-tenancy model

### Tenant identification

Tenants are resolved in this order (`utils/resolve-tenant-request.js`):

1. `X-Tenant-Subdomain` header
2. Subdomain from `Host` / `Origin` / `X-Forwarded-Host`
3. `subdomain` or `tenant` in request body
4. `tenant` query parameter

### Tenant record

Each organization has:

| Field | Description |
|-------|-------------|
| `tenant_name` | Display name |
| `sub_domain` | Unique slug (e.g. `acmebootcamp`) |
| `domain` | Full domain (e.g. `acmebootcamp.skillara.com`) |
| `email` | Organization contact email |
| `status` | Active (`true`) or inactive (`false`) |
| `logo` | Base64 image blob + metadata |
| `branding` | `welcome_message`, `primary_color`, `secondary_color` |
| `planId` | Reference to subscription plan |
| `subscriptionStatus` | `ACTIVE`, `EXPIRED`, or `TRIAL` |
| `user_count` | Cached active user count |

### Data isolation

- All tenant users, courses, enrollments, etc. are scoped by `tenantId`
- Super admin has `tenantId: null` and platform-wide access
- Email uniqueness is per-tenant (`email + tenantId` compound index)

---

## 5. Authentication & authorization

### Token model

- **Access token** — short-lived JWT in `Authorization: Bearer` header
- **Refresh token** — stored in `sessionStorage`, sent on refresh/logout
- Sessions persisted in `Session` collection (hashed refresh token)

### Login endpoints

| Endpoint | Role | Notes |
|----------|------|-------|
| `POST /api/auth/admin/login` | `SUPER_ADMIN` | No tenant context required |
| `POST /api/auth/tenant/login` | `TENANT_ADMIN` | Requires subdomain resolution |
| `POST /api/auth/register` | `STUDENT` | Self-registration within tenant |
| `POST /api/auth/refresh` | Any | Rotate tokens |
| `POST /api/auth/logout` | Any | Invalidate session |
| `GET /api/auth/me` | Any | Current user profile |

### Roles

| Role | Code | Assigned when |
|------|------|---------------|
| Super Admin | `SUPER_ADMIN` | Server seed on first startup |
| Organization Admin | `TENANT_ADMIN` | Organization creation (owner step) |
| Tutor / Instructor | `TUTOR` | Tenant admin creates user |
| Student | `STUDENT` | Tenant admin creates user, or self-registers |

Role labels in the admin UI (`utils/roles.js`):

- `SUPER_ADMIN` → Super Admin
- `TENANT_ADMIN` → Organization Admin
- `TUTOR` → Tutor / Instructor
- `STUDENT` → Student

### Security measures implemented

- bcrypt password hashing
- Login rate limiting (5 req/min)
- Account lockout after failed attempts
- Helmet security headers
- CORS with configurable origins + `*.localhost` in dev
- `mongo-sanitize` on request body
- Plan-based AI usage limits (`checkPlanLimits` middleware)

---

## 6. Development completed

### Phase 1 — Core platform foundation

- [x] Express 5 server with MongoDB connection
- [x] Multi-tenant middleware and subdomain resolution
- [x] User model with four roles
- [x] JWT auth (access + refresh) with session store
- [x] Super admin auto-seed
- [x] Default plan seeding
- [x] Structured API responses (`prepareResponseMsg`)
- [x] Winston logging + request duration logging
- [x] Global error handler

### Phase 2 — Organization management (Super Admin)

- [x] List all tenants
- [x] Create tenant with admin user (`TENANT_ADMIN` role auto-assigned)
- [x] Update tenant (name, email, plan, branding, logo)
- [x] Activate / deactivate tenant
- [x] Subdomain availability check
- [x] Public tenant resolve endpoint (for login branding)
- [x] 6-step create organization wizard UI
- [x] Edit organization slide-over panel
- [x] Active/inactive toggle on tenant list

### Phase 3 — Branding & tenant login experience

- [x] Tenant `branding` schema (welcome message, primary/secondary colors)
- [x] Logo upload (base64) on create and edit
- [x] Branding step in create wizard with login preview
- [x] Tenant admin login shows org logo, name, colors, welcome message
- [x] CORS fixes for subdomain dev (`*.localhost`, LAN IPs)
- [x] Vite proxy forwards tenant headers

### Phase 4 — User management (Tenant Admin)

- [x] List users in organization
- [x] Create user with role (`TUTOR` or `STUDENT`)
- [x] Role displayed in UI with friendly labels and badges
- [x] Success confirmation showing assigned role
- [x] User count synced on tenant record

### Phase 5 — Learning platform backend

- [x] Plans and subscriptions
- [x] Courses, modules, lessons (CRUD)
- [x] Enrollments
- [x] User progress tracking
- [x] Quizzes and quiz attempts
- [x] Assignment submissions
- [x] AI endpoints (tutor, quiz generation, summarization) with usage tracking

### Phase 6 — Admin UI polish

- [x] Bearer token auth in admin panel (`sessionStorage`)
- [x] Protected routes by role
- [x] Toast notifications
- [x] Roles & Permissions page (UI only, localStorage)
- [x] Removed development credentials from login screen
- [x] Dashboard shell with navigation

### Phase 7 — Client app foundation

- [x] Workspace finder flow
- [x] Tenant-scoped login
- [x] Auth context with token storage
- [x] Vite proxy configuration

---

## 7. Admin panel features

### Super Admin routes

| Route | Page | Status |
|-------|------|--------|
| `/` | Dashboard | Shell |
| `/tenants` | Organizations list + create/edit panels | Complete |
| `/tenants/new` | Full-page create wizard | Complete |
| `/roles` | Roles & Permissions matrix | UI only |
| `/billing` | Billing and plans | Coming soon |
| `/platform/users` | Platform users | Coming soon |
| `/moderation` | Moderation | Coming soon |
| `/analytics` | Analytics | Coming soon |
| `/ai-system` | AI and system | Coming soon |
| `/settings` | Settings | Coming soon |

### Tenant Admin routes

| Route | Page | Status |
|-------|------|--------|
| `/` | Dashboard | Shell |
| `/users` | User list with role badges | Complete |
| `/users/new` | Create user (Student / Tutor) | Complete |

### Create Organization wizard (6 steps)

1. **Organization info** — name, contact email, country
2. **Subdomain** — real-time availability check, preview URL
3. **Branding** — logo, welcome message, brand colors, login preview
4. **Plan** — select subscription plan
5. **Owner** — admin name, email, password, assigned role (`TENANT_ADMIN`)
6. **Review** — summary before submit

On success, the API returns `tenantAdminUser` with `id`, `email`, and `role`.

---

## 8. Backend API overview

Base URL: `/api`

### Auth — `/api/auth`

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/admin/login` | Public | Super admin login |
| POST | `/tenant/login` | Public | Tenant admin login |
| POST | `/register` | Tenant | Student self-registration |
| POST | `/refresh` | Public | Refresh tokens |
| POST | `/logout` | Public | Logout session |
| POST | `/logout-all` | User | Logout all sessions |
| GET | `/me` | User | Current user |
| POST | `/change-password` | User | Change password |

### Tenants — `/api/tenants`

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/` | Super Admin | List tenants |
| POST | `/` | Super Admin | Create tenant + admin |
| GET | `/resolve` | Public | Resolve tenant by subdomain |
| GET | `/check/:subdomain` | Public | Subdomain availability |
| GET | `/:id` | Super Admin | Get tenant |
| PATCH | `/:id` | Super Admin | Update tenant |
| PATCH | `/:id/status` | Super Admin | Activate/deactivate |

### Users — `/api/users`

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/` | Tenant Admin | List users |
| POST | `/` | Tenant Admin | Create user (TUTOR/STUDENT) |
| GET | `/:id` | Tenant Admin | Get user |
| PUT | `/:id` | Tenant Admin | Update user |
| PATCH | `/:id/status` | Tenant Admin | Enable/disable user |
| DELETE | `/:id` | Tenant Admin | Delete user |

### Plans — `/api/plans`

| Method | Path | Description |
|--------|------|-------------|
| GET | `/` | List plans |
| POST | `/` | Create plan (super admin) |
| PUT/PATCH | `/:id` | Update plan |

### Courses — `/api/courses`

Full CRUD for courses, modules, and lessons. Tenant-scoped.

### Enrollments — `/api/enrollments`

Enroll, list, drop courses. Student-facing.

### Progress — `/api/progress`

Track and retrieve learning progress per course.

### Quizzes — `/api/quizzes`

Create quizzes, submit attempts, view results.

### Assignments — `/api/assignments`

Submit assignments, list submissions.

### AI — `/api/ai`

| Endpoint | Description |
|----------|-------------|
| `POST /tutor` | AI tutoring chat |
| `POST /generate-quiz` | AI quiz generation |
| `POST /summarize` | Content summarization |

All AI endpoints are plan-limited via `checkPlanLimits`.

---

## 9. Data models

| Model | Collection | Purpose |
|-------|------------|---------|
| `User` | User | All platform users |
| `Tenant` | Tenant | Organizations |
| `Plan` | Plan | Subscription tiers |
| `Subscription` | Subscription | Tenant plan linkage |
| `Session` | Session | Refresh token sessions |
| `Course` | Course | Courses |
| `Module` | Module | Course modules |
| `Lesson` | Lesson | Module lessons |
| `Enrollment` | Enrollment | Student course enrollments |
| `UserProgress` | UserProgress | Lesson/course progress |
| `Quiz` | Quiz | Quizzes |
| `QuizAttempt` | QuizAttempt | Student quiz attempts |
| `Submission` | Submission | Assignment submissions |
| `AIUsage` | AIUsage | AI token/request tracking |

---

## 10. Local development guide

### Step-by-step: first organization

1. Start MongoDB and the API server
2. Start admin panel (`npm run dev` in `SkillAra_adminpanel`)
3. Open `http://localhost:5174/login`
4. Log in as super admin (credentials from `SkillAra_server/.env`)
5. Click **+ New Organization** and complete the 6-step wizard
6. Note the subdomain (e.g. `acmebootcamp`) and owner credentials
7. Open `http://acmebootcamp.localhost:5174/login` as tenant admin
8. Go to **Users → + Create User** to add students/tutors
9. Students use `http://acmebootcamp.localhost:5173/login`

### Common issues

| Issue | Cause | Fix |
|-------|-------|-----|
| `401` on `/api/auth/me` | No/expired token | Log in again |
| `404` on `/api/tenants/resolve` | Wrong subdomain or inactive tenant | Verify subdomain exists in DB |
| `Tenant context required` | API call without tenant resolution | Use subdomain URL or pass `X-Tenant-Subdomain` |
| CORS blocked | Origin not allowed | Add origin to `CORS_ORIGINS` or use Vite proxy |
| Schema changes not applied | Server not restarted | Restart `npm run dev` in server |

### Dev proxy (admin panel)

Vite proxies `/api` to `http://localhost:5000` and forwards:

- `X-Forwarded-Host`
- `X-Tenant-Subdomain`

Do **not** set `VITE_API_URL` unless calling the API directly (bypasses proxy).

---

## 11. Environment configuration

### Server — `SkillAra_server/.env`

```env
PORT=5000
NODE_ENV=development
MONGO_URI=mongodb://127.0.0.1:27017/skillara
ROOT_DOMAIN=skillara.com
JWT_ACCESS_SECRET=<strong-secret>
JWT_REFRESH_SECRET=<strong-secret>
CORS_ORIGINS=http://localhost:5173,http://localhost:5174
DEFAULT_SUPER_ADMIN_EMAIL=superadmin@skillara.com
DEFAULT_SUPER_ADMIN_PASSWORD=<set-your-own>
OPENAI_API_KEY=
```

### Admin panel — `SkillAra_adminpanel/.env`

```env
VITE_ROOT_DOMAIN=skillara.com
VITE_API_URL=
```

### Client — `SkillAra_client/.env`

```env
VITE_ROOT_DOMAIN=skillara.com
VITE_API_URL=
```

---

## 12. Known limitations & roadmap

### Not yet implemented

- [ ] Stripe / payment integration
- [ ] Backend for Roles & Permissions (currently UI + localStorage only)
- [ ] Billing management UI
- [ ] Platform analytics dashboard
- [ ] Email verification and password reset
- [ ] Production DNS + SSL deployment guide
- [ ] Full student learning UI (course player, quiz UI)
- [ ] File upload to cloud storage (currently base64 logo)
- [ ] API documentation (Swagger was removed; can be re-added)
- [ ] Automated tests

### Production checklist (before go-live)

- [ ] Rotate all secrets and super admin password
- [ ] Use MongoDB Atlas with IP allowlist
- [ ] Set `NODE_ENV=production`
- [ ] Configure wildcard DNS and HTTPS
- [ ] Restrict `CORS_ORIGINS` to real domains
- [ ] Remove or protect seed credentials
- [ ] Set up CI/CD and environment secrets in GitHub

---

## 13. Changelog summary

### Auth & multi-tenancy

- Unified tenant resolution from host, headers, body, and query
- Bearer token flow for admin and client frontends
- Improved tenant login error messages
- CORS support for `*.localhost` and development LAN IPs

### Organization management

- Full tenant CRUD with plan assignment
- 6-step premium create organization wizard
- Edit panel: logo, branding colors, welcome message, plan, status
- Active/inactive organization toggle
- Subdomain availability API

### Branding

- Tenant branding schema and API
- Branded tenant admin login (logo, colors, welcome message)
- Login preview in create wizard

### Users & roles

- Role assignment on org creation (`TENANT_ADMIN`)
- Role selection on user creation (`TUTOR` / `STUDENT`)
- Friendly role labels and badges in admin UI
- Removed hardcoded dev credentials from login screen

### Learning platform backend

- Courses, modules, lessons
- Enrollments and progress
- Quizzes and assignments
- AI features with plan limits

### Admin UI

- Super admin vs tenant admin route separation
- Roles & Permissions matrix (UI prototype)
- Toast notifications and protected routes

---

*For questions or contributions, open an issue on GitHub after the repository is published.*
