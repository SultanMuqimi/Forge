# Forge — Stack Reference

This document explains **why** Forge picks the stacks it does. Read this if you want to understand the thinking, or you're considering contributing a new stack.

Forge is opinionated on purpose. Most developers don't need twelve frameworks to choose from — they need the right one for their situation, picked by someone who's already thought about the tradeoffs.

---

## CORE PRINCIPLES

1. **Boring beats clever.** We pick technologies with large communities, stable APIs, and known failure modes. If a new framework breaks weekly, it's not ready for production use.
2. **Match complexity to scale.** A solo founder doesn't need Kubernetes. A Fortune 500 doesn't need a no-code tool. The stack fits the situation.
3. **TypeScript over JavaScript.** Always. Runtime errors that could have been caught at compile time are a waste of your life.
4. **PostgreSQL unless there's a clear reason not to.** It's the most capable, most reliable, most extensible relational database on earth.
5. **Monolith by default.** Microservices are an organizational tool, not a technical one. Don't adopt them unless your org demands it.

---

## THE STACKS

### 1. Next.js (Full-Stack)

**When Forge picks this:**
- Non-developers and juniors building web apps
- SaaS products where speed to market matters
- Content-heavy apps that benefit from SSR/SSG (blogs, marketing sites, e-commerce)
- Teams that want one codebase for frontend + backend

**Why:**
- One codebase, one deployment, one mental model — ideal for solo founders and small teams
- Vercel deployment is effectively one-click
- App Router + Server Components handle most data-fetching without a separate backend
- The ecosystem is massive — auth, payments, email, analytics all have first-party Next.js guides

**Tradeoffs:**
- Vendor-friendly to Vercel — self-hosting works but loses the polish
- Not ideal for AI backends that need long-running Python processes (use FastAPI instead)
- Serverless cold starts can hurt latency-sensitive apps

**Rule file:** `stacks/nextjs/CLAUDE.md`

---

### 2. React + Node.js (Separate Frontend + Backend)

**When Forge picks this:**
- Business web apps with meaningful backend logic
- Teams that want the frontend and backend to evolve independently
- Apps where you'll eventually want a mobile client sharing the same API
- Developers comfortable managing two codebases or monorepo

**Why:**
- Clear separation of concerns — frontend team and backend team can move independently
- Fastify is blazingly fast, modern, and has excellent TypeScript support
- React + Vite is the leanest, fastest frontend dev experience available
- The API can serve web, mobile, and third parties without refactoring

**Tradeoffs:**
- Two deployments, two things to maintain
- More decisions to make — CORS, API versioning, client SDK generation
- Slower initial setup than Next.js

**Rule files:** `stacks/react-node/frontend-CLAUDE.md`, `stacks/react-node/backend-CLAUDE.md`

---

### 3. FastAPI + React (AI-Powered Apps)

**When Forge picks this:**
- Any app with AI features — always
- Apps that need Python's ML/data ecosystem (NumPy, Pandas, scikit-learn, transformers)
- APIs that need async Python (FastAPI handles this beautifully)

**Why:**
- Python is the native language of AI — every provider SDK is Python-first
- FastAPI is the fastest, cleanest way to build async Python APIs — on par with Node.js performance
- Pydantic v2 gives you automatic validation, serialization, and OpenAPI docs
- The AI tooling gap between Python and Node.js is real — don't fight it

**Non-negotiable rule:** AI-powered apps always use Python FastAPI as the backend. No exceptions. Even if a junior developer only knows JavaScript, this is the one place Forge doesn't bend.

**Tradeoffs:**
- Python deployment is slightly more involved than Node.js
- Two languages in the codebase (Python backend, TypeScript frontend)
- Type sharing between backend and frontend requires OpenAPI code generation

**Rule files:** `stacks/fastapi-react/backend-CLAUDE.md`, `stacks/fastapi-react/frontend-CLAUDE.md`, `addons/ai-features.md`

---

### 4. ASP.NET Core + React (Enterprise / Microsoft Ecosystem)

**When Forge picks this:**
- Enterprise apps, government, regulated industries
- Organizations already on the Microsoft stack (Azure, Active Directory, SQL Server)
- Teams with C# / .NET expertise
- Apps where Clean Architecture and long-term maintainability matter more than ship-fast

**Why:**
- .NET 8+ is fast, stable, and well-supported by Microsoft for decades
- Clean Architecture with MediatR and CQRS is well-documented and proven at scale
- First-class Azure integration, Active Directory, SQL Server, Service Bus
- Exceptional tooling — Visual Studio, Rider, strong debuggers, strong profilers

**Tradeoffs:**
- More boilerplate than Node.js or Python
- Overkill for simple apps — don't pick this for a weekend project
- Heavier deployment footprint

**Rule files:** `stacks/aspnet-react/backend-CLAUDE.md`, `stacks/aspnet-react/frontend-CLAUDE.md`

---

### 5. Flutter (Mobile)

**When Forge picks this:**
- Mobile-first or mobile-only apps
- Apps that need to ship to both iOS and Android with one codebase
- Teams that want native-quality UX without learning Swift and Kotlin separately

**Why:**
- Single codebase for iOS, Android, and (increasingly) web and desktop
- Genuinely native performance — not a webview wrapper
- Dart is easy to learn for anyone with JavaScript, C#, or Java background
- Riverpod gives clean, testable state management

**Tradeoffs:**
- Smaller talent pool than React Native
- Dart isn't used elsewhere — so your backend is in a different language
- Some platform-specific features still require native code

**Rule file:** `stacks/flutter/CLAUDE.md`

---

### 6. Tauri + React (Lightweight Desktop)

**When Forge picks this:**
- Desktop apps that need to be small, fast, and secure
- Utilities, productivity tools, local data apps
- Any desktop app where Electron feels like overkill

**Why:**
- App bundles are 3–10 MB — versus 100+ MB for Electron
- Uses the OS's native webview — no bundled Chromium, lower memory
- Rust backend gives real performance and security
- Strict security model by default (allowlist-based access to OS APIs)

**Tradeoffs:**
- Webview differences between platforms occasionally cause rendering quirks
- Smaller ecosystem than Electron
- Rust has a learning curve if you need to extend the backend heavily

**Rule file:** `stacks/tauri-react/CLAUDE.md`

---

### 7. Electron + React (Heavy Desktop)

**When Forge picks this:**
- Desktop apps that need deep OS integration (native menus, system tray, file system access, shell commands)
- Apps bundling Node.js for complex local processing
- Teams already expert in Node.js who want to reuse that knowledge

**Why:**
- The most mature cross-platform desktop framework
- Full Node.js runtime available in the main process
- Massive ecosystem, battle-tested by VS Code, Slack, Discord, Figma

**Tradeoffs:**
- Large bundle sizes (100+ MB)
- Higher memory usage
- Security requires careful configuration (contextIsolation, nodeIntegration off, CSP)

**Rule file:** `stacks/electron-react/CLAUDE.md`

---

## WHAT WE DON'T USE (AND WHY)

### Banned permanently

- **PHP / Laravel / WordPress** — legacy-shaped stacks with ecosystem baggage. Modern alternatives are simply better for new projects. Forge will not generate PHP code under any circumstance, even on request.

### Not picked by default

These aren't banned — seniors can request them and Forge will respect the choice. But they're not the default because:

- **Go** — excellent for systems/networking, narrower ecosystem for CRUD/SaaS
- **Rust (backend)** — incredible, but overkill for most business apps and slower to write
- **Elixir / Phoenix** — brilliant, but smaller talent pool
- **Svelte / SvelteKit** — fantastic DX, but smaller ecosystem than React for enterprise apps
- **Vue** — solid, but Forge standardizes on React for ecosystem consistency
- **Angular** — heavy and opinionated in ways that conflict with Forge's philosophy
- **Ruby on Rails** — great, but we're betting on TypeScript-everywhere for long-term productivity
- **Django** — good, but we prefer FastAPI for modern async Python and automatic OpenAPI
- **NestJS** — fine, but adds class-based ceremony on top of Node.js that Fastify avoids
- **React Native** — Flutter is our mobile default; React Native is acceptable for teams already invested in it

---

## DATABASE CHOICES

### Default: PostgreSQL

Picked for nearly every situation. Why:
- Mature, reliable, SQL-standard compliant
- Extensions cover what you need: pgvector (AI), PostGIS (geo), pg_cron (jobs), full-text search
- Works at scale from 10 users to 10 million
- Available as managed service everywhere (Supabase, Railway, RDS, Cloud SQL, Azure Database)

### When we deviate

- **SQLite** — desktop apps (Tauri, Electron), mobile local storage, development/testing
- **MongoDB** — almost never. Forge rarely recommends it. Pick only if data is genuinely document-shaped and schema varies wildly per record.
- **Redis** — never as primary storage. Always as a cache, queue, or pub/sub layer alongside PostgreSQL.
- **SQL Server** — when the customer is already on Microsoft stack and mandates it.

---

## HOSTING DEFAULTS

| Stack | Hosting |
|---|---|
| Next.js | Vercel |
| React + Node.js | Vercel (frontend) + Railway (backend) |
| FastAPI + React | Vercel (frontend) + Railway or Fly.io (backend) |
| ASP.NET Core | Azure App Service (enterprise) |
| Flutter | App Store + Play Store |
| Tauri / Electron | GitHub Releases + auto-updater |

We avoid raw VPS deployment and Kubernetes for non-developers — the managed platforms are cheap, reliable, and remove an entire class of sysadmin problems.

---

## STACK CHANGES — WHEN TO ACCEPT

Forge updates this list as the ecosystem evolves. Criteria for adding a new stack:

1. Has at least 3 years of production use in serious applications
2. Has a clear, maintained path to production deployment
3. Has a large enough community that hiring is possible
4. Has a clear reason to exist that none of the current stacks serve well

Criteria for removing a stack:

1. Core maintainers abandon it
2. A clearly better alternative in the same niche has emerged
3. Security issues that aren't being fixed promptly

Open a PR or issue in the Forge repo if you want to propose a change.
