# CampusConnect

> **Multi-tenant ISP SaaS platform** — prepaid Starlink-backed internet for campuses and student housing, sold through a mobile app and managed through a web dashboard.

Built by a solo full-stack engineer. Currently running locally, pre-launch.

---

## What It Does

CampusConnect lets organizations (hostels, colleges, student housing) resell prepaid internet access to their residents — without exposing voucher codes, without manual provisioning, and without friction.

**The student experience:**
1. Open the CampusConnect mobile app
2. Select a data plan (e.g. 50GB / 30 days / ₦5,000)
3. Pay via Paystack
4. Payment is verified server-side via webhook
5. Data balance is activated on the account
6. Device is automatically authorized on the Wi-Fi (via Omada)
7. Internet works — no voucher code, no portal redirect, no admin intervention

**The admin experience:**
- Real-time dashboard showing students, revenue, network health, and active devices
- Per-organization plan management, pricing, and Omada configuration
- Multi-tenant isolation — each organization sees only their own data

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         AI Layer                                 │
│   Google Stitch MCP ◄──SSE──► Antigravity Agents (Python)       │
│                               Design · Backend · Orchestrator    │
└───────────────────────────────────┬─────────────────────────────┘
                                    │ MCP Tool Calls
┌───────────────────────────────────▼─────────────────────────────┐
│                        Go Backend (chi)                          │
│   REST API · JWT Auth · RBAC · Rate Limiting · Zap Logger        │
│                         PostgreSQL (sqlx)                        │
└───────────────────┬──────────────────────────┬──────────────────┘
                    │ REST                      │ REST
         ┌──────────▼──────────┐    ┌──────────▼──────────┐
         │  Next.js 16 Admin   │    │   Flutter Mobile    │
         │  Tailwind CSS v4    │    │   Riverpod · Dio    │
         │  App Router         │    │   Clean Architecture│
         └─────────────────────┘    └─────────────────────┘
                                              │
                              ┌───────────────▼───────────────┐
                              │      External Integrations     │
                              │  Paystack · Omada · Firebase   │
                              │  FCM · Starlink (upstream)     │
                              └───────────────────────────────┘
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| **Backend** | Go 1.22+ · chi router · sqlx · zap · JWT · PostgreSQL 15 |
| **Mobile** | Flutter · Dart · Riverpod · GoRouter · Dio · Clean Architecture |
| **Admin Dashboard** | Next.js 16 (App Router) · Tailwind CSS v4 · React |
| **AI Layer** | Python · Google Antigravity SDK · Google Stitch MCP |
| **Infrastructure** | Docker · Docker Compose · Firebase FCM |
| **Payments** | Paystack (webhook verification · idempotency · subaccounts) |
| **Networking** | Omada Controller (device auth/deauth · captive portal · voucher pool) |
| **Internet Source** | Starlink (upstream WAN) |

---

## Key Engineering Decisions

### Payment Safety
The mobile app never determines whether a payment succeeded. The flow is:

```
App initiates checkout
    → Backend creates pending transaction
    → Paystack processes payment
    → Paystack sends signed webhook
    → Backend verifies webhook signature
    → Backend calls Paystack verify endpoint directly
    → Backend checks reference + amount
    → Only then: data activated + device authorized
```

This prevents fake payment screens, replayed references, and app-side manipulation.

### Multi-Tenant Isolation
Every database query is scoped to `organization_id`. Plans, users, devices, transactions, Omada settings, and Paystack subaccounts are fully isolated per tenant. Tenant A cannot access, modify, or even observe Tenant B's data.

### Network Provisioning (3 modes)
CampusConnect supports three Omada provisioning strategies depending on the organization's hardware setup:

1. **Direct Authorization** — CampusConnect authorizes/deauthorizes MAC addresses through the Omada API after payment events (preferred)
2. **External Portal + Accounting** — Omada redirects new devices to CampusConnect, which checks account status and authorizes inline
3. **Hidden Voucher Pool** — CampusConnect manages Omada vouchers internally; users never see or type a code (fallback)

### Data Cap Logic
Plans enforce two simultaneous limits:

```
remaining_mb: tracks data consumption
expires_at:   tracks time validity

Access ends when EITHER limit is reached — whichever comes first.
```

A background worker polls subscriptions, deauthorizes expired/exhausted devices, and triggers push notifications at configurable thresholds (low data, expiry warning, access removed).

### Balance Stacking
If a user purchases a new plan while an existing one is active, CampusConnect carries over remaining data and extends expiry — configurable per organization.

---

## Backend API (Go)

Built with `chi` router, clean layered architecture: Config → Models → Repositories → Services → Handlers → Middleware → Router.

**Auth**
```
POST /api/auth/register
POST /api/auth/login
POST /api/auth/refresh
```

**Plans**
```
GET  /api/plans           — list active plans (student-facing)
POST /api/plans           — create plan (admin only)
PUT  /api/plans/:id       — update plan (admin only)
```

**Payments**
```
POST /api/payments/initialize    — create pending transaction + Paystack checkout
POST /api/payments/webhook       — Paystack webhook handler (signature verified)
GET  /api/payments/verify/:ref   — manual verification endpoint
```

**Users & Devices**
```
GET  /api/users                  — paginated student directory (admin)
GET  /api/users/:id              — student profile + devices + subscription
PUT  /api/users/:id/status       — suspend/activate student
POST /api/devices/register       — register device MAC address
```

**Notifications**
```
POST /api/notifications/push-token    — register FCM device token
GET  /api/notifications               — list in-app notifications
```

**Database migrations**
```bash
cd backend && make migrate-up
```

---

## Flutter Mobile App

Clean Architecture with feature-first directory layout.

```
lib/
├── app/          # Theme, colors, typography, routes
├── core/         # Dio client, interceptors, token manager, error mapping
├── features/
│   ├── auth/         # Sign-in, sign-up, SecureStorage persistence
│   ├── dashboard/    # Glassmorphic consumption metrics, plan status
│   ├── notifications/
│   └── support/      # FAQ, ticket list
└── shared/       # Shimmer loaders, glass cards, primary buttons, badges
```

State managed with **Riverpod**. Routing with **GoRouter** including auth redirect guards. HTTP via **Dio** with JWT interceptor and automatic token refresh.

---

## Next.js Admin Dashboard

Built with **Next.js 16 App Router** and **Tailwind CSS v4**, dark-mode first.

| Route | Purpose |
|---|---|
| `/` | Bento overview — students, MRR, uptime, live logs |
| `/students` | Directory with plan status, device MACs, live speeds |
| `/plans` | Plan table, pricing, availability toggles, create modal |
| `/payments` | Financial summary, weekly charts, gateway health |
| `/network` | Starlink status, AP health grid, latency histograms |
| `/support` | Ticket queue, priority tags, live chat panel |
| `/analytics` | Growth, revenue, bandwidth charts |
| `/settings` | Branding, notifications, admin invites, revenue share |

---

## AI Layer (Google Antigravity + Stitch MCP)

Three specialized Python agents coordinated by a master orchestrator:

- **Design Agent** — connects to Google Stitch MCP, inspects wireframes, compiles layout schemas
- **Backend Agent** — reviews Go schema declarations, validates endpoint routers
- **Master Orchestrator** — translates prompts into multi-agent tasks (download design → scaffold React component → update Go model)

---

## Local Development

**Prerequisites:** Go 1.22+ · Flutter 3.24+ · Node.js 20+ · Python 3.11+ · Docker

```bash
# 1. Clone and configure
git clone https://github.com/Scientist265/campusconnect-public
cp .env.example .env

# 2. Start PostgreSQL
docker compose up -d

# 3. Run Go backend
cd backend && make migrate-up && make run
# → http://localhost:8080

# 4. Run admin dashboard
cd admin && npm install && npm run dev
# → http://localhost:3000

# 5. Run Flutter app
cd mobile && flutter pub get && flutter run

# 6. Run AI agents
cd ai && pip install -r requirements.txt
python -m agents.orchestrator
```

---

## Production Deployment

| Component | Recommended hosting |
|---|---|
| Go API | AWS ECS / GCP Cloud Run / VPS + Nginx + Let's Encrypt |
| Next.js Dashboard | Vercel / Netlify |
| PostgreSQL | AWS RDS / GCP Cloud SQL |
| Flutter App | Google Play Store (AAB) + Apple App Store (Xcode Archive) |

---

## Business Model

CampusConnect is a **multi-tenant SaaS** — organizations pay to use the platform and keep revenue from their own students. Each tenant has:

- Isolated data and user base
- Their own Paystack subaccount (revenue goes directly to them)
- Their own Omada controller configuration
- Their own plan pricing and data tiers
- A platform super-admin onboards new organizations via a guided dashboard workflow

---

## Status

| Component | Status |
|---|---|
| Go backend | ✅ Built · running locally |
| Flutter mobile app | ✅ Built · running on device |
| Next.js admin dashboard | ✅ Built · running locally |
| Paystack integration | ✅ Webhook verified · idempotency implemented |
| Omada integration | 🔄 Direct auth mode in testing |
| Firebase FCM | ✅ Push tokens · notification delivery |
| Production deployment | 🔜 Pre-launch |
| App Store / Play Store | 🔜 Pending deployment |

---

## About the Builder

**Ibraheem Omowumi H.** — Flutter Engineer & Go backend developer.

5 production Flutter apps live on the App Store:
- **DigitalPurse** — 200k+ users, fintech virtual card platform
- **Helperr** — Canadian gig marketplace, Stripe, WebSocket bidding
- **BringThisFood Vendor** — real-time restaurant ops, integrated wallet
- **HOLA** — AI + voice + 3D GLB rendering, Arabic/English, live in the UK
- **VOYA** — Stripe + Paystack, visa digitization, AI concierge

→ [Arc.dev profile](https://arc.dev/@dev.scientist)
→ [Upwork profile](https://www.upwork.com/freelancers/~01d560f699e7a7e9bc)

---

*CampusConnect is an active pre-launch product. Architecture and features subject to change.*
