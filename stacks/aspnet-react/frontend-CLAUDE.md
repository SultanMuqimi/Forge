# CLAUDE.md — React Frontend (ASP.NET + React)
### Forge Stack Rules | React Frontend with ASP.NET Core Backend

---

This file extends the universal CLAUDE.md. Read that first.

The React frontend in an ASP.NET+React project follows the **same rules as the React+Node frontend** — see `stacks/react-node/frontend-CLAUDE.md` for full details. This file only documents what is **different** when paired with an ASP.NET Core backend.

---

## WHAT'S DIFFERENT

### API Client Typing
- ASP.NET Core exposes Swagger/OpenAPI at `/swagger/v1/swagger.json`
- Generate TypeScript types: `npx openapi-typescript http://localhost:5000/swagger/v1/swagger.json -o src/types/api.ts`
- Regenerate after backend changes

### CORS
- Backend handles CORS — whitelist the frontend URL in `appsettings.json`
- Frontend does not need special CORS config beyond standard fetch/axios

### Authentication
- JWT tokens from ASP.NET Core come in the same format as other backends
- Store in HTTP-only cookies (recommended) or memory (never localStorage for production)

### Environment Variables
```
VITE_API_URL=https://localhost:5001
```

Note: ASP.NET Core defaults to HTTPS in development with a self-signed cert. May need to trust the cert: `dotnet dev-certs https --trust`

---

## EVERYTHING ELSE

For component rules, state management, routing, forms, styling, testing, and session completion — refer to `stacks/react-node/frontend-CLAUDE.md`. All those rules apply identically.

---

*This file is part of Forge by Sultan Al-Muqimi / NQTH LLC.*
