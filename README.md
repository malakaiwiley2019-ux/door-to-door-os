[README.md](https://github.com/user-attachments/files/30350818/README.md)
# SaaS Foundation

Scaffolding only — **no product features are built yet**. This gives you a
running app shell with the pieces every multi-tenant SaaS needs underneath
it, wired together and ready to build on.

## Stack

- **Next.js 14** (App Router, API routes) + TypeScript
- **PostgreSQL** via **Prisma**
- **NextAuth** for session/auth plumbing (credentials provider stubbed in)
- **Multi-tenancy:** shared database, tenant isolation via a `tenant_id`
  column on tenant-scoped tables (not separate schemas or databases)

## What's here

| Piece | File(s) | Purpose |
|---|---|---|
| DB schema | `prisma/schema.prisma` | `Tenant`, `User`, `Membership` (role per tenant), plus NextAuth's required tables |
| Prisma client | `src/lib/prisma.ts` | Singleton client, safe for dev hot-reload |
| Auth config | `src/lib/auth.ts` | NextAuth options, JWT sessions, credentials provider |
| Tenant resolution | `src/middleware.ts` | Extracts tenant slug from subdomain (or `?tenant=` / header in local dev) |
| Tenant access check | `src/lib/tenant.ts` | `getCurrentTenant()` — confirms the signed-in user actually belongs to the resolved tenant before any data access |
| Health check | `src/app/api/health/route.ts` | Confirms the app can reach the DB |
| Dev seed | `prisma/seed.ts` | Creates one demo tenant + owner user |

## Why shared-DB tenancy

You chose shared-database, tenant-scoped-by-column over separate
schemas/databases per tenant. That means:

- Every tenant-owned table needs a `tenantId` column and an index on it
  (see `Membership` for the pattern).
- **All queries for tenant data must filter by `tenantId`.** There's no
  database-level wall between tenants — the wall is `WHERE tenantId = ...`
  in every query. `getCurrentTenant()` + `scopedToTenant()` in
  `src/lib/tenant.ts` exist so this isn't reinvented per-feature, but it's
  worth double-checking as you add tables: it's the one place this
  architecture can silently leak data if a query is written without the
  filter.
- Migrating to per-tenant schemas later is possible but not free — worth
  revisiting if you ever have tenants with very different data-volume or
  compliance needs.

## Local setup

```bash
# 1. Start Postgres
docker compose up -d

# 2. Install deps
npm install

# 3. Configure env
cp .env.example .env
# edit NEXTAUTH_SECRET — generate one with:
openssl rand -base64 32

# 4. Create tables
npm run prisma:migrate

# 5. Seed a demo tenant/user
npm run db:seed

# 6. Run
npm run dev
```

Then visit `http://localhost:3000/api/health` to confirm the DB connection
is live.

## Local tenant testing

Without real subdomains in dev, pass the tenant via query string or header:

```
http://localhost:3000/some-route?tenant=acme
```

or send `x-tenant-slug-override: acme` as a request header. In production,
this comes from the real subdomain (`acme.yourapp.com`) automatically —
see `src/middleware.ts`.

## Deliberately not included yet

No signup/login pages, no dashboard, no billing, no tenant-switcher UI, no
actual product features. Those all build on top of this foundation —
adding a new tenant-scoped model means: add it to `schema.prisma` with a
`tenantId` field + index, migrate, and query it through
`scopedToTenant(tenantId)`.
