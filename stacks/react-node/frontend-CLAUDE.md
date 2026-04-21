# CLAUDE.md — React Frontend (Vite + TypeScript)
### Forge Stack Rules | React Frontend in React+Node Projects

---

This file extends the universal CLAUDE.md. Read that first. This file covers the React frontend in projects that use a separate Node.js backend.

---

## WHEN TO USE THIS STACK

- Business apps with a separate backend API
- Admin dashboards and internal tools
- SPAs that don't need SEO
- Projects where frontend and backend are deployed independently

---

## TECHNOLOGY STANDARDS

### Required
- **React 18+** with functional components and hooks only
- **Vite** as the build tool (never Create React App)
- **TypeScript** strict mode
- **Tailwind CSS** for styling
- **React Router v6+** for routing

### Required Libraries
- **TanStack Query (React Query)** for server state
- **Zustand** for client state (only when needed)
- **React Hook Form** + **Zod** for forms
- **Axios** or **Fetch** wrapper for API calls
- **shadcn/ui** for component primitives
- **Lucide React** for icons

---

## FOLDER STRUCTURE

```
frontend/
├── CLAUDE.md
├── .env.example
├── package.json
├── tsconfig.json
├── vite.config.ts
├── tailwind.config.ts
├── index.html
├── public/
└── src/
    ├── main.tsx
    ├── App.tsx
    ├── routes/                    (route definitions)
    ├── pages/                     (page components)
    ├── components/
    │   ├── ui/                    (shadcn components)
    │   └── features/              (feature-specific)
    ├── hooks/                     (custom hooks)
    ├── services/                  (API calls organized by feature)
    ├── stores/                    (Zustand stores)
    ├── lib/
    │   ├── api.ts                 (axios instance with interceptors)
    │   └── utils.ts
    ├── types/
    └── styles/
```

---

## COMPONENT RULES

- Functional components only — no class components
- One component per file
- Co-locate tests with components (`Button.tsx` + `Button.test.tsx`)
- Small components — if over 200 lines, split it
- Props are typed with TypeScript interfaces, never `any`

### Component Structure
```typescript
interface UserCardProps {
  user: User
  onEdit?: (user: User) => void
}

export function UserCard({ user, onEdit }: UserCardProps) {
  // hooks first
  const [isHovered, setIsHovered] = useState(false)
  
  // derived values
  const displayName = user.name || user.email
  
  // handlers
  const handleClick = () => {
    onEdit?.(user)
  }
  
  // render
  return (
    <div onClick={handleClick}>
      {displayName}
    </div>
  )
}
```

---

## STATE MANAGEMENT

### Server State — TanStack Query Only
- Never store server data in Zustand or useState
- Configure a single `QueryClient` at the app root
- Use `useQuery` for reads, `useMutation` for writes
- Set reasonable stale times and cache times
- Invalidate queries after mutations

### Client State — Hierarchy
1. **Local state** (`useState`) — component-level UI state
2. **URL state** — anything shareable/bookmarkable belongs in URL params
3. **Context** — for theme, auth session, rarely-changing global state
4. **Zustand** — for complex client state that spans many components

**Never put server data in client state stores.**

---

## API CALLS

### Centralized Client
```typescript
// src/lib/api.ts
import axios from 'axios'

export const api = axios.create({
  baseURL: import.meta.env.VITE_API_URL,
  timeout: 10000,
})

api.interceptors.request.use((config) => {
  const token = getAuthToken()
  if (token) config.headers.Authorization = `Bearer ${token}`
  return config
})

api.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      // handle logout
    }
    return Promise.reject(error)
  }
)
```

### Service Layer
- All API calls go through `src/services/[feature].ts`
- Components never call `api` directly — they call services
- Services are typed and handle response parsing

```typescript
// src/services/users.ts
import { api } from '@/lib/api'
import type { User } from '@/types'

export const usersService = {
  async list(): Promise<User[]> {
    const { data } = await api.get<User[]>('/users')
    return data
  },
  async get(id: string): Promise<User> {
    const { data } = await api.get<User>(`/users/${id}`)
    return data
  },
  async create(input: CreateUserInput): Promise<User> {
    const { data } = await api.post<User>('/users', input)
    return data
  },
}
```

---

## ROUTING

- Use React Router v6+ with `createBrowserRouter`
- Define routes declaratively in `src/routes/`
- Protected routes use a wrapper component, not manual checks in each page
- Lazy-load route components for code splitting

```typescript
const DashboardPage = lazy(() => import('@/pages/dashboard'))

const router = createBrowserRouter([
  {
    path: '/',
    element: <RootLayout />,
    children: [
      { index: true, element: <HomePage /> },
      {
        path: 'dashboard',
        element: <ProtectedRoute><DashboardPage /></ProtectedRoute>,
      },
    ],
  },
])
```

---

## FORMS

Same pattern as Next.js stack:
- React Hook Form + Zod
- Validation schemas defined separately from components
- Show errors inline
- Disable submit during submission
- Handle server errors gracefully

---

## PERFORMANCE

- Lazy-load routes with `React.lazy` and `Suspense`
- Memoize expensive calculations with `useMemo`
- Memoize callback props with `useCallback` when passed to memoized children
- Use `React.memo` only when profiling shows it helps — not preemptively
- Debounce search inputs (300ms default)
- Virtualize long lists (react-virtual or similar)

---

## STYLING

- Tailwind utility classes
- Use `cn()` utility for conditional classes:
```typescript
import { clsx, type ClassValue } from 'clsx'
import { twMerge } from 'tailwind-merge'

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs))
}
```
- Extract repeated combinations into components
- Mobile-first responsive design

---

## TESTING

- **Vitest** for unit tests
- **React Testing Library** for component tests
- **Playwright** for E2E tests
- Test behavior, not implementation details
- Queries preference order: `getByRole`, `getByLabelText`, `getByText`, `getByTestId` (last resort)

---

## ERROR HANDLING

- Use Error Boundaries at route level minimum
- Show user-friendly error messages
- Log errors to a monitoring service in production (Sentry, etc.)
- TanStack Query errors: handle in `onError` callback or via global error handler

---

## ENVIRONMENT VARIABLES

- All Vite env vars must start with `VITE_`
- Document every variable in `.env.example`
- Typical variables:
```
VITE_API_URL=http://localhost:3001
VITE_APP_NAME=
```

---

## SESSION COMPLETION CHECKLIST — REACT FRONTEND SPECIFIC

- [ ] `npm run build` completes successfully
- [ ] No TypeScript errors
- [ ] No ESLint errors
- [ ] No console errors or warnings in the browser
- [ ] All routes are properly protected
- [ ] API calls go through the service layer
- [ ] Loading and error states exist for all data-fetching views
- [ ] Forms validate correctly and show errors

---

*This file is part of Forge by Sultan Al-Muqimi / NQTH LLC.*
