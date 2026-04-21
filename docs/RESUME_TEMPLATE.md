# Forge — Project Resume Template

When you need to stop building and come back later (next day, next week, next month), this is how you pick up exactly where you left off without losing context.

---

## HOW THIS WORKS

1. Before you stop, ask Forge: **"Give me a project status summary."**
2. Forge fills out the template below with everything about your project
3. Save the result somewhere safe — a text file, a Notion page, a Google Doc
4. When you come back:
   - Open a new Claude conversation
   - Paste the Forge interview prompt
   - When asked if new or continuing, pick **📂 Continuing**
   - Paste your saved status summary
5. Forge reads the context and generates your next session prompt immediately

No restart. No re-interview. No lost progress.

---

## THE TEMPLATE

Copy this, fill it in (or let Forge fill it for you), and save it.

```
FORGE PROJECT STATUS — [date]

PROJECT: [project name]
TYPE: [web app / mobile / desktop / SaaS / AI-powered / internal tool]
USER LEVEL: [non-technical / junior / mid / senior]
LANGUAGE PREFERENCE: [English / Arabic]

STACK:
- Frontend: [tech + version]
- Backend: [tech + version]
- Database: [primary + any secondary like Redis, pgvector]
- Auth: [method — JWT, NextAuth, Supabase, etc.]
- Hosting: [Vercel, Railway, Azure, etc.]
- Key libraries: [anything important — Stripe, Socket.IO, Anthropic SDK, etc.]

ARCHITECTURE: [monolith / MVC / Clean Architecture / unified full-stack]
REPO: [GitHub URL, if pushed]

SESSIONS COMPLETED:
1. [Session title] — [what was built, 1 sentence]
2. [Session title] — [what was built, 1 sentence]
3. [continue listing every session]

CURRENT STATE:
[Plain description — what works, what's half-built, what hasn't been touched.
Be specific: "User registration and login work. Dashboard page exists but has
no real data. Payment integration not started."]

FEATURES BUILT:
- [feature 1]
- [feature 2]
- [feature 3]

FEATURES PLANNED BUT NOT BUILT:
- [feature]
- [feature]

NEXT SESSION SHOULD COVER:
[Specific scope. Example: "Build the Stripe checkout flow — integrate Stripe
SDK, create /api/checkout/session endpoint, add webhook handler for
payment_intent.succeeded, update order status on successful payment."]

KNOWN ISSUES:
[Anything broken, skipped, flagged for later. Example: "Email sending works
in dev but SendGrid key not set for production. Profile picture upload
throws 413 on files over 2MB — needs backend config."]

DECISIONS MADE ALONG THE WAY:
[Important choices that weren't obvious from the stack. Example: "Chose to
store uploaded files in S3 rather than the database. Decided to use
soft-deletes for all user data for compliance. Using Clerk for auth instead
of NextAuth."]

ENV VARIABLES SET:
[List names only, never values. Example: DATABASE_URL, STRIPE_SECRET_KEY,
ANTHROPIC_API_KEY, SENDGRID_API_KEY]
```

---

## EXAMPLE — FILLED OUT

```
FORGE PROJECT STATUS — 2026-04-20

PROJECT: CardShop-GCC
TYPE: SaaS (digital products marketplace)
USER LEVEL: mid
LANGUAGE PREFERENCE: English

STACK:
- Frontend: React 18 + Vite + TypeScript
- Backend: ASP.NET Core 8 (Clean Architecture)
- Database: PostgreSQL 16, Redis 7 for caching
- Auth: JWT with refresh tokens, role-based authorization
- Hosting: Railway (API), Vercel (frontend)
- Key libraries: MediatR, FluentValidation, EF Core, Serilog

ARCHITECTURE: Clean Architecture (CardShop.API / .Core / .Infrastructure)
REPO: https://github.com/SultanMuqimi/SAAS

SESSIONS COMPLETED:
1. Scaffolding — Clean Architecture solution structure, project references
2. Database — PostgreSQL connection, EF Core setup, initial migrations
3. Auth — JWT authentication with refresh tokens, role-based authorization
4. Product catalog — Products CRUD, Redis caching, GCC seed data

CURRENT STATE:
Backend scaffolded with full Clean Architecture. Database migrations run.
JWT auth works end-to-end. Product catalog API returns cached results.
Frontend not started yet.

FEATURES BUILT:
- User registration and login
- Role-based access control (customer, admin)
- Product catalog with categories (game cards, gift cards, prepaid)
- Redis caching for product lists

FEATURES PLANNED BUT NOT BUILT:
- Checkout and order flow
- Payment integration (Tap or Stripe)
- User profile and order history
- Admin dashboard
- Frontend (not started)

NEXT SESSION SHOULD COVER:
Session 5 — Purchase flow backend. Create Order and OrderItem entities,
POST /api/orders endpoint, idempotency for order creation, status
lifecycle (pending → paid → delivered), basic order retrieval endpoints
for the current user.

KNOWN ISSUES:
- Email sending not configured
- No rate limiting on public endpoints yet
- Swagger UI works in dev but needs to be disabled in production config

DECISIONS MADE ALONG THE WAY:
- No try-catch in controllers — all exceptions handled by
  ExceptionHandlingMiddleware
- Permanent feature/backend branch for active work
- Claude GitHub Actions PR review workflow configured
- Products stored with base currency OMR, converted at display time

ENV VARIABLES SET:
DATABASE_URL, REDIS_URL, JWT_SECRET, JWT_REFRESH_SECRET, JWT_ISSUER,
JWT_AUDIENCE
```

---

## TIPS FOR A CLEAN RESUME

- **Ask for the summary before closing**, not days later when you've forgotten details
- **Save it somewhere you'll find it** — not buried in a chat you can't search
- **Update it if you do manual work** between sessions (e.g., you pushed to GitHub, changed a DB schema, rotated a key)
- **Keep old summaries** — they're a record of your build journey and useful if you ever need to explain the project to someone else

---

## FOR NON-DEVELOPERS

If you're not technical, don't worry about filling this out yourself. Just ask Forge:

> "I need to stop here. Give me the project status summary."

Forge will do all the work. Copy what it gives you, paste it into Notes or a Google Doc, and come back whenever you're ready.
