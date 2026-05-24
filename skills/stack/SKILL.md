---
name: stack
description: "Master guardrail skill for controlled block-by-block project building. Enforces the canonical tech stack: Strapi v5 (backend/CMS) + frappe-ui design tokens (UI) + Vercel (frontend hosting). Use at the start of ANY new project, feature, or component. Trigger on: 'start a new project', 'build this app', 'new feature', 'set up the project', 'what stack should I use', 'how do I structure this'. This skill controls deviation — no technology decisions are made without consulting it first."
user-invocable: true
argument-hint: "[project or feature name]"
---

# Stack — Canonical Stack Guardrails

The stack is decided. Every project starts with the same foundation. Decisions that are already made don't need to be made again — they just need to be followed.

**The only three technology decisions:**
- **Backend / CMS:** Strapi v5
- **UI tokens:** frappe-ui (Tailwind preset)
- **Frontend hosting:** Vercel

Deviation requires an explicit, written reason. Not a feeling. A reason.

---

## THE ARCHITECTURE

```
┌─────────────────────────────────────────────────┐
│  FRONTEND (Vercel)                              │
│  Next.js / Astro / Vue                          │
│  frappe-ui Tailwind tokens for all UI           │
└────────────────┬────────────────────────────────┘
                 │ HTTP (REST API)
                 │ Authorization: Bearer $STRAPI_API_TOKEN
                 ▼
┌─────────────────────────────────────────────────┐
│  BACKEND CMS (Railway / Render / Fly.io)        │
│  Strapi v5                                      │
│  PostgreSQL (prod) / SQLite (dev)               │
│  Cloudinary for media                           │
└─────────────────────────────────────────────────┘
```

**Critical:** Strapi is a stateful Node.js server. It CANNOT run on Vercel serverless. Strapi deploys to Railway/Render/Fly.io. The frontend (Next.js etc.) deploys to Vercel and calls Strapi over HTTP.

---

## GUARDRAILS — WHAT YOU CANNOT DEVIATE FROM

These are non-negotiable without a written architectural decision record:

### NEVER
- Use a different CMS or backend before exhausting Strapi's capabilities
- Write custom CSS without first checking if frappe-ui tokens cover it
- Add a new database (Redis, MongoDB, Elasticsearch) without a `/db-review` pass confirming Postgres can't handle it
- Deploy Strapi to Vercel (it will not work — stateful server)
- Use `SELECT *` in any Strapi custom query
- Store money as FLOAT (always integers in smallest unit)
- Build UI without running `/spec` first
- Skip `/tdd` for any logic that has business rules

### ALWAYS
- Run `/spec` before writing code for any new feature
- Run `/db-review` before finalizing any new content type schema
- Run `/tdd` for any custom controller, service, or business logic
- Use frappe-ui color tokens (`text-ink-gray-*`, `bg-surface-*`, `border-outline-*`) — never hardcode hex values
- Use `documentId` (not `id`) in all Strapi v5 API calls
- Paginate all list queries (`pagination[page]` + `pagination[pageSize]`)
- Use `populate` explicitly — never assume relations are included

---

## BLOCK-BY-BLOCK BUILD ORDER

Every project follows this sequence. Do not skip blocks. Do not build Block 3 before Block 1 is verified.

```
Block 1: Schema & Content Types
  → Define content types in Strapi admin
  → Run /db-review on every schema before moving forward
  → Verify API endpoints return correct shape
  Output: Strapi running locally, content types created, API confirmed working

Block 2: Auth & Permissions
  → Set Strapi roles and permissions (public vs. authenticated)
  → Generate API token for frontend
  → Test with curl before touching frontend
  Output: curl commands work for all required endpoints

Block 3: Data Layer (Frontend)
  → Write typed fetch functions for each Strapi endpoint
  → Run /tdd on fetch functions (mock HTTP, test shape)
  → Never consume raw API response in UI — always parse through a typed function
  Output: Typed data access layer, tested in isolation

Block 4: UI Foundation
  → Set up frappe-ui Tailwind preset
  → Build layout shell (nav, sidebar, page wrapper)
  → Use only frappe-ui tokens — no hardcoded colors or arbitrary spacing
  Output: Working layout with correct token usage

Block 5: Feature UI
  → Build each feature page/component on top of Block 4's shell
  → One component at a time, tested in isolation
  → Connect to Block 3's data layer
  Output: Working feature, connected to real data

Block 6: Deployment
  → Deploy Strapi to Railway/Render with PostgreSQL
  → Set production environment variables
  → Deploy frontend to Vercel, point to production Strapi URL
  → Smoke test every critical path
  Output: Live, working system

Block 7: Hardening
  → Run /hot-path on every endpoint used in the main user flow
  → Run /audit on the frontend
  → Run /debt-audit after first working version
  Output: Performance and quality baseline established
```

---

## BLOCK 1: STRAPI SETUP

### Initial setup

```bash
npx create-strapi@latest my-project --quickstart
cd my-project
npm run develop
```

Admin panel: `http://localhost:1337/admin` (create admin account on first run)

### Content type schema rules

Every content type must have:
- `slug` field (type: uid, targetField: title) for clean URLs
- `publishedAt` (automatically added by `draftAndPublish: true`)
- Explicit field constraints (required, maxLength, etc.)

Example content type schema:
```json
{
  "kind": "collectionType",
  "collectionName": "articles",
  "info": {
    "singularName": "article",
    "pluralName": "articles",
    "displayName": "Article"
  },
  "options": { "draftAndPublish": true },
  "attributes": {
    "title": { "type": "string", "required": true, "maxLength": 255 },
    "slug": { "type": "uid", "targetField": "title" },
    "body": { "type": "richtext" },
    "cover": { "type": "media", "multiple": false, "allowedTypes": ["images"] }
  }
}
```

### Environment variables (`.env`)

```bash
HOST=0.0.0.0
PORT=1337
APP_KEYS=key1,key2,key3,key4
API_TOKEN_SALT=<generate>
ADMIN_JWT_SECRET=<generate>
TRANSFER_TOKEN_SALT=<generate>
JWT_SECRET=<generate>
ENCRYPTION_KEY=<generate>

# Dev: SQLite (default)
DATABASE_CLIENT=sqlite
DATABASE_FILENAME=.tmp/data.db

# Prod: Postgres (swap these in production)
# DATABASE_CLIENT=postgres
# DATABASE_HOST=...
# DATABASE_PORT=5432
# DATABASE_NAME=strapi
# DATABASE_USERNAME=strapi
# DATABASE_PASSWORD=...
# DATABASE_SSL=true

# Media (production)
# CLOUDINARY_NAME=...
# CLOUDINARY_KEY=...
# CLOUDINARY_SECRET=...
```

Generate secure keys:
```bash
node -e "console.log(require('crypto').randomBytes(32).toString('base64'))"
```

### Media: Cloudinary (production)

```bash
npm install @strapi/provider-upload-cloudinary
```

`config/plugins.ts`:
```ts
export default ({ env }) => ({
  upload: {
    config: {
      provider: 'cloudinary',
      providerOptions: {
        cloud_name: env('CLOUDINARY_NAME'),
        api_key: env('CLOUDINARY_KEY'),
        api_secret: env('CLOUDINARY_SECRET'),
      },
    },
  },
})
```

---

## BLOCK 2: API PATTERNS

### REST endpoint structure

| Method | URL | Action |
|--------|-----|--------|
| GET | `/api/[collection]` | List |
| GET | `/api/[collection]/:documentId` | Get one |
| POST | `/api/[collection]` | Create |
| PUT | `/api/[collection]/:documentId` | Update |
| DELETE | `/api/[collection]/:documentId` | Delete |

**V5 identifier is `documentId` (string), not numeric `id`.**

### Response shape (v5 — flat, no `.data.attributes` nesting)

```json
{
  "data": {
    "id": 1,
    "documentId": "cqaabo4pxlrhrjjhxbima0hq",
    "title": "Hello",
    "slug": "hello",
    "createdAt": "...",
    "updatedAt": "...",
    "publishedAt": "..."
  },
  "meta": {}
}
```

List response: `{ "data": [...], "meta": { "pagination": { "page": 1, "pageSize": 10, "total": 42 } } }`

### Query parameters

```
# Fields
?fields[0]=title&fields[1]=slug

# Populate
?populate=cover                    # one relation
?populate=*                        # all top-level

# Filter
?filters[status][$eq]=published
?filters[title][$containsi]=hello
?filters[publishedAt][$notNull]=true

# Sort
?sort=createdAt:desc

# Pagination (always include)
?pagination[page]=1&pagination[pageSize]=10
```

### Frontend fetch pattern

```ts
// lib/strapi.ts — typed data access layer (Block 3)

const STRAPI_URL = process.env.STRAPI_API_URL
const STRAPI_TOKEN = process.env.STRAPI_API_TOKEN

async function strapiGet<T>(path: string, params?: Record<string, string>): Promise<T> {
  const url = new URL(`${STRAPI_URL}/api${path}`)
  if (params) {
    Object.entries(params).forEach(([k, v]) => url.searchParams.set(k, v))
  }
  const res = await fetch(url.toString(), {
    headers: { Authorization: `Bearer ${STRAPI_TOKEN}` },
    next: { revalidate: 60 }, // Next.js ISR
  })
  if (!res.ok) throw new Error(`Strapi error: ${res.status} on ${path}`)
  const json = await res.json()
  return json.data as T
}

// Usage
const articles = await strapiGet<Article[]>('/articles', {
  'fields[0]': 'title',
  'fields[1]': 'slug',
  'populate': 'cover',
  'pagination[page]': '1',
  'pagination[pageSize]': '10',
  'sort': 'createdAt:desc',
})
```

---

## BLOCK 4: FRAPPE-UI SETUP

### Installation

```bash
npm install frappe-ui
# frappe-ui requires tailwindcss
npm install -D tailwindcss @tailwindcss/forms @tailwindcss/typography
```

### `tailwind.config.js`

```js
module.exports = {
  presets: [require('frappe-ui/tailwind/preset')],
  content: [
    './src/**/*.{js,ts,jsx,tsx,vue,astro}',
    './node_modules/frappe-ui/src/**/*.{vue,js,ts}',
  ],
  darkMode: ['selector', '[data-theme="dark"]'],
}
```

### Token reference — use these, nothing else

**Text (ink):**
```
text-ink-gray-9    → primary text (headings, body)
text-ink-gray-7    → secondary text
text-ink-gray-5    → placeholder / disabled
text-ink-gray-3    → very subtle labels
text-ink-white     → text on dark/colored surfaces
```

**Background (surface):**
```
bg-surface-white   → page background / cards
bg-surface-gray-1  → subtle section background
bg-surface-gray-2  → input backgrounds, hover states
bg-surface-gray-3  → pressed / active states
bg-surface-modal   → modal overlay backdrop
```

**Border (outline):**
```
border-outline-gray-1   → card borders, dividers
border-outline-gray-3   → form inputs, stronger borders
border-outline-gray-4   → focus rings, active borders
border-outline-gray-modals → modal borders
```

**Font sizes (use these, not arbitrary values):**
```
text-2xs  → 11px   (badges, timestamps)
text-xs   → 12px   (captions, meta)
text-sm   → 13px   (secondary content)
text-base → 14px   (body text — default)
text-lg   → 15px   (slightly emphasized body)
text-xl   → 16px   (subheadings)
text-2xl  → 20px   (section headings)
text-3xl  → 24px   (page titles)
```

**Shadows:**
```
shadow-sm   → cards, subtle elevation
shadow      → dropdowns, popovers
shadow-md   → modals
shadow-lg   → floating panels
shadow-xl   → priority overlays
```

**Border radius:**
```
rounded-sm  → 4px    (tags, badges)
rounded     → 8px    (inputs, buttons)
rounded-md  → 10px   (cards)
rounded-lg  → 12px   (panels)
rounded-xl  → 16px   (modals)
```

### Dark mode

Set `data-theme="dark"` on `<html>` to activate dark mode. All frappe-ui tokens automatically switch. No custom dark mode CSS needed.

```html
<html data-theme="dark">
```

---

## BLOCK 6: VERCEL DEPLOYMENT

### Frontend (Next.js) on Vercel

```bash
# Install Vercel CLI
npm i -g vercel

# Deploy
vercel
```

Required environment variables in Vercel dashboard:
```
STRAPI_API_URL=https://your-strapi.railway.app
STRAPI_API_TOKEN=your_read_only_token_from_strapi_admin
```

For client-side access (if needed):
```
NEXT_PUBLIC_STRAPI_URL=https://your-strapi.railway.app
```

### Strapi on Railway

```
1. Push Strapi to GitHub
2. New project in Railway → Deploy from GitHub
3. Add PostgreSQL plugin
4. Set environment variables (DATABASE_* from Railway's Postgres plugin)
5. Set all APP_KEYS, JWT secrets, etc.
6. Deploy
```

Strapi will be live at a Railway-generated URL. Use that URL as `STRAPI_API_URL` in Vercel.

---

## INTEGRATION WITH OTHER SKILLS

Run these in sequence for any new feature:

```
/spec        → Define what to build before touching code
/db-review   → Validate Strapi content type schema before creating it
/tdd         → Write tests for any custom Strapi service or controller
/hot-path    → Review frontend fetch functions and API queries after building
/debt-audit  → After first working version; before adding the next feature
/diagnose    → When something breaks
```

---

## DECISION LOG — WHEN TO DEVIATE

If a situation arises where the stack seems inadequate, do this before deviating:

1. State the specific limitation: "Strapi cannot do X because..."
2. Confirm it: check Strapi docs, try a plugin, ask in Strapi Discord
3. If confirmed: document the decision with the reason
4. Choose the minimal addition: one new tool, not a new paradigm

The bar for deviation is: "This is technically impossible with the canonical stack." Not "this might be slightly easier with X."
