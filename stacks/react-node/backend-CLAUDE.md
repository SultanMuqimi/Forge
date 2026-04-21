# CLAUDE.md — Node.js + TypeScript Backend
### Forge Stack Rules | Node.js Backend in React+Node Projects

---

This file extends the universal CLAUDE.md. Read that first. This file covers the Node.js + TypeScript backend when paired with a React frontend.

---

## TECHNOLOGY STANDARDS

### Required
- **Node.js 20+ LTS**
- **TypeScript** strict mode
- **Fastify** (preferred) or **Express** as the web framework
- **PostgreSQL** as the database
- **Prisma** or **Drizzle** as the ORM
- **Zod** for validation

### Required Libraries
- **Pino** for logging (not Winston unless already chosen)
- **helmet** for security headers
- **cors** configured explicitly
- **express-rate-limit** or **@fastify/rate-limit**
- **bcryptjs** or **argon2** for passwords
- **jsonwebtoken** for JWT (or session library)

---

## WHY FASTIFY OVER EXPRESS

Fastify is the default choice because:
- Faster (2-3x throughput of Express)
- Built-in schema validation
- Better TypeScript support
- Plugin system for clean architecture
- Built-in logging with Pino

Use Express only if the team is more familiar with it or the project already uses it.

---

## FOLDER STRUCTURE

```
backend/
├── CLAUDE.md
├── .env.example
├── package.json
├── tsconfig.json
├── prisma/
│   ├── schema.prisma
│   └── migrations/
└── src/
    ├── server.ts                  (entry point)
    ├── app.ts                     (app configuration)
    ├── config/
    │   ├── env.ts                 (validated env vars)
    │   └── database.ts
    ├── modules/                   (organize by feature)
    │   └── users/
    │       ├── users.routes.ts
    │       ├── users.controller.ts
    │       ├── users.service.ts
    │       ├── users.repository.ts
    │       ├── users.schemas.ts   (Zod schemas)
    │       └── users.types.ts
    ├── middleware/
    │   ├── auth.middleware.ts
    │   ├── error-handler.ts
    │   └── rate-limit.ts
    ├── lib/
    │   ├── db.ts                  (Prisma client)
    │   ├── logger.ts
    │   └── errors.ts              (custom error classes)
    └── utils/
```

---

## LAYERED ARCHITECTURE

### Request Flow
```
Route → Controller → Service → Repository → Database
```

- **Routes** register endpoints and middleware
- **Controllers** handle HTTP concerns (parse request, call service, format response)
- **Services** contain business logic (no HTTP, no database directly)
- **Repositories** handle database access
- **Schemas** define validation with Zod

### Example
```typescript
// users.schemas.ts
export const createUserSchema = z.object({
  email: z.string().email(),
  name: z.string().min(1).max(100),
  password: z.string().min(8),
})
export type CreateUserInput = z.infer<typeof createUserSchema>

// users.repository.ts
export const usersRepository = {
  async create(data: CreateUserInput & { passwordHash: string }) {
    return db.user.create({
      data: {
        email: data.email,
        name: data.name,
        passwordHash: data.passwordHash,
      },
    })
  },
  async findByEmail(email: string) {
    return db.user.findUnique({ where: { email } })
  },
}

// users.service.ts
export const usersService = {
  async create(input: CreateUserInput) {
    const existing = await usersRepository.findByEmail(input.email)
    if (existing) throw new ConflictError('Email already registered')
    
    const passwordHash = await bcrypt.hash(input.password, 12)
    return usersRepository.create({ ...input, passwordHash })
  },
}

// users.controller.ts
export async function createUser(req: FastifyRequest, reply: FastifyReply) {
  const input = createUserSchema.parse(req.body)
  const user = await usersService.create(input)
  reply.status(201).send(toUserDTO(user))
}

// users.routes.ts
export async function usersRoutes(app: FastifyInstance) {
  app.post('/users', createUser)
}
```

---

## ERROR HANDLING — GLOBAL ONLY

**Never put try-catch in controllers or services unless you're recovering.** Let errors propagate to the global handler.

### Custom Error Classes
```typescript
// src/lib/errors.ts
export class AppError extends Error {
  constructor(public message: string, public statusCode: number) {
    super(message)
  }
}

export class NotFoundError extends AppError {
  constructor(message = 'Resource not found') {
    super(message, 404)
  }
}

export class ConflictError extends AppError {
  constructor(message = 'Conflict') {
    super(message, 409)
  }
}

export class UnauthorizedError extends AppError {
  constructor(message = 'Unauthorized') {
    super(message, 401)
  }
}

export class ValidationError extends AppError {
  constructor(message: string, public details?: unknown) {
    super(message, 400)
  }
}
```

### Global Error Handler
```typescript
// Fastify example
app.setErrorHandler((error, request, reply) => {
  if (error instanceof AppError) {
    return reply.status(error.statusCode).send({
      error: error.message,
      ...(error instanceof ValidationError && { details: error.details }),
    })
  }
  
  if (error instanceof ZodError) {
    return reply.status(400).send({
      error: 'Validation failed',
      details: error.flatten(),
    })
  }
  
  request.log.error(error)
  return reply.status(500).send({ error: 'Internal server error' })
})
```

---

## AUTHENTICATION

### JWT Pattern
- Access tokens: 15 minutes
- Refresh tokens: 7 days, stored in database (to allow revocation)
- HTTP-only cookies for web, Authorization header for mobile/external clients

### Middleware
```typescript
export async function authMiddleware(req: FastifyRequest, reply: FastifyReply) {
  const token = extractToken(req)
  if (!token) throw new UnauthorizedError()
  
  const payload = verifyAccessToken(token)
  req.user = await usersRepository.findById(payload.userId)
  
  if (!req.user) throw new UnauthorizedError('User not found')
}
```

- Always fetch fresh user data in the middleware, never trust token contents beyond identity
- Check token expiration properly
- Implement token refresh endpoint

---

## DATABASE

### Prisma Client Singleton
```typescript
// src/lib/db.ts
import { PrismaClient } from '@prisma/client'

export const db = new PrismaClient({
  log: process.env.NODE_ENV === 'development' ? ['query', 'error'] : ['error'],
})
```

### Migrations
- Create migrations: `npx prisma migrate dev --name description`
- Apply in production: `npx prisma migrate deploy`
- Never run `prisma db push` on production

### Transactions
```typescript
await db.$transaction(async (tx) => {
  const user = await tx.user.create({ data: userData })
  await tx.profile.create({ data: { userId: user.id, ...profileData } })
})
```

---

## VALIDATION

### Every Endpoint Validates
```typescript
const schema = z.object({
  body: createUserSchema,
  params: z.object({ id: z.string().uuid() }),
  query: z.object({ page: z.coerce.number().default(1) }),
})
```

- Validate body, query, and params separately
- Use `z.coerce` for query parameters (they arrive as strings)
- Never trust client-provided IDs without authorization checks

---

## LOGGING

### Pino Configuration
```typescript
import pino from 'pino'

export const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  transport: process.env.NODE_ENV === 'development'
    ? { target: 'pino-pretty' }
    : undefined,
  redact: ['req.headers.authorization', 'req.body.password'],
})
```

- Redact sensitive fields
- Include request IDs for tracing
- Log levels: `fatal`, `error`, `warn`, `info`, `debug`, `trace`

---

## SECURITY CHECKLIST

- [ ] Helmet configured with proper CSP
- [ ] CORS whitelist specific origins (no `*` in production)
- [ ] Rate limiting on all public endpoints (stricter on auth)
- [ ] Password hashing with bcrypt (cost 12) or argon2
- [ ] JWT secrets are strong (32+ random bytes) and in env vars
- [ ] Input validation on every endpoint
- [ ] Output sanitization — never return password hashes or internal IDs
- [ ] SQL injection prevented via Prisma (never raw SQL with user input)
- [ ] HTTPS enforced in production
- [ ] Security headers set
- [ ] Dependencies audited (`npm audit`)

---

## TESTING

### Stack
- **Vitest** or **Jest** for unit tests
- **Supertest** for HTTP integration tests
- Test database (separate from dev)

### What to Test
- Service layer: unit tests for business logic
- Repository layer: integration tests against test DB
- Routes: integration tests covering 200s, 400s, 401s, 403s, 404s, 500s
- Middleware: unit tests

### Test Data
- Use factories to create test data
- Reset database between tests (truncate or transaction rollback)
- Never use production data in tests

---

## ENVIRONMENT VARIABLES

Required at minimum:
```
NODE_ENV=development
PORT=3001
DATABASE_URL=postgresql://user:pass@localhost:5432/dbname
JWT_ACCESS_SECRET=<32+ random bytes>
JWT_REFRESH_SECRET=<32+ random bytes>
CORS_ORIGIN=http://localhost:5173
LOG_LEVEL=info
```

### Validation on Startup
Validate env vars with Zod at startup — fail fast if misconfigured:

```typescript
// src/config/env.ts
const envSchema = z.object({
  NODE_ENV: z.enum(['development', 'staging', 'production']),
  PORT: z.coerce.number().default(3001),
  DATABASE_URL: z.string().url(),
  JWT_ACCESS_SECRET: z.string().min(32),
  JWT_REFRESH_SECRET: z.string().min(32),
  CORS_ORIGIN: z.string(),
})

export const env = envSchema.parse(process.env)
```

---

## SPECIFIC DO NOTS

- Never use callbacks — async/await only
- Never use `var` — only `const` and `let`
- Never use `any` in TypeScript without documenting why
- Never put business logic in controllers
- Never put database queries in services
- Never return raw Prisma results to clients — map to DTOs
- Never use synchronous file operations in request handlers
- Never import database models into controllers directly

---

## SESSION COMPLETION CHECKLIST — NODE.JS BACKEND SPECIFIC

- [ ] `npm run build` succeeds (TypeScript compiles)
- [ ] All endpoints validated with Zod
- [ ] All protected endpoints have auth middleware
- [ ] Rate limiting applied
- [ ] Error handler registered
- [ ] Logger configured with redaction
- [ ] `/health` endpoint exists and checks database
- [ ] Environment variables documented in `.env.example`
- [ ] Migrations are current
- [ ] Tests pass

---

*This file is part of Forge by Sultan Al-Muqimi / NQTH LLC.*
