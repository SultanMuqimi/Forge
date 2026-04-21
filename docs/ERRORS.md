# Forge — Common Errors & Fixes

When Claude Code runs into trouble mid-build, this is the first place to look. Most errors fall into a small number of recurring patterns. Find your symptom, apply the fix, tell Forge what happened, continue building.

---

## HOW TO USE THIS FILE

1. Find the section that matches your symptom
2. Try the fix
3. If it doesn't work, copy the **exact error message** back to Forge
4. Forge will generate a focused fix prompt for Claude Code

Forge never moves to the next session until the current error is resolved. That rule exists to protect you from a cascading pile of broken code.

---

## SETUP & INSTALLATION ERRORS

### `claude: command not found` / `The term 'claude' is not recognized`

**Cause:** Claude Code isn't installed, or it's installed but not on your PATH.

**Fix:**
1. Run `node --version` — if you don't see a version, install Node.js LTS from https://nodejs.org first
2. Run `npm install -g @anthropic-ai/claude-code`
3. Close and reopen your terminal (this reloads PATH)
4. Run `claude --version` to confirm

If still failing on Windows: check `echo %PATH%` includes the npm global bin folder (usually `%AppData%\npm`).
If still failing on Mac/Linux: check `echo $PATH` includes `/usr/local/bin` or your npm prefix (`npm config get prefix`).

---

### `EACCES: permission denied` (Linux/Mac, on global npm install)

**Cause:** npm trying to write to a protected system folder.

**Fix:** Do not use `sudo npm install`. Instead, configure npm to install globals in your home folder:

```bash
mkdir -p ~/.npm-global
npm config set prefix '~/.npm-global'
echo 'export PATH=~/.npm-global/bin:$PATH' >> ~/.bashrc  # or ~/.zshrc
source ~/.bashrc
npm install -g @anthropic-ai/claude-code
```

---

### Claude Code says "not authenticated" or keeps asking to log in

**Cause:** Session expired or token was never stored.

**Fix:** Run `claude` and follow the login flow again. If it loops, delete the config folder and retry:
- Windows: `%USERPROFILE%\.claude`
- Mac/Linux: `~/.claude`

---

## RULE FILE FETCHING ERRORS

### Claude Code can't reach GitHub raw URLs

**Cause:** Network firewall, VPN restriction, or GitHub temporarily unreachable.

**Fix:** Check connectivity:
```bash
curl -I https://raw.githubusercontent.com
```
If this fails, you're on a restricted network. Options:
- Download the CLAUDE.md files manually from the Forge repo
- Place them in your project's root as `CLAUDE.md`
- Tell Claude Code: "Use the local CLAUDE.md file in the project root as your rules"

---

### The fetched CLAUDE.md is 404

**Cause:** Wrong repo path or branch name in the URL.

**Fix:** The URL pattern is:
```
https://raw.githubusercontent.com/<user>/<repo>/<branch>/<path>
```
Default branch is usually `main`. If your fork uses `master`, update the URLs accordingly.

---

## BUILD / DEPENDENCY ERRORS

### `Module not found: Can't resolve '...'`

**Cause:** Dependency isn't installed, or the import path is wrong.

**Fix:** Tell Claude Code:
> "Run the install command for the missing package, then retry. If the package doesn't exist, find the correct name or alternative."

Common gotchas:
- `@/components/...` imports need `tsconfig.json` path aliases configured
- Scoped packages (`@prisma/client`) vs. tools (`prisma`) are separate installs
- Peer dependency warnings are not errors — unless they say `ERR_PEER_DEP_REQUIRED`

---

### TypeScript compile errors flood the terminal

**Cause:** Usually one root cause cascading into many symptoms.

**Fix:** Always fix the **first error** in the list, not the last. Tell Claude Code:
> "Focus only on the first TypeScript error. Fix it, re-run tsc, and work through them in order."

Never delete errors by adding `any` — that's a debt trap. If Claude Code reaches for `any`, push back and ask for the real type.

---

### `Cannot find module 'X' or its corresponding type declarations`

**Cause:** Missing types package.

**Fix:** Install the corresponding `@types/...` package:
```bash
npm install -D @types/node @types/react
```
Most modern packages ship their own types — if `@types/foo` doesn't exist on npm, the types are built-in.

---

## DATABASE ERRORS

### `ECONNREFUSED` or `could not connect to server` (PostgreSQL)

**Cause:** Database isn't running, wrong host/port, or firewall blocking.

**Fix checklist:**
1. Is PostgreSQL actually running? `docker ps` (if using Docker) or check your DB service
2. Is `DATABASE_URL` in `.env` correct? Format: `postgresql://user:pass@host:port/dbname`
3. If using Supabase/Railway/Neon — double-check you copied the **connection pooler** URL for serverless, **direct** URL for long-running servers
4. Firewall: some managed DBs require your IP to be allowlisted

---

### Prisma: `Environment variable not found: DATABASE_URL`

**Cause:** `.env` not loaded, or Prisma looking in wrong place.

**Fix:**
- Ensure `.env` is in the project root (or wherever `schema.prisma` lives)
- Do not commit `.env` — it should be in `.gitignore`
- Check `.env` has `DATABASE_URL="..."` — the quotes matter for URLs with special characters

---

### Migration fails: "database is not in sync with Prisma schema"

**Cause:** You changed the schema without creating a migration, or you're pointing at a DB that already has data.

**Fix:**
- For **development with empty data** — `npx prisma migrate reset` (destroys data, recreates schema)
- For **real data you want to keep** — `npx prisma migrate dev --name describe_the_change`
- Never run `prisma db push` in production — it can silently lose data

---

## AUTH ERRORS

### JWT expired / `TokenExpiredError`

**Cause:** Access token lifetime passed without refresh.

**Fix:** Verify refresh token flow is implemented. Access tokens should be 15min, refresh tokens 7–30 days. Client should auto-refresh on 401.

---

### CORS: `blocked by CORS policy`

**Cause:** Frontend origin not whitelisted on backend.

**Fix:** In backend CORS config, add the frontend URL:
- Dev: `http://localhost:5173` (or whatever Vite uses)
- Prod: the actual deployed frontend URL

Never use `origin: "*"` with `credentials: true` — browsers reject this combo.

---

### `Invalid token signature` after restart

**Cause:** `JWT_SECRET` changed between restarts (maybe read from a missing env, fell back to random).

**Fix:** Set a stable `JWT_SECRET` in `.env` and **never** fall back to a random value at runtime. Always fail hard if the env is missing.

---

## DEPLOYMENT ERRORS

### Vercel build fails: "Module not found" (but works locally)

**Cause:** Case-sensitivity. Mac/Windows filesystems are case-insensitive; Linux (Vercel's build env) is case-sensitive.

**Fix:** Make sure every import exactly matches the file name — `Button.tsx` vs `button.tsx` matters on Linux.

---

### Railway/Fly deployment: app crashes on startup

**Cause:** Environment variables not set on the platform.

**Fix:**
1. List what your app reads from `process.env` / `os.environ`
2. Set each one in the platform's env settings
3. Redeploy

Never hardcode env values for production "just to test" — it's how secrets leak.

---

### Prisma: "Prisma Client not generated" on deploy

**Cause:** Build step doesn't run `prisma generate`.

**Fix:** Add to `package.json`:
```json
"scripts": {
  "postinstall": "prisma generate",
  "build": "prisma generate && next build"
}
```

---

## AI / LLM ERRORS

### `401 Unauthorized` from Anthropic/OpenAI

**Cause:** Invalid API key or key not loaded.

**Fix:**
- Confirm `ANTHROPIC_API_KEY` or `OPENAI_API_KEY` is set in `.env`
- Restart your dev server after adding env vars — Node.js doesn't hot-reload them
- Check the key isn't expired or revoked in the provider's dashboard

---

### `Rate limit exceeded`

**Cause:** Too many requests in a short window.

**Fix:**
- Implement retry with exponential backoff
- For production, add a queue (BullMQ, Cloud Tasks) to smooth bursts
- Consider caching responses when the same input is requested frequently

---

### AI responses are slow

This is normal for chat models. Use **streaming** (SSE) so tokens appear as they're generated rather than waiting for the full response. See `addons/ai-features.md`.

---

## REAL-TIME / WEBSOCKET ERRORS

### WebSocket disconnects after ~60 seconds

**Cause:** Proxy or load balancer closing idle connections.

**Fix:** Implement heartbeat — client and server ping/pong every 30 seconds. Also increase keep-alive timeouts on your reverse proxy if self-hosted.

---

### Socket.IO: "disconnected" loop on Vercel

**Cause:** Vercel serverless does not support persistent WebSocket connections.

**Fix:** Run your WebSocket server on Railway/Fly/Render — somewhere with long-running processes. Or use a managed service (Pusher, Ably, Supabase Realtime).

---

## GIT / VERSION CONTROL ERRORS

### `.env` accidentally committed

**Fix immediately:**
1. Add `.env` to `.gitignore`
2. `git rm --cached .env`
3. Commit the removal
4. **Rotate every secret that was in the file** — assume it's compromised
5. Consider using `git filter-repo` or BFG to scrub history for public repos

---

### Merge conflicts after long-running branch

**Fix:** Rebase early, rebase often. If deep in conflicts:
```bash
git rebase --abort
git pull origin main
git rebase main
```
Resolve each conflict file by file, test after each one.

---

## "CLAUDE CODE IS STUCK" ERRORS

### Claude Code is looping on the same error

**Cause:** It's trying fixes that aren't addressing the real issue.

**Fix:** Stop it. Come back to Forge with:
1. What Claude Code was asked to build
2. The exact error message
3. What fixes it already tried

Forge will generate a **targeted diagnostic prompt** that redirects Claude Code to the actual root cause.

---

### Claude Code is making changes outside the session scope

**Cause:** The session prompt was too broad, or Claude Code is over-reaching.

**Fix:** Stop the session. Tell Forge what happened. Forge will regenerate the session prompt with tighter scope — explicit "only touch these files, only build this feature."

---

### Output doesn't match your vision

Not an error — a design mismatch. Tell Forge what's wrong. It will either regenerate the session with different requirements or give you a corrective prompt for Claude Code.

---

## GENERAL DEBUGGING PRINCIPLES

1. **Read the actual error message.** Not the first line. All of it. The real cause is usually buried three lines down.
2. **Reproduce it deterministically.** If it only fails sometimes, find the pattern before fixing.
3. **Change one thing at a time.** Never apply three fixes simultaneously — you won't know which one worked.
4. **Check the obvious first.** Is the server running? Is the env var set? Is the typo in your code?
5. **When truly stuck, reset to a known-good state.** `git stash`, checkout the last working commit, and rebuild from there with Claude Code.

---

## WHEN IN DOUBT

Come back to Forge with:
- What you were trying to do
- What you actually saw
- The exact error text (copy-pasted, not summarized)

Forge will triage it and give you the next move.
