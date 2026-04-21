# CLAUDE.md — React Frontend (FastAPI + React)
### Forge Stack Rules | React Frontend with Python FastAPI Backend

---

This file extends the universal CLAUDE.md. Read that first.

Read this file fully before writing any frontend code.

---

## TECHNOLOGY STACK — use exactly these, nothing else

- React 18 + TypeScript + Vite — base framework, never Create React App
- Tailwind CSS — all styling, zero custom CSS files
- shadcn/ui — component library (components copied into project, no runtime bloat)
- Framer Motion — animations only, used sparingly (see Performance rules)
- TanStack Query (React Query v5) — all server state, caching, and API calls
- Zustand — global client state only (auth user, cart, language)
- React Router v6 — routing with lazy-loaded pages
- Axios — HTTP client with JWT interceptors
- react-i18next — Arabic/English language switching with full RTL/LTR support
- React Hook Form + Zod — all forms and validation, no uncontrolled inputs

Never install a second UI library alongside shadcn/ui.
Never install a second animation library alongside Framer Motion.
Never install Redux — Zustand is the state manager.

## DESIGN SYSTEM

### Colors (default palette — override per project if branded)
- Primary: #1D4ED8 (deep blue)
- Accent: #F97316 (vibrant orange for CTAs, highlights, badges)
- Success: #10B981 (emerald)
- Error: #EF4444 (red)
- Background: #FFFFFF and #F9FAFB
- Text primary: #111827
- Text muted: #6B7280
- Cards: white background, subtle shadow, rounded-xl corners

### Typography
- Arabic: Noto Sans Arabic
- English headlines: Plus Jakarta Sans
- Body: Inter
- Bilingual pattern when needed: Arabic headline above, English subtitle below
- Never mix more than two font families in the same view

### Mode
- Light mode only by default — no dark mode unless explicitly requested
- Feel: vibrant, alive, trustworthy — retail energy, not developer tooling
- Reference aesthetic: modern fintech meets premium retail

### RTL / LTR
- Full RTL support via react-i18next + Tailwind RTL plugin
- Language switcher available on every page when app targets Arabic users
- Never hardcode text direction — always derive from i18n context
- All spacing, padding, and flex direction must respect RTL automatically via Tailwind logical properties (ms-*, me-*, ps-*, pe-*)

## PERFORMANCE RULES

- Lazy load every route with React.lazy() and Suspense
- Never import an entire icon library — import icons individually
- Images always use loading="lazy" with explicit width and height
- Framer Motion allowed only on: page transitions, card hover, cart additions, success states — nowhere else
- TanStack Query handles ALL server state — never useState for API data
- Zustand stores ONLY: authenticated user, cart items, current language
- Never fetch data directly in a component — always through a custom hook using React Query

## FOLDER STRUCTURE
```
client/
└── src/
    ├── components/
    │   ├── ui/          ← shadcn/ui base components only
    │   └── shared/      ← reusable across features
    ├── features/        ← one folder per feature
    │   └── [feature]/
    │       ├── components/
    │       ├── hooks/
    │       ├── api/
    │       └── types/
    ├── hooks/           ← global custom hooks
    ├── lib/             ← axios instance, utils, i18n config
    ├── pages/           ← route-level components, lazy-loaded
    ├── store/           ← Zustand stores
    └── types/           ← global TypeScript types
```

## CODE RULES

- TypeScript strict mode — no .jsx files, ever
- Props always typed with `interface`, never inline object types
- No component longer than 150 lines — split if it grows beyond that
- No try-catch in components — errors handled by React Query error states and a global error boundary
- All API calls live in `feature/api/` using React Query hooks
- Base URL always from environment variable `VITE_API_URL` — never hardcoded
- JWT token attached via Axios interceptor — never added manually per request
- On 401 response: automatically attempt token refresh, redirect to login if refresh fails
- All forms use React Hook Form with Zod schema validation — no uncontrolled inputs
- All user-facing strings go through react-i18next — never hardcode Arabic or English text

## SECURITY

- Never store JWT access token in localStorage — use memory (Zustand) for access token, httpOnly cookie for refresh token
- Never log tokens, user data, or API responses to console in production
- Sanitize all user input before display — never use dangerouslySetInnerHTML unless explicitly required and sanitized (use DOMPurify)
- All environment variables prefixed with `VITE_` — never expose server secrets in the frontend build
- Content Security Policy headers set on the hosting platform
- HTTPS only in production — no mixed content

## TESTING

- Unit test every custom hook with Vitest + Testing Library
- Test form validation logic separately from the component
- Mock API calls with MSW (Mock Service Worker) — never mock Axios directly
- Test RTL layout separately from LTR — direction switching must be verified
- Minimum: one happy-path test per page, one validation test per form

## SESSION COMPLETION CHECKLIST

Before ending any frontend session:
- [ ] `npm run build` succeeds with zero TypeScript errors
- [ ] `npm run lint` passes
- [ ] All new routes are lazy-loaded
- [ ] All user-facing strings go through i18n
- [ ] RTL layout verified in Arabic mode
- [ ] No console.log statements left in code
- [ ] README updated with any new environment variables

---

## FASTAPI SPECIFIC OVERRIDES

### API Client Typing
Since the backend is Python, TypeScript types are not shared natively. Recommended approach:

- FastAPI auto-generates OpenAPI spec at `/openapi.json`
- Generate types with: `npx openapi-typescript http://localhost:8000/openapi.json -o src/types/api.ts`
- Add to `package.json`:
```json
{
  "scripts": {
    "generate-api-types": "openapi-typescript ${VITE_API_URL}/openapi.json -o src/types/api.ts"
  }
}
```
- Regenerate after every backend Pydantic model change

### Environment Variables
```
VITE_API_URL=http://localhost:8000
```

### CORS
FastAPI handles CORS on the backend — the frontend dev URL must be whitelisted there.

### Workflow when backend changes
1. Backend updates Pydantic models
2. Frontend runs `npm run generate-api-types`
3. TypeScript surfaces any breaking changes
4. Fix frontend code to match

---

*This file is part of Forge by Sultan Al-Muqimi / NQTH LLC.*
