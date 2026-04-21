# CLAUDE.md — Universal Rules
### Forge Quality Standards | Applies to Every Project

---

## YOUR ROLE

You are Claude Code working on a project generated via Forge. This file defines the universal standards for how you write code, regardless of the stack. Read this carefully and follow every rule. If a stack-specific CLAUDE.md also exists in this project, those rules combine with these — never override them.

**Before you write a single line of code, read this entire file.**

---

## CORE PRINCIPLES

1. **Clarity over cleverness.** Readable code beats clever code. Always.
2. **Boring is good.** Use proven patterns. Avoid experimental approaches unless there is a real reason.
3. **Match complexity to scale.** A simple app does not need microservices. A prototype does not need Kubernetes. Build what fits.
4. **If you are unsure, ask the user.** Do not guess on business logic.
5. **Finish what you start.** Do not leave half-implemented functions. Do not leave TODO comments for things you could do now.

---

## SECURITY — NON-NEGOTIABLE

These rules apply to every project, every time, no exceptions.

### Secrets and Configuration
- **Never hardcode secrets.** API keys, passwords, tokens, database URLs all go in environment variables
- Create a `.env.example` file listing every required environment variable with placeholder values
- Add `.env` and `.env.local` to `.gitignore` immediately
- Never commit real credentials, even briefly

### Authentication
- Passwords must be hashed with bcrypt (cost factor 12+) or argon2 — never plain text, never MD5, never SHA1
- Use JWT with short expiration (15 minutes for access tokens, 7 days for refresh tokens) or server-side sessions
- Always validate tokens on every protected request
- Implement proper logout that invalidates tokens/sessions server-side

### Input Validation
- Every API endpoint validates its inputs before doing anything else
- Use a validation library (Zod, Joi, Pydantic, FluentValidation, etc.) — never trust raw input
- Validate types, ranges, formats, and lengths
- Reject unknown fields, do not silently accept them

### SQL Injection Prevention
- Never concatenate user input into SQL strings
- Use parametrized queries or an ORM for all database access
- Raw SQL is allowed only when the ORM cannot express the query — and only with parameters

### XSS Prevention
- Frontend: escape all user-generated content before rendering
- Never use `dangerouslySetInnerHTML` or equivalent unless content is sanitized with a proven library (DOMPurify)
- Set proper Content-Security-Policy headers in production

### CORS and Headers
- Configure CORS explicitly — never use `*` in production
- Set security headers: `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Strict-Transport-Security`
- Use HTTPS-only cookies in production (`Secure`, `HttpOnly`, `SameSite=Strict` or `Lax`)

### Rate Limiting
- Every public endpoint has rate limiting
- Authentication endpoints have stricter limits (5 attempts per 15 minutes per IP)
- Use a proven library — never build rate limiting from scratch

### Data Exposure
- Never return password hashes, tokens, or internal IDs to the frontend unless absolutely needed
- Use response DTOs/schemas — never return raw database entities
- Log sensitive operations (login attempts, password changes, permission changes) without logging the secret values themselves

---

## PERFORMANCE — ALWAYS CONSIDER

### Database
- Every foreign key has an index
- Every frequently-queried column has an index
- Avoid N+1 queries — use joins, includes, or batch loading
- Use database transactions for multi-step operations that must succeed or fail together
- Connection pooling is configured correctly (default to 10-20 connections for small apps)

### Caching
- Cache expensive computations (Redis, in-memory for single-instance apps)
- Cache static assets with proper HTTP headers
- Use stale-while-revalidate patterns for data that can tolerate slight staleness
- Never cache user-specific data in shared caches

### API Design
- Paginate lists — never return unbounded collections
- Default page size: 20. Maximum: 100
- Use cursor-based pagination for large datasets, offset-based for small ones
- Compress responses (gzip/brotli) in production

### Frontend
- Lazy-load routes and heavy components
- Optimize images (WebP/AVIF, proper sizing, lazy loading)
- Debounce search inputs and expensive operations
- Use React Query/SWR or equivalent for server state — never refetch the same data repeatedly
- Minimize bundle size — tree-shake, split code, avoid heavy libraries when light alternatives exist

### Async Operations
- Never block the event loop
- Use proper async/await — never forget `await`
- Background jobs for anything over 2 seconds (queue systems like BullMQ, Celery, Hangfire)

---

## RELIABILITY — BUILD THINGS THAT DON'T BREAK

### Error Handling
- Use global error handling middleware — never try-catch in every controller or route handler
- Log errors with context (user ID, request ID, timestamp, stack trace) — never just log the message
- Return proper HTTP status codes (400 for bad input, 401 for unauthenticated, 403 for unauthorized, 404 for not found, 422 for validation, 500 for server errors)
- Never leak internal error details to users in production — show friendly messages, log technical details

### Logging
- Use a real logger (Winston, Pino, Serilog, Python logging) — never `console.log` in production
- Log levels: ERROR, WARN, INFO, DEBUG — use them correctly
- Include request IDs to trace requests across services
- Never log passwords, tokens, or sensitive personal data

### Health Checks
- Every backend has a `/health` endpoint that returns 200 when healthy
- Include database connectivity check
- Include external dependency checks (Redis, third-party APIs) when relevant

### Retries and Timeouts
- Every external API call has a timeout (default 10 seconds)
- Retry transient failures with exponential backoff (3 attempts maximum)
- Circuit breakers for critical external dependencies in production apps

### Graceful Degradation
- If a non-critical feature fails, the rest of the app still works
- Handle missing data gracefully — empty states, not crashes
- Frontend: loading states, error states, empty states for every data-fetching component

---

## TESTING — REQUIRED

### What to Test
- Every business logic function has unit tests
- Every API endpoint has at least one integration test (happy path + one error case minimum)
- Critical user flows have end-to-end tests
- Aim for meaningful coverage, not arbitrary percentages

### How to Test
- Tests are isolated — one test's output does not affect another
- Use a separate test database, not the dev or production one
- Mock external services — never hit real APIs in tests
- Tests run fast — a test suite over 30 seconds needs optimization

### When to Test
- Write tests in the same session you write the code
- Every bug fix adds a test that prevents regression
- Never skip tests to "move faster" — you will pay for it later

---

## CODE STRUCTURE

### File Organization
- Follow the folder structure defined in the Forge-generated prompt exactly
- Group by feature, not by file type — keep related code together
- Each file has one clear responsibility

### Naming Conventions
- **Variables and functions:** camelCase (JS/TS) or snake_case (Python) — follow the language's convention
- **Classes and components:** PascalCase
- **Files:** kebab-case for non-component files, PascalCase for React components
- **Constants:** UPPER_SNAKE_CASE
- **Database tables:** snake_case, plural (e.g., `users`, `order_items`)
- **Environment variables:** UPPER_SNAKE_CASE (e.g., `DATABASE_URL`)
- Names describe what the thing does, not what it is. `getUserById` not `userFunction`

### Function Rules
- Functions do one thing
- Functions are short (under 50 lines ideally, never over 100)
- Maximum 4 parameters — more than that, use an options object
- Pure functions when possible — side effects are explicit

### No Dead Code
- No commented-out code
- No unused imports
- No unused variables
- No unreachable code
- Remove it or explain why it stays (as a proper comment)

---

## DATABASE

### Schema Design
- Every table has a primary key (UUID or auto-increment integer)
- Every table has `created_at` and `updated_at` timestamps
- Use foreign keys with proper `ON DELETE` and `ON UPDATE` actions
- Soft delete (`deleted_at` column) only when business requires it — otherwise hard delete is simpler
- Use proper data types — don't store everything as strings

### Migrations
- Every schema change is a migration
- Migrations are reversible when possible
- Never modify historical migrations after they are run in production
- Test migrations on a copy of production data before running in production

### Access Patterns
- Access database through a repository pattern or ORM — no raw SQL in business logic
- Business logic never imports the database driver directly
- Transactions for operations that modify multiple tables

---

## VERSION CONTROL

### Commits
- Commit messages describe what changed and why
- Use conventional commits when possible (`feat:`, `fix:`, `docs:`, `refactor:`, `test:`)
- Small, focused commits over giant ones
- Never commit broken code to the main branch

### .gitignore
Every project's `.gitignore` includes at minimum:
- `node_modules/`, `__pycache__/`, `bin/`, `obj/`, `target/` (language-specific build artifacts)
- `.env`, `.env.local`, `.env.*.local`
- `dist/`, `build/`, `.next/`
- IDE files: `.vscode/settings.json` (but keep `.vscode/extensions.json`), `.idea/`
- OS files: `.DS_Store`, `Thumbs.db`
- Logs: `*.log`, `npm-debug.log*`, `yarn-debug.log*`

### Branches
- `main` or `master` is always deployable
- Feature branches for new work
- Never force-push to shared branches

---

## DOCUMENTATION

### README.md — Required for Every Project
Every project has a README.md with these sections minimum:
1. **Project name and one-line description**
2. **What it does** (2-4 sentences)
3. **Tech stack** (list of main technologies)
4. **Prerequisites** (Node version, Python version, database, etc.)
5. **Local setup** (step-by-step commands to run locally)
6. **Environment variables** (list every required variable with description)
7. **How to run** (dev command, test command, build command)
8. **Project structure** (brief overview of folders)

### Code Comments
- Comment **why**, not **what**. The code shows what it does.
- Complex algorithms get a brief explanation
- Non-obvious business logic gets context
- No comments for obvious things (`// increment counter` above `counter++`)

---

## DEPENDENCIES

### Choosing Libraries
- Prefer well-maintained libraries (recent commits, high download counts, active issues being resolved)
- Prefer smaller focused libraries over massive "kitchen sink" ones
- Check license compatibility (MIT, Apache 2.0, BSD are safe — avoid GPL for commercial projects)
- Check the library's security track record

### Version Pinning
- Pin dependency versions in lockfiles (package-lock.json, yarn.lock, poetry.lock, etc.)
- Commit lockfiles to the repo
- Use `^` for minor updates, `~` for patch updates in package.json — never `*` or no version

---

## ENVIRONMENT CONFIGURATION

### Three Environments Minimum
- **Development** — local machine, relaxed security, verbose logging
- **Staging/Preview** — production-like, for testing before release
- **Production** — strict security, optimized, minimal logging

### What Changes Between Environments
- Database URL
- API keys (dev keys vs production keys)
- Log levels (DEBUG in dev, INFO or WARN in production)
- Feature flags
- Rate limits (looser in dev)

### What Never Changes
- Core application logic
- Validation rules
- Security rules

---

## ACCESSIBILITY — FOR ANY USER INTERFACE

- Every form input has a label (not just a placeholder)
- Every image has meaningful alt text (or `alt=""` for decorative)
- Keyboard navigation works — tab through every interactive element
- Sufficient color contrast (WCAG AA minimum — 4.5:1 for normal text)
- Focus states are visible
- Semantic HTML (use `<button>` for buttons, `<a>` for links, proper heading hierarchy)

---

## WHAT NEVER TO DO

- Never use `eval()` or equivalent string-to-code functions
- Never trust data from the client
- Never use outdated crypto (MD5, SHA1 for passwords, ECB mode)
- Never log sensitive data
- Never commit secrets
- Never skip input validation "just this once"
- Never build custom auth when a proven library exists
- Never build custom crypto
- Never disable TypeScript strict mode
- Never use `any` type in TypeScript unless there is no alternative (and document why)
- Never silence errors without handling them
- Never use PHP, Laravel, or WordPress — this stack is permanently banned in Forge projects

---

## SESSION COMPLETION CHECKLIST

At the end of every Claude Code session, before telling the user it's done, verify:

- [ ] The code compiles/runs without errors
- [ ] All new code follows the rules above
- [ ] Tests are written for new logic
- [ ] Tests pass
- [ ] README.md is updated if setup steps changed
- [ ] `.env.example` is updated if new environment variables were added
- [ ] No secrets are in the code
- [ ] No `console.log`, `print()`, or debug statements left behind
- [ ] Error handling is in place
- [ ] The user can actually run what you built

If any item is not checked, fix it before ending the session.

---

## SUMMARY FOR THE END OF EVERY SESSION

At the end of a session, provide the user:
1. **What was built** (clear bullet list)
2. **How to run it** (exact commands)
3. **What to test** (what the user should verify works)
4. **Any new environment variables** they need to set
5. **What the next logical session should cover**

---

*This file is part of Forge by Sultan Al-Muqimi / NQTH LLC. Last updated with Forge v1.4.*
