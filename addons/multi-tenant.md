# Multi-Tenant SaaS Addon
### Forge Addon | Append when building a SaaS with multiple organizations

---

## WHAT IS MULTI-TENANCY

Your app serves multiple organizations (tenants). Each tenant has its own users and data. Users of Tenant A must never see data from Tenant B — ever.

---

## TENANCY MODELS

### Shared Database, Shared Schema (Default)
- One database, one schema
- Every table has a `tenant_id` column
- Cheapest, easiest to maintain
- **Use this unless you have specific reasons not to**

### Shared Database, Schema-Per-Tenant
- One database, separate PostgreSQL schemas per tenant
- Better isolation, harder to operate at scale
- Use for regulated industries needing stronger isolation

### Database-Per-Tenant
- Full physical isolation
- Most expensive, most isolated
- Use only for enterprise customers needing dedicated infrastructure

---

## SCHEMA RULES (SHARED-SHARED MODEL)

### Every Tenant-Owned Table Has `tenant_id`
```sql
CREATE TABLE organizations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    slug TEXT NOT NULL UNIQUE,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    email TEXT NOT NULL,
    name TEXT NOT NULL,
    role TEXT NOT NULL DEFAULT 'member',  -- owner, admin, member
    created_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE (organization_id, email)
);

CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- INDEX ON EVERY tenant_id COLUMN
CREATE INDEX ON users(organization_id);
CREATE INDEX ON projects(organization_id);
```

---

## ROW-LEVEL SECURITY (POSTGRESQL)

Enable RLS as a defense-in-depth layer:

```sql
ALTER TABLE projects ENABLE ROW LEVEL SECURITY;

CREATE POLICY projects_tenant_isolation ON projects
    USING (organization_id = current_setting('app.current_tenant')::UUID);
```

Set the tenant in each request:
```python
# Before any query
await db.execute(text(f"SET app.current_tenant = '{user.organization_id}'"))
```

This means even if application code has a bug, the database refuses cross-tenant queries.

---

## APPLICATION-LEVEL ENFORCEMENT

### Middleware/Dependency Extracts Tenant
Every authenticated request must know the tenant. Typical pattern:

```python
# FastAPI example
async def get_current_tenant(user: User = Depends(get_current_user)) -> UUID:
    return user.organization_id

@router.get("/projects")
async def list_projects(
    tenant_id: UUID = Depends(get_current_tenant),
    service: ProjectsService = Depends(...),
):
    return await service.list(tenant_id=tenant_id)
```

### Service Layer Always Receives Tenant ID
- Every service method that reads/writes tenant data takes `tenant_id` as the first parameter
- Never trust client-provided tenant IDs — always derive from authenticated user

### Repository Layer Enforces Filtering
```python
class ProjectsRepository:
    async def list(self, tenant_id: UUID) -> list[Project]:
        result = await self.db.execute(
            select(Project).where(Project.organization_id == tenant_id)
        )
        return result.scalars().all()
    
    async def get(self, tenant_id: UUID, project_id: UUID) -> Project | None:
        result = await self.db.execute(
            select(Project).where(
                Project.id == project_id,
                Project.organization_id == tenant_id,  # ALWAYS
            )
        )
        return result.scalar_one_or_none()
```

**Never have a method that takes just `project_id` without `tenant_id`.**

---

## URL/ROUTING PATTERNS

### Subdomain-Based (Best UX)
- `acme.yourapp.com`, `beta-corp.yourapp.com`
- Requires wildcard DNS and SSL
- Subdomain identifies tenant for unauthenticated pages

### Path-Based (Easier Setup)
- `yourapp.com/acme`, `yourapp.com/beta-corp`
- Simpler to deploy
- Common for B2B tools

### Pure Authentication (Simplest)
- User logs in, tenant is inferred from their account
- No tenant in URL
- Less copy-paste friendly but simpler

Pick one and stick with it.

---

## USER-TENANT RELATIONSHIPS

### Single-Tenant Users (Simple)
- Each user belongs to one organization
- Simpler code
- Limiting if users need multiple orgs

### Multi-Tenant Users (Flexible)
- Users can belong to multiple organizations
- Requires user-organization join table
- More complex session management

```sql
CREATE TABLE memberships (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    role TEXT NOT NULL DEFAULT 'member',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE (user_id, organization_id)
);
```

If multi-tenant: user session includes current active organization; provide UI to switch.

---

## ROLES AND PERMISSIONS

### Standard Roles
- **Owner** — full control, billing, can delete org
- **Admin** — full control except billing and org deletion
- **Member** — standard usage
- **Viewer** — read-only

### Permission Checks
```python
def require_role(min_role: str):
    roles_order = ['viewer', 'member', 'admin', 'owner']
    
    async def check(user: User = Depends(get_current_user)):
        if roles_order.index(user.role) < roles_order.index(min_role):
            raise UnauthorizedError()
        return user
    
    return check

# Usage
@router.delete("/projects/{id}")
async def delete_project(
    id: UUID,
    user: User = Depends(require_role('admin')),
):
    # ...
```

---

## BILLING AND SUBSCRIPTIONS

### Per-Organization Billing
- One subscription per organization (not per user)
- Owner/admin manages billing
- Seat-based pricing: track member count

### Subscription State on Organization
```sql
ALTER TABLE organizations ADD COLUMN subscription_status TEXT;
ALTER TABLE organizations ADD COLUMN subscription_tier TEXT;
ALTER TABLE organizations ADD COLUMN subscription_seats INTEGER;
ALTER TABLE organizations ADD COLUMN stripe_customer_id TEXT UNIQUE;
```

### Feature Flags by Tier
```python
FEATURES_BY_TIER = {
    'free': {'max_projects': 3, 'can_export': False},
    'pro': {'max_projects': 100, 'can_export': True},
    'enterprise': {'max_projects': None, 'can_export': True, 'custom_domain': True},
}

def get_features(org: Organization) -> dict:
    return FEATURES_BY_TIER.get(org.subscription_tier, FEATURES_BY_TIER['free'])
```

---

## ONBOARDING FLOW

### Standard Flow
1. User signs up (creates user + creates their organization)
2. They become owner of that org
3. They can invite others
4. Invited users either join via link or are created directly

### Invitations
```sql
CREATE TABLE invitations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    email TEXT NOT NULL,
    role TEXT NOT NULL,
    token TEXT NOT NULL UNIQUE,
    invited_by UUID NOT NULL REFERENCES users(id),
    expires_at TIMESTAMPTZ NOT NULL,
    accepted_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

- Time-limited tokens (7 days default)
- Email with invite link
- Accept creates the membership

---

## DATA EXPORT AND DELETION

### Export
- Every tenant owner can export all their data (GDPR requirement)
- Provide JSON or CSV exports
- Include all related records

### Deletion
- Hard delete or soft delete — decide based on compliance needs
- `ON DELETE CASCADE` from organizations table removes all related data
- Log deletion events for audit trail

---

## TESTING

### Critical Tests
- Cross-tenant isolation: User A cannot access Tenant B's data, even by guessing IDs
- Role enforcement: Members can't do admin actions
- RLS policies: SQL-level isolation works
- Billing state: Features gate correctly based on subscription

### Test Data Setup
- Fixtures with multiple tenants
- Every test that touches tenant data verifies isolation

---

## OBSERVABILITY

### Per-Tenant Metrics
- Track usage per tenant (API calls, storage, etc.)
- Identify heavy users for capacity planning
- Alert on anomalies (one tenant suddenly using 10x resources)

### Audit Logs
```sql
CREATE TABLE audit_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    user_id UUID,
    action TEXT NOT NULL,
    resource_type TEXT NOT NULL,
    resource_id UUID,
    metadata JSONB,
    ip_address INET,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

Log: logins, permission changes, billing changes, data exports, deletions.

---

## SPECIFIC DO NOTS

- Never fetch data without filtering by tenant_id
- Never trust tenant_id from client input — always derive from authenticated user
- Never use shared caches without tenant-prefixed keys (Redis: `tenant:{id}:key`)
- Never allow cross-tenant queries even for admin users (if truly needed, separate "super admin" role with explicit warnings)
- Never forget to filter by tenant in JOINs (the JOIN table's tenant_id check matters too)
- Never ship without RLS as a second line of defense (when using PostgreSQL)
- Never mix personal and organization data — every record belongs to a tenant

---

## SESSION COMPLETION CHECKLIST — MULTI-TENANT SPECIFIC

- [ ] Every tenant-owned table has `tenant_id` column
- [ ] Every `tenant_id` column is indexed
- [ ] Every repository method filters by tenant_id
- [ ] RLS policies enabled on critical tables
- [ ] Role-based permissions enforced
- [ ] Cross-tenant isolation tests pass
- [ ] Audit logging in place for sensitive actions
- [ ] Invitation flow works end-to-end
- [ ] Subscription state is at org level, not user level

---

*This addon is part of Forge by Sultan Al-Muqimi / NQTH LLC.*
