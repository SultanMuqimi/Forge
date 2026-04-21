# CLAUDE.md — React Frontend (FastAPI + React)
### Forge Stack Rules | React Frontend with Python FastAPI Backend

---

This file extends the universal CLAUDE.md. Read that first.

The React frontend in a FastAPI+React project follows the **same rules as the React+Node frontend** — see `stacks/react-node/frontend-CLAUDE.md` for full details. This file only documents what is **different** when the backend is FastAPI instead of Node.js.

---

## WHAT'S DIFFERENT

### API Client Typing
Since the backend is Python, you won't have automatic TypeScript types shared with the backend. Options:

1. **Generate types from OpenAPI** (recommended)
   - FastAPI auto-generates OpenAPI spec at `/openapi.json`
   - Use `openapi-typescript` to generate types: `npx openapi-typescript http://localhost:8000/openapi.json -o src/types/api.ts`
   - Regenerate after backend changes

2. **Manually define types** in `src/types/` matching backend Pydantic schemas

### Environment Variables
```
VITE_API_URL=http://localhost:8000
```

### CORS
FastAPI handles CORS on the backend side — make sure the dev URL is whitelisted there.

---

## RECOMMENDED WORKFLOW

When the backend changes:
1. Backend team updates Pydantic models
2. Frontend regenerates types: `npm run generate-api-types`
3. TypeScript compiler surfaces any breaking changes
4. Fix frontend code to match

Add this script to `package.json`:
```json
{
  "scripts": {
    "generate-api-types": "openapi-typescript ${VITE_API_URL}/openapi.json -o src/types/api.ts"
  }
}
```

---

## EVERYTHING ELSE

For component rules, state management, routing, forms, styling, testing, and session completion — refer to `stacks/react-node/frontend-CLAUDE.md`. All those rules apply identically.

---

*This file is part of Forge by Sultan Al-Muqimi / NQTH LLC.*
