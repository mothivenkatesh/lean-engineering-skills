---
name: stack
description: "Locks the technology stack for every project. Enforces: Strapi v5 for backend/CMS, frappe-ui design tokens for UI, Vercel for frontend hosting. Run at the start of any new project, feature, or component. Trigger on: 'start a new project', 'build this app', 'new feature', 'set up the project', 'what stack should I use'. Deviation requires a written reason, not a feeling."
user-invocable: true
argument-hint: "[project or feature name]"
---

# Stack — Canonical Stack Guardrails

## INPUT

Consumes: `DECISION.md` from `/challenge`

Uses: what is being built, the minimum version, the constraints.

Can also run standalone at the start of any project.

---

## THE LOOP

**TRIGGER:** Starting a new project, feature, or component. Or when a technology decision needs to be made.

**CYCLE:**
1. Map what is being built to the stack layer it belongs to
2. Apply NEVER/ALWAYS guardrails for each layer
3. Lock setup commands and configuration
4. Document any deviation with a written reason

**EXIT CONDITION:** Every technology decision is made and locked. STACK.md is written.

---

## THE STACK

Three decisions. Already made. Don't remake them.

```
Backend / CMS: Strapi v5
UI tokens:     frappe-ui (Tailwind preset)
Frontend host: Vercel
```

Deviation requires a written architectural decision record. Not a feeling.

---

## THE ARCHITECTURE

```
┌─────────────────────────────────────────────────┐
│  FRONTEND (Vercel)                              │
│  Next.js / Astro / Vue                          │
│  frappe-ui Tailwind tokens for all UI           │
└────────────────┬────────────────────────────────┘
                 │ HTTP REST
                 │ Authorization: Bearer $STRAPI_API_TOKEN
                 ▼
┌─────────────────────────────────────────────────┐
│  BACKEND CMS (Railway / Render / Fly.io)        │
│  Strapi v5                                      │
│  PostgreSQL (prod) / SQLite (dev)               │
│  Cloudinary for media                           │
└─────────────────────────────────────────────────┘
```

**Critical:** Strapi is a stateful Node.js server. It cannot run on Vercel serverless. Frontend deploys to Vercel. Strapi deploys to Railway/Render/Fly.io.

---

## GUARDRAILS

### NEVER
- Use a different CMS before exhausting Strapi's capabilities
- Write custom CSS without checking frappe-ui tokens first
- Add Redis, MongoDB, Elasticsearch without a `/db-review` pass confirming Postgres can't handle it
- Deploy Strapi to Vercel (it will not work)
- Use `SELECT *` in any Strapi custom query
- Store money as FLOAT (always integers in smallest unit: paise, cents)
- Build UI without running `/spec` first
- Skip `/tdd` for any logic with business rules
- Use Strapi's numeric `id` — always use `documentId` (v5 change)
- Nest API response data manually — Strapi v5 is flat by default

### ALWAYS
- Run `/spec` before writing code for any new feature
- Run `/db-review` before finalizing any content type schema
- Run `/premortem` before any deploy
- Run `/hot-path` before any endpoint goes to production
- Validate required env vars at application startup (fail loudly, not silently)
- Use TIMESTAMPTZ, not TIMESTAMP
- Paginate all list queries

---

## STRAPI V5 PATTERNS

### Content type structure
```
src/api/[resource]/
├── content-types/[resource]/schema.json
├── controllers/[resource].ts   ← HTTP only
├── services/[resource].ts      ← Business logic only
└── routes/[resource].ts
```

### V5 breaking changes from v4

**documentId, not id:**
```javascript
// WRONG (v4)
strapi.db.query('api::article.article').findOne({ where: { id: 1 } })

// RIGHT (v5)
strapi.documents('api::article.article').findOne({ documentId: 'abc123' })
```

**Flat response shape:**
```javascript
// WRONG (v4 shape)
article.data.attributes.title

// RIGHT (v5 shape — flat)
article.title
```

**Always explicit populate:**
```javascript
// WRONG: silently returns no relations
strapi.documents('api::article.article').findMany()

// RIGHT
strapi.documents('api::article.article').findMany({
  populate: { author: true, category: true }
})
```

**Always paginate:**
```javascript
strapi.documents('api::article.article').findMany({
  pagination: { page: 1, pageSize: 25 }
})
```

### REST API query patterns

```
GET /api/articles?populate=author&pagination[page]=1&pagination[pageSize]=25
GET /api/articles/:documentId?populate=deep
POST /api/articles
PUT /api/articles/:documentId
DELETE /api/articles/:documentId

Filters:
?filters[title][$contains]=hello
?filters[publishedAt][$notNull]=true

Sort:
?sort=createdAt:desc
```

---

## FRAPPE-UI TOKEN REFERENCE

### Install
```bash
npm install frappe-ui
```

### Tailwind config
```javascript
module.exports = {
  presets: [require('frappe-ui/tailwind/preset')],
  content: ['./src/**/*.{vue,js,ts,jsx,tsx}', './node_modules/frappe-ui/src/**/*.{vue,js,ts}'],
}
```

### Typography
| Class | Size |
|-------|------|
| `text-2xs` | 11px |
| `text-xs` | 12px |
| `text-sm` | 13px |
| `text-base` | 14px |
| `text-lg` | 16px |
| `text-xl` | 18px |
| `text-2xl` | 20px |
| `text-3xl` | 24px |

### Color tokens
| Use | Token |
|-----|-------|
| Primary text | `text-ink-gray-9` |
| Secondary text | `text-ink-gray-7` |
| Muted | `text-ink-gray-5` |
| Page background | `bg-surface-white` |
| Card background | `bg-surface-gray-1` |
| Card border | `border-outline-gray-1` |
| Primary action | `bg-ink-gray-9 text-white` |
| Destructive | `text-ink-red-5` |

---

## SETUP COMMANDS

### New Strapi v5 project
```bash
npx create-strapi@latest my-backend --quickstart
cd my-backend && npm run develop
```

### New Next.js frontend
```bash
npx create-next-app@latest my-frontend --typescript --tailwind --app
cd my-frontend && npm install frappe-ui
```

### Connect frontend to Strapi
```typescript
// lib/strapi.ts
export async function fetchFromStrapi(path: string, options: RequestInit = {}) {
  const res = await fetch(`${process.env.STRAPI_URL}/api${path}`, {
    ...options,
    headers: {
      Authorization: `Bearer ${process.env.STRAPI_API_TOKEN}`,
      'Content-Type': 'application/json',
      ...(options.headers ?? {}),
    },
  })
  if (!res.ok) throw new Error(`Strapi error: ${res.status}`)
  return res.json()
}
```

### Deploy Strapi to Railway

Environment variables required:
- `DATABASE_URL`
- `APP_KEYS`
- `API_TOKEN_SALT`
- `ADMIN_JWT_SECRET`
- `TRANSFER_TOKEN_SALT`
- `JWT_SECRET`

---

## 7-BLOCK BUILD SEQUENCE

```
Block 1: Strapi schema (content types, fields, relations)
Block 2: Strapi permissions (public vs authenticated routes)
Block 3: Frontend data layer (fetch functions, types)
Block 4: Frontend UI (frappe-ui components, layout)
Block 5: Authentication (if needed)
Block 6: Production deploy (Railway + Vercel, env vars)
Block 7: Smoke tests (critical paths verified in production)
```

Build in this order. Don't build the UI before the schema is stable.

---

## OUTPUT: STACK.md

```markdown
# STACK.md

## Project: [name]

## Stack Decisions
Backend: Strapi v5 on Railway
Frontend: Next.js on Vercel
UI tokens: frappe-ui
Database: PostgreSQL (prod) / SQLite (dev)
Media: Cloudinary

## Deviations from canonical stack
None. [Or: reason for deviation]

## Environment Variables Required
- STRAPI_URL
- STRAPI_API_TOKEN
- DATABASE_URL
- [project-specific vars]

## Build Sequence
Block 1: [...]
Block 7: [...]

## Feeds into
All downstream skills. STACK.md is the fixed foundation.
```

---

## FEEDS INTO

All downstream skills reference STACK.md as the locked foundation. Technology decisions don't get remade mid-build.
