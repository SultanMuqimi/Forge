# CLAUDE.md — Next.js Stack
### Forge Stack Rules | Next.js Full-Stack Projects

---

This file extends the universal CLAUDE.md at the project root. Read that first, then this. Everything in the universal file applies here too — this file adds Next.js-specific rules.

---

## WHEN TO USE THIS STACK

Next.js is the right choice for:
- Non-technical users building their first app (simplest setup)
- Full-stack apps where frontend and backend live together
- SaaS products that need SEO
- Marketing sites with dynamic features
- Apps hosted on Vercel (zero-config deployment)

**Default choice** for non-developers and junior developers unless the app has specific needs that require a different stack.

---

## TECHNOLOGY STANDARDS

### Required
- **Next.js 14+** (App Router, not Pages Router)
- **TypeScript** — strict mode, never plain JavaScript
- **PostgreSQL** via Supabase (for non-developers) or managed PostgreSQL (for developers)
- **Tailwind CSS** for styling
- **Prisma** or **Drizzle** as the ORM (Prisma for beginners, Drizzle for advanced)

### Recommended
- **React Hook Form** + **Zod** for forms and validation
- **shadcn/ui** for component primitives (not a dependency — copy components into the project)
- **Lucide React** for icons
- **date-fns** for date handling (not moment.js)

### Authentication Options (pick one)
- **NextAuth.js v5 (Auth.js)** — when building custom auth
- **Supabase Auth** — when using Supabase as the database (easiest for beginners)
- **Clerk** — when paying for managed auth is acceptable

### AI/State Management
- **Server state:** TanStack Query (React Query) or native Next.js data fetching
- **Client state:** Zustand for complex state, React Context for simple state
- Never use Redux unless the project is genuinely large and complex

---

## FOLDER STRUCTURE

Standard App Router structure:

```
project-root/
├── CLAUDE.md
├── .env.example
├── .env.local                     (gitignored)
├── package.json
├── tsconfig.json
├── tailwind.config.ts
├── next.config.js
├── prisma/
│   ├── schema.prisma
│   └── migrations/
├── public/
│   └── (static assets)
└── src/
    ├── app/                       (routes and pages)
    │   ├── layout.tsx
    │   ├── page.tsx
    │   ├── (auth)/                (route groups for layouts)
    │   │   ├── login/
    │   │   └── register/
    │   ├── (dashboard)/
    │   │   └── dashboard/
    │   └── api/                   (API route handlers)
    │       └── [feature]/
    │           └── route.ts
    ├── components/
    │   ├── ui/                    (shadcn components)
    │   └── features/              (feature-specific components)
    ├── lib/
    │   ├── db.ts                  (Prisma client singleton)
    │   ├── auth.ts                (auth configuration)
    │   └── utils.ts
    ├── hooks/                     (custom React hooks)
    ├── services/                  (business logic, not tied to UI)
    ├── types/                     (shared TypeScript types)
    └── middleware.ts              (auth middleware)
```

---

## APP ROUTER RULES

### Server Components by Default
- Every component is a Server Component unless it needs interactivity
- Only add `"use client"` when you need: useState, useEffect, event handlers, browser APIs
- Fetch data directly in Server Components with async/await — no useEffect for data fetching

### Data Fetching
```typescript
// Good — server component fetches directly
async function UsersPage() {
  const users = await db.user.findMany()
  return <UserList users={users} />
}

// Bad — using useEffect in client component when server would work
```

### API Routes
- Use route handlers in `app/api/[feature]/route.ts`
- Export HTTP method functions: `GET`, `POST`, `PUT`, `DELETE`
- Validate request bodies with Zod before processing
- Return `NextResponse.json()` with proper status codes

```typescript
import { NextResponse } from 'next/server'
import { z } from 'zod'

const createUserSchema = z.object({
  email: z.string().email(),
  name: z.string().min(1).max(100),
})

export async function POST(request: Request) {
  const body = await request.json()
  const result = createUserSchema.safeParse(body)
  
  if (!result.success) {
    return NextResponse.json(
      { error: 'Invalid input', details: result.error.flatten() },
      { status: 400 }
    )
  }
  
  const user = await userService.create(result.data)
  return NextResponse.json(user, { status: 201 })
}
```

### Server Actions
- Use Server Actions for form submissions and mutations when possible
- Mark them with `"use server"` directive
- Validate inputs with Zod inside the action
- Revalidate paths/tags after mutations

---

## DATABASE — PRISMA

### Schema Rules
- Every model has `id`, `createdAt`, `updatedAt`
- Use `cuid()` or `uuid()` for IDs — never integers for public-facing records
- Define relationships explicitly
- Use `@unique` constraints properly
- Index fields used in `where` clauses frequently

### Client Singleton
Always use a singleton pattern for the Prisma client:

```typescript
// src/lib/db.ts
import { PrismaClient } from '@prisma/client'

const globalForPrisma = global as unknown as { prisma: PrismaClient }

export const db = globalForPrisma.prisma || new PrismaClient()

if (process.env.NODE_ENV !== 'production') globalForPrisma.prisma = db
```

### Queries
- Use `select` to return only needed fields — never return whole rows to the client
- Use `include` sparingly — prefer separate queries for clarity when relationships are complex
- Handle `null` results explicitly — `findUnique` can return null

---

## AUTHENTICATION

### Middleware Protection
Protected routes use `middleware.ts`:

```typescript
import { auth } from '@/lib/auth'
import { NextResponse } from 'next/server'

export default auth((req) => {
  if (!req.auth && req.nextUrl.pathname.startsWith('/dashboard')) {
    return NextResponse.redirect(new URL('/login', req.url))
  }
})

export const config = {
  matcher: ['/dashboard/:path*', '/api/protected/:path*'],
}
```

### Session Access
- Server Components: use the auth helper directly
- Client Components: use the session hook
- API routes: validate auth at the top of every protected handler

---

## Frontend Stack & Design System

### Technology stack (use exactly these)
- Next.js 14+ App Router + TypeScript — base
- Tailwind CSS — all styling, zero custom CSS files
- shadcn/ui — component library
- Framer Motion — animations only, used sparingly
- TanStack Query (React Query v5) — client-side data fetching only (server components handle SSR fetching)
- Zustand — global client state (auth, cart, language)
- react-i18next — Arabic/English language switching with RTL/LTR support
- React Hook Form + Zod — all forms and validation
- Next.js Image component — all images, never raw `<img>` tags

Never install a second UI library alongside shadcn/ui.
Never install a second animation library alongside Framer Motion.

### Design system
- Primary: #1D4ED8, Accent: #F97316, Success: #10B981, Error: #EF4444
- Background: #FFFFFF / #F9FAFB, Text: #111827 / #6B7280
- Cards: white, subtle shadow, rounded-xl
- Fonts: Noto Sans Arabic (Arabic), Plus Jakarta Sans (headlines), Inter (body)
- Light mode only by default
- Full RTL/LTR support via react-i18next + Tailwind RTL plugin

### Performance rules
- Server Components for all data fetching — Client Components only when interactivity is required
- Lazy load Client Components with `dynamic()` and `{ ssr: false }` when needed
- Framer Motion only on: page transitions, card hover, cart additions, success states
- Never import entire icon libraries — import individually
- All images via Next.js Image with explicit width and height

### Frontend security
- Never store JWT access token in localStorage — memory only
- Server secrets in plain env vars, client-safe values prefixed with `NEXT_PUBLIC_`
- Never use `dangerouslySetInnerHTML` without DOMPurify sanitization

### shadcn/ui Components
- Copy components into `src/components/ui/` — do not install as a dependency
- Customize freely — they are your code now
- Use the CLI to add new components: `npx shadcn-ui@latest add button`

---

## FORMS

### Pattern
- React Hook Form for state management
- Zod for validation schemas
- `@hookform/resolvers/zod` to connect them

```typescript
const schema = z.object({
  email: z.string().email('Invalid email'),
  password: z.string().min(8, 'Password must be at least 8 characters'),
})

type FormData = z.infer<typeof schema>

function LoginForm() {
  const form = useForm<FormData>({
    resolver: zodResolver(schema),
  })
  
  // ...
}
```

- Show validation errors inline, not just on submit
- Disable submit button while submitting
- Show loading states clearly
- Handle server errors gracefully — do not crash on API failures

---

## PERFORMANCE

### Images
- Always use `next/image` — never raw `<img>` tags
- Set explicit `width` and `height` or use `fill` with a sized parent
- Use `priority` for above-the-fold images
- Use WebP or AVIF formats

### Fonts
- Use `next/font` — never import fonts via CSS imports
- Use variable fonts when available (smaller file size)
- Preload critical fonts

### Loading States
- Use `loading.tsx` in route folders for automatic loading UI
- Use `Suspense` boundaries for streaming
- Show skeleton screens, not spinners, for perceived performance

### Caching
- Use `fetch` with `{ next: { revalidate: N } }` for ISR
- Use `revalidatePath()` and `revalidateTag()` after mutations
- Tag fetches for granular invalidation

---

## ERROR HANDLING

### Error Boundaries
- Use `error.tsx` in route folders for error UI
- Use `global-error.tsx` for top-level errors
- Never show raw error messages to users — log them, show friendly messages

### Not Found
- Use `not-found.tsx` for custom 404 pages
- Call `notFound()` from Server Components when a resource is missing

---

## TESTING

### Stack
- **Vitest** for unit tests (faster than Jest for Next.js)
- **Playwright** for E2E tests
- **React Testing Library** for component tests

### What to Test
- Server Actions: unit tests for logic, integration tests for database interactions
- API routes: integration tests covering success and error cases
- Components with logic: unit tests
- Critical user flows: E2E tests (login, signup, main feature)

---

## DEPLOYMENT

### Vercel (Recommended for Non-Developers)
- Connect the GitHub repo to Vercel
- Environment variables configured in Vercel dashboard
- Automatic deployments on push to `main`
- Preview deployments for pull requests

### Environment Variables Required
At minimum every Next.js project needs:
```
DATABASE_URL=
NEXT_PUBLIC_APP_URL=
NEXTAUTH_SECRET=         (if using NextAuth)
NEXTAUTH_URL=            (if using NextAuth)
```

### Before Deploying
- Run `npm run build` locally and verify no errors
- Run tests and ensure all pass
- Verify all environment variables are set in the production environment
- Test with a production build (`npm run build && npm start`)

---

## SPECIFIC DO NOTS FOR NEXT.JS

- Never use the Pages Router for new projects — App Router only
- Never mix server and client state management
- Never fetch data in client components when a server component would work
- Never use `getServerSideProps` or `getStaticProps` (those are Pages Router)
- Never import server-only code into client components (use `"server-only"` package to enforce)
- Never use `<img>` tags — always `next/image`
- Never use `<a>` for internal links — always `next/link`
- Never install shadcn/ui as a dependency — copy components in

---

## SESSION COMPLETION CHECKLIST — NEXT.JS SPECIFIC

In addition to the universal checklist, verify:

- [ ] `npm run build` completes without errors
- [ ] No TypeScript errors (`npm run type-check` or equivalent)
- [ ] No ESLint errors (`npm run lint`)
- [ ] All Server Components are async where they need to be
- [ ] All Client Components have `"use client"` directive
- [ ] Middleware protects the right routes
- [ ] Environment variables documented in `.env.example`
- [ ] Database migrations are up to date (`npx prisma migrate dev`)

---

*This file is part of Forge by Sultan Al-Muqimi / NQTH LLC.*
