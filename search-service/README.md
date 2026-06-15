# Amazon Picks

An AI-driven quick-commerce prototype built around two ideas: tell the app the *plan* (not the items) and get a ready-to-checkout cart in seconds, and let the catalog itself predict your reorders before you have to look.

The repo is split into two independently runnable services that talk to each other over HTTP:

| Service | Tech | Role |
|---|---|---|
| `website/` | Next.js 15, React 19, Prisma 7, Postgres | Customer-facing storefront — product catalog, browse / search / cart / checkout / orders, plus the AI-driven Quick Mode and Conversational modals |
| `search-service/` | Node 20, Express, Prisma 5, Postgres (pgvector + pg_trgm), Redis | "SmartCart" cart-planning engine. Takes a natural-language query and returns a structured cart driven by a deterministic retrieval pipeline over a Requirement Graph |

The website calls `search-service` whenever the user types something like "movie night for 4 people" or "Diwali decorations". Everything else (browsing, predictions, orders) is local to the website.

---

## High-level architecture

```
                                   ┌────────────────────────────┐
                                   │ Next.js website (port 3000)│
                                   │  ┌──────────────────────┐  │
   browser ───► quick-mode modal ──┼─►│ /api/quick           │  │
              conversational modal │  │ /api/conversation/*  │  │
              category / search    │  │ /api/products/*      │  │
              cart / checkout      │  │ /api/orders          │  │
              "Buy it again"       │  │ /api/predictions/*   │  │
                                   │  └──────────┬───────────┘  │
                                   │             │              │
                                   │       Prisma│ (catalog,    │
                                   │             │  orders)     │
                                   │             ▼              │
                                   │       Postgres             │
                                   └─────────────┼──────────────┘
                                                 │ HTTP /v1/cart/plan
                                                 ▼
                              ┌──────────────────────────────────┐
                              │ search-service (port 3001/3000)  │
                              │  classifier → planner → resolver │
                              │  → coverage → substitution       │
                              │  → composer → auditor            │
                              └──┬─────────┬──────────────┬──────┘
                                 │         │              │
                          Prisma │  Redis  │ LLM/embeds   │
                                 ▼         ▼              ▼
                          Postgres     Redis        Bedrock or
                       (pgvector +     (sessions,   Gemini + Gemini
                        pg_trgm)        cache)      embeddings (768d)
```

The two services maintain *separate* Postgres databases — they were built independently and the website's catalog (`Product` rows with `price/mrp/rankScore`) is not the same shape as SmartCart's catalog (`Product` rows with `embedding vector(768)` and a `domain` tag). The website talks to SmartCart for cart planning only; everything that backs product cards on the storefront lives in the website's own DB.

---

## Folder structure

```
amazon-hackon-main/
├── README.md                            ← this file
├── .gitignore
│
├── search-service/                      SmartCart API (cart-planning engine)
│   ├── README.md                        Deploy bundle notes (docker-compose)
│   ├── Dockerfile                       Multi-stage Node 20 build
│   ├── docker-compose.yml               app + postgres (pgvector/pg16) + redis
│   ├── package.json                     name: "smartcart-api" v5.0.0, ESM
│   ├── tsconfig.json
│   ├── .dockerignore
│   ├── .gitignore
│   │
│   ├── prisma/
│   │   ├── schema.prisma                Product, SubCategory, MissionKB,
│   │   │                                RequirementCache, Brand, CartRequestLog
│   │   └── migrations/
│   │       ├── 20260615000000_init/
│   │       ├── 20260615120000_v2_mission_kb/
│   │       ├── 20260615180000_v35_brand_cache/
│   │       ├── 20260616000000_v45_domains/
│   │       └── 20260616120000_v5_cart_request_log/
│   │
│   └── src/
│       ├── server.ts                    Express bootstrap, /v1/health, Swagger UI
│       │
│       ├── api/                         HTTP layer (no business logic)
│       │   ├── cart.controller.ts       POST /v1/cart/plan, GET /v1/cart/status/:id
│       │   ├── openapi.ts               Hand-written OpenAPI 3 spec
│       │   └── openapi.types.ts
│       │
│       ├── lib/
│       │   ├── infra/                   Cross-cutting infrastructure
│       │   │   ├── db.ts                Shared Prisma client
│       │   │   ├── redis.ts             Redis client + cacheGet/cacheSet helpers
│       │   │   ├── cache.ts             Postgres-backed RequirementCache (30-day TTL)
│       │   │   ├── llm.ts               Provider switch (Bedrock vs Gemini)
│       │   │   ├── gemini.ts            embedOne(), toPgVector() — embedding helpers
│       │   │   └── providers/
│       │   │       ├── bedrock.ts       Anthropic on AWS Bedrock — JSON tool-use call
│       │   │       └── gemini.ts        Gemini generateContent JSON mode
│       │   │
│       │   ├── middleware/
│       │   │   ├── auth.ts              Bearer-token auth (SMARTCART_API_KEY)
│       │   │   └── requestId.ts         Per-request UUID + access log line
│       │   │
│       │   ├── pipeline/                The 7-stage cart-planning pipeline
│       │   │   ├── classifier.ts        Stage 1 — query → queryType
│       │   │   ├── planner.ts           Stage 2 — Requirement Graph (LLM)
│       │   │   ├── domains.ts           Product-domain rules + festival keywords
│       │   │   ├── constraints.ts       Per-requirement allow/block + name regex
│       │   │   ├── resolver.ts          6-tier retrieval chain
│       │   │   ├── coverage.ts          fulfilled/required ratio
│       │   │   ├── substitution.ts      LLM picks substitutes from same parent cat
│       │   │   ├── cart.ts              Composer (essentials/recommended/premium)
│       │   │   ├── auditor.ts           LLM strict reviewer with up to 2 retries
│       │   │   └── orchestrator.ts      Stitches all stages together
│       │   │
│       │   ├── services/
│       │   │   ├── cartPlanning.service.ts   Service entry point — drives the pipeline
│       │   │   └── responseMapper.ts          Internal → public CartResponse mapping
│       │   │
│       │   └── types/
│       │       └── cart.types.ts        Frozen public response shape
│       │
│       └── scripts/                     One-shot CLI scripts (after `npm run build`)
│           ├── seedProducts.ts          Loads tableConvert.com_*.json into Product
│           ├── seedCategories.ts        Builds SubCategory vocabulary + embeddings
│           ├── seedMissions.ts          Loads MissionKB (dishes, missions, festivals)
│           ├── extractBrands.ts         Computes Brand stats / brandScore
│           ├── extractDomains.ts        Tags every Product with a domain
│           ├── embedProducts.ts         Generates 768-d Gemini embeddings
│           └── eval.ts                  Offline accuracy evaluation harness
│
└── website/                             Storefront (Next.js)
    ├── README.md                        (empty placeholder)
    ├── API.md                           Detailed SmartCart API reference (lives here)
    ├── package.json                     name: "amazon-picks" v0.1.0
    ├── next.config.ts                   Allows cdn.zeptonow.com images, 2 MB actions
    ├── tsconfig.json
    ├── postcss.config.mjs
    ├── components.json                  shadcn/radix config
    ├── prisma.config.ts
    ├── seed.ts                          ~8 MB generated catalog dump (used by seed-db)
    ├── .gitignore
    │
    ├── prisma/
    │   ├── schema.prisma                Product (catalog), Order
    │   └── migrations/
    │       └── 20260614140245_init/
    │
    ├── scripts/
    │   ├── seed-db.ts                   `npm run seed` — inserts seed.ts into Postgres
    │   ├── build-seed.mjs               Builds seed.ts from raw catalog data
    │   └── dedupe-seed.mjs              Drops duplicate IDs before seeding
    │
    ├── app/                             Next.js App Router
    │   ├── layout.tsx                   Header, Footer, Toaster, both AI modals
    │   ├── page.tsx                     Home: hero + predictive section + tiles + rows
    │   ├── globals.css                  Tailwind v4 + Amazon-skinned CSS variables
    │   ├── loading.tsx
    │   │
    │   ├── cart/page.tsx                Local cart view
    │   ├── checkout/page.tsx            Local checkout view
    │   ├── orders/page.tsx              Order history (user "garv")
    │   ├── order/[id]/page.tsx          Order detail
    │   ├── product/[id]/page.tsx        PDP with related row
    │   ├── category/[slug]/page.tsx     Category + search results (uses /api/products)
    │   │
    │   └─ api/
    │       ├── categories/route.ts             GET — category list w/ icon + count
    │       ├── products/route.ts               GET — paged search/sort
    │       ├── products/[id]/route.ts          GET — product + related
    │       ├── products/by-ids/route.ts        GET ?ids=a,b,c — batch lookup
    │       ├── orders/route.ts                 GET / POST — list + place order
    │       ├── predictions/eligible/route.ts   GET — recurring-staple suggestions
    │       ├── conversation/route.ts           POST — proxy to SmartCart /v1/cart/plan
    │       ├── conversation/questions/route.ts POST — Gemini-driven clarifiers + fallback
    │       └── quick/route.ts                  POST — Quick Mode (one-shot SmartCart call)
    │
    ├── components/
    │   ├── header.tsx                   Sticky Amazon-style header
    │   ├── footer.tsx
    │   ├── hero-quick.tsx               Home hero + AI prompt input
    │   ├── predictive-section.tsx       "Buy it again, right on time" reorder cards
    │   ├── product-card.tsx, product-row.tsx, product-tile.tsx
    │   ├── product-detail-client.tsx
    │   ├── filter-rail.tsx, cart-view.tsx, checkout-view.tsx
    │   │
    │   ├── conversational/
    │   │   └── conversational-modal.tsx       Multi-turn chat — full conversation UI
    │   │
    │   ├── quick-mode/
    │   │   ├── quick-mode-button.tsx           Header CTA
    │   │   ├── quick-mode-modal.tsx            Animated 3-step modal (input/think/results)
    │   │   ├── input-step.tsx
    │   │   ├── thinking-step.tsx
    │   │   └── results-step.tsx
    │   │
    │   └── ui/                          shadcn-style primitives (button, dialog, …)
    │
    ├── hooks/
    │   └── use-hydrated-cart.ts         SSR-safe Zustand cart hook
    │
    └── lib/
        ├── db.ts                        Prisma client (driver-adapter Postgres)
        ├── format.ts                    rupees / counts / slug helpers
        ├── utils.ts                     `cn()` Tailwind class merger
        ├── theme.ts                     Vibe palettes + category tile art + tints
        ├── recurring.ts                 Recurring-staple keyword config
        ├── schemas.ts                   Zod request schemas for /api routes
        ├── gemini.ts                    Direct Gemini client for /api/conversation/questions
        │
        ├── cart-store.ts                Zustand cart (localStorage)
        ├── conv-store.ts                Zustand conversational-modal state
        ├── prediction-store.ts          Zustand prediction history / snooze / removed
        ├── quick-mode-store.ts          Zustand Quick Mode modal open state
        │
        └── services/
            ├── products.ts              Catalog read API (categories, list, related…)
            ├── orders.ts                Order create / list / get + line builder
            ├── tier-builder.ts          CartLineItem helper type
            └── smartcart.ts             HTTP client to search-service /v1/cart/plan
```

---

## What the website does

### Core storefront
- **Home** (`app/page.tsx`) — Amazon-skinned grid: AI hero, predictive reorders, themed category tiles (only categories with bespoke art are shown — generic fallbacks are hidden), top-deals row plus two themed rows derived from whatever beverage / snack categories exist.
- **Browse** — `/category/[slug]` and `/category/search?q=…` are the same page, served by `app/category/[slug]/page.tsx` against `/api/products` with pagination, sort modes (`relevance | price-asc | price-desc | rating | discount`), and a left-rail filter.
- **Product** — `app/product/[id]/page.tsx` reads from Prisma, shows MRP / discount / star bar / stock state, plus a related row built from same sub-category or category (excluding self).
- **Cart / Checkout / Orders** — all local. Orders are persisted to the website's Postgres under hard-coded `userId="garv"` with a hard-coded Connaught Place address.

### AI cart-building (two modes)

| | Quick Mode | Conversational Mode |
|---|---|---|
| Trigger | Top-right "Quick Mode" button | Hero input box on home page |
| Flow | One shot — input → spinner → results | Multi-turn chat with up to 2-3 clarifying questions |
| Frontend store | `quick-mode-store.ts` | `conv-store.ts` |
| API endpoint | `POST /api/quick` | `POST /api/conversation` (with optional question round via `/api/conversation/questions`) |
| Backend call | `planCart({ query: intent })` once | `planCart({ query, parameters, sessionId })` after questions answered |
| Output | Vibe-themed cart preview + total + delivery ETA | Streamed messages + cart line items inside the modal |

Both ultimately call `lib/services/smartcart.ts` → `POST {SMARTCART_API_URL}/v1/cart/plan`. The website never asks SmartCart's `/v1/cart/chat` — *clarifying questions are generated locally* by `app/api/conversation/questions/route.ts` (Gemini if `GEMINI_API_KEY` is set, otherwise a topic-routed question bank covering baby / pet / cleaning / personal-care / healthcare / beverage / breakfast / party / movie / meal / default).

### "Buy it again, right on time"
`components/predictive-section.tsx` reads `/api/predictions/eligible`, which scans the catalog for products matching a recurrence config (`lib/recurring.ts`: milk every 2d, water every 2d, bread every 3d, badam every 3d, eggs every 4d, Maggi every 5d, butter every 7d, cornflakes every 10d). On first load it auto-seeds 4 prior orders per match in `prediction-store.ts` so the demo immediately shows "Due now / Due tomorrow" cards. The user can add, skip, snooze (24h), or remove a product from prediction.

---

## What the SmartCart pipeline does

The orchestrator (`search-service/src/lib/pipeline/orchestrator.ts`) runs this exact chain for every `/v1/cart/plan` call:

```
{ query, parameters }
        │
        ▼
  ┌──────────────┐   classifier.ts
  │ 1. Classify  │   queryType ∈ { product | brand | ingredient | dish |
  │              │                 mission | festival | category | unknown }
  └──────┬───────┘   confidence + slug + brand + ingredient + categories
         │
         ▼
  ┌──────────────┐   planner.ts
  │ 2. Plan      │   Build a Requirement[] (essentials / recommended /
  │              │   premium). Each has type (exact_product/brand/name/
  │              │   subcategory), nameMatch, hints, brand, priority,
  └──────┬───────┘   quantity. Cached in RequirementCache (30-day TTL).
         │
         ▼
  ┌──────────────┐   constraints.ts + domains.ts
  │ 3. Constrain │   For each requirement build a Constraint:
  │              │   { allowed, blocked, festival, nameBoost, nameExclude }
  │              │   driven by queryType + festival + parameters
  └──────┬───────┘   (taste / vegetarian / vegan / glutenFree / ...).
         │
         ▼
  ┌──────────────┐   resolver.ts — strict 6-tier chain
  │ 4. Resolve   │   1. Exact product (nameMatch + optional sub-cat fence)
  │              │   2. Brand match (exact, then pg_trgm fuzzy)
  │              │   3. Sub-category match (hints ∩ subCategory)
  │              │   4. Category broaden (parent of hints)
  │              │   5. Synonym (pg_trgm on SubCategory.name)
  │              │   6. Embedding (pgvector cosine on synthetic text)
  │              │
  │              │   First tier with hits wins. Constraint filters BEFORE
  │              │   ranking. Score formula:
  │              │     0.30·rating + 0.25·log(reviews) + 0.20·popularity
  │              │     + 0.25·brandScore + bonuses (name-start, brand exact,
  │              │     constraint nameBoost).
  │              │
  │              │   Critical invariant: when nameMatch is set and misses,
  │              │   return [] — do NOT fall through to embedding. Atta
  └──────┬───────┘   never replaces Pav.
         │
         ▼
  ┌──────────────┐   coverage.ts
  │ 5. Coverage  │   coverage = fulfilled_required / total_required
  │              │   ≥ 0.90 → pass (run substitution for any gaps)
  └──────┬───────┘   < 0.90 → needs_user_input (caller's responsibility)
         │
         ▼ (only when there are gaps)
  ┌──────────────┐   substitution.ts
  │ 6. Substitute│   LLM picks 1-2 in-category substitutes per missing
  │              │   essential (e.g. Pav → Bread Roll / Burger Bun).
  │              │   "diaper / formula / baby food / incontinence" are
  └──────┬───────┘   never auto-substituted.
         │
         ▼
  ┌──────────────┐   cart.ts
  │ 7. Compose   │   Picks top fresh per requirement (no cross-tier dupes).
  │              │   recommended = top 2 by score.
  │              │   premium = top 3 by composite:
  │              │     0.40·complementary + 0.30·rating
  │              │     + 0.20·mission_relevance + 0.10·margin_proxy
  └──────┬───────┘
         │
         ▼ (loops up to 2 retries)
  ┌──────────────┐   auditor.ts
  │ 8. Audit     │   Strict LLM reviewer flags productIds to remove for:
  │              │   domain mismatch, off-topic items, wrong brand for
  │              │   a brand query, festival-rivalry violations, baby-food
  │              │   masquerading as adult food. NEVER suggests replacements
  │              │   — orchestrator drops the bans, recomposes, re-audits.
  └──────┬───────┘
         │
         ▼
   final CartResponse  (frozen shape — see website/API.md §6)
```

Every response is cached in Redis (`request:{requestId}`, 24h) and persisted to the `CartRequestLog` table (forever). `GET /v1/cart/status/:requestId` replays from either.

`?debug=1` (or `X-Debug: 1`) attaches a `debug` block exposing classifier confidence, every resolverPath taken, every constraint that fired, all substitutions, the auditor verdict, and a notes log.

For the full request/response contract, see [`website/API.md`](website/API.md).

---

## Running locally

### Prerequisites
- Node 20+
- npm 10+ (or pnpm — both projects use npm scripts and a `package-lock.json`)
- Docker (only for the SmartCart deploy bundle; you can run Postgres / Redis natively if you prefer)
- A Postgres 16 with `vector` and `pg_trgm` extensions for SmartCart, and any Postgres 14+ for the website
- LLM credentials — at minimum `GEMINI_API_KEY`. Optional `AWS_BEARER_TOKEN_BEDROCK` if you set `LLM_PROVIDER=bedrock`

### 1. Bring up SmartCart (port 3000 inside the container)

```bash
cd search-service
cp .env.example .env       # then fill GEMINI_API_KEY, AWS_BEARER_TOKEN_BEDROCK,
                           # SMARTCART_API_KEY (any string — leave unset to disable auth)
docker compose up -d --build
```

Healthcheck: `curl http://localhost:3000/v1/health`.

Seed the catalog and knowledge bases (one-time, in this order):

```bash
docker compose exec app npm run seed:products
docker compose exec app npm run embed:products
docker compose exec app npm run seed:categories
docker compose exec app npm run seed:missions
docker compose exec app npm run extract:brands
docker compose exec app npm run extract:domains
```

Swagger UI: `http://localhost:3000/v1/docs`. OpenAPI JSON: `http://localhost:3000/v1/openapi.json`.

### 2. Bring up the website (port 3000 by default — change one of them)

```bash
cd website
# .env.local
# DATABASE_URL=postgres://user:pass@localhost:5432/amazon_picks
# SMARTCART_API_URL=http://localhost:3001     ← whatever port SmartCart is on
# SMARTCART_API_KEY=...                       ← match search-service/.env if auth is on
# GEMINI_API_KEY=...                          ← used for the conversational clarifier

npm install
npm run db:migrate
npm run seed                # loads seed.ts (~8 MB) into Product
npm run dev
```

Open `http://localhost:3000`. Click *Quick Mode* in the header and try "movie night for 4 people", or use the hero input for the conversational flow.

### Note on port conflicts
Both projects default `PORT=3000`. The simplest fix is to publish SmartCart on a different host port:

```yaml
# search-service/docker-compose.yml — already supports APP_PORT override
APP_PORT=3001 docker compose up -d
```

…and point `SMARTCART_API_URL=http://localhost:3001` from the website.

---

## Environment reference

### `search-service/.env`

| Var | Required | Notes |
|---|---|---|
| `DATABASE_URL` | yes | Compose overrides this to the in-cluster `postgres` service |
| `REDIS_URL` | yes | Compose overrides this to `redis://redis:6379` |
| `LLM_PROVIDER` | yes | `bedrock` or `gemini` |
| `GEMINI_API_KEY` | always | Needed for embeddings even on Bedrock |
| `AWS_BEARER_TOKEN_BEDROCK` | if Bedrock | Bedrock API key |
| `BEDROCK_REGION`, `BEDROCK_MODEL_ID`, `BEDROCK_EMBED_MODEL_ID` | if Bedrock | Defaults provided |
| `SMARTCART_API_KEY` | recommended | Bearer-token auth. Empty = auth disabled |
| `PORT` | no | Defaults to `3000` |

### `website/.env.local`

| Var | Required | Notes |
|---|---|---|
| `DATABASE_URL` | yes | Postgres for catalog + orders |
| `SMARTCART_API_URL` | yes | Base URL of `search-service` (default `http://localhost:3001`) |
| `SMARTCART_API_KEY` | when SmartCart has auth on | Bearer token forwarded to `/v1/cart/plan` |
| `GEMINI_API_KEY` | recommended | Powers `/api/conversation/questions`; falls back to a topic-routed bank if absent |

---

## Public API

The website exposes routes at `/api/*` — these are internal, not designed for third parties. Schemas are in `website/lib/schemas.ts`.

The SmartCart service has its own stable, versioned API at `/v1/*`. Full spec: [`website/API.md`](website/API.md).

| Method | Path | Purpose |
|---|---|---|
| GET | `/v1/health` | Liveness — public |
| GET | `/v1/openapi.json` | OpenAPI spec — public |
| GET | `/v1/docs` | Swagger UI — public |
| POST | `/v1/cart/plan` | Single-shot stateless plan |
| GET | `/v1/cart/status/:requestId` | Replay a stored response |

`/v1/cart/chat` is documented in `API.md` as a multi-turn endpoint, but the website does not currently call it — it generates clarifying questions locally and then sends the answered query to `/v1/cart/plan` as `parameters`.

---

## Data model

### `search-service` (postgres + pgvector + pg_trgm)
- **Product** — id, name, image, price, rating, reviews, quantity, sub/category, brand, domain, inStock, syntheticText, embedding vector(768) via raw SQL
- **SubCategory** — vocabulary with name + parent + description + embedding vector(768) via raw SQL
- **MissionKB** — slug-keyed KB of dishes/missions/festivals: aliases, essentials/recommended/premium as JSON Requirement[]
- **RequirementCache** — `(query, queryType)` → planner output, 30-day TTL enforced at read time
- **Brand** — name, productCount, avgRating, totalReviews, brandScore (0..1)
- **CartRequestLog** — full frozen `CartResponse` for every `/v1/cart/plan` call, indexed by requestId + sessionId + createdAt

### `website` (postgres)
- **Product** — id, name, brand, category, subCategory, size, price, mrp, rating, ratingCount, deliveryMin, status, tags[], rankScore, img
- **Order** — id (cuid), userId (default `"garv"`), items (JSON snapshot), subtotal/savings/total, paymentMethod, zoneCode (default `"110001"`), deliveryMin, address (JSON), status (default `"placed"`), createdAt

The two Postgres instances do not need to share data. SmartCart returns `productId` strings that the website doesn't need to look up — the response carries name, image, price, rating, reviews, brand and sub-category inline, which is enough to render Quick Mode and conversational results without a join.

---

## Useful npm scripts

### `search-service/`
```
npm run build            # prisma generate + tsc
npm start                # node dist/server.js
npm run db:migrate       # prisma migrate deploy
npm run seed:products    # tableConvert.com_*.json → Product
npm run seed:categories  # SubCategory vocab + embeddings
npm run seed:missions    # MissionKB
npm run embed:products   # Gemini 768-d embeddings → Product.embedding
npm run extract:brands   # Brand stats + brandScore
npm run extract:domains  # Tags Product.domain via rules + LLM fallback
```

### `website/`
```
npm run dev              # next dev
npm run build            # next build
npm run start            # next start
npm run lint
npm run db:migrate       # prisma migrate dev
npm run db:generate      # prisma generate
npm run db:studio        # prisma studio
npm run seed             # tsx scripts/seed-db.ts (loads seed.ts into Postgres)
```

---

## Tech notes worth knowing

- The website uses **Tailwind v4** (`@tailwindcss/postcss`), **React 19**, **Next.js 15** App Router, **Prisma 7** with the `@prisma/adapter-pg` driver adapter, and **Zustand** for cart / conversation / prediction state. UI primitives are shadcn-style with Radix Dialog and Radio under the hood. `framer-motion` drives the modal transitions and `sonner` is the toast layer.
- The website's `app/page.tsx`, `app/product/[id]/page.tsx`, and `app/category/[slug]/page.tsx` all set `export const dynamic = "force-dynamic"` — every request hits the database, no static caching.
- `search-service` is **ESM-only** (`"type": "module"` + `tsconfig.module: NodeNext`). All imports in `src/` use `.js` suffixes even when the source is `.ts`.
- Embeddings are pinned to **Gemini text-embedding-004** at 768 dims because the seeded catalog already has Gemini embeddings — switching to Bedrock for *generation* doesn't change embeddings.
- **All LLM calls are JSON-mode** with strict schemas — the classifier, planner, auditor, substitution engine, and the website's question generator. The provider abstraction (`search-service/src/lib/infra/llm.ts`) makes Gemini and Bedrock interchangeable for generation.
- The auditor is a real safety net, not theatre: it has dropped a Christmas tree from a Diwali query, dropped Hakka noodles from a "Baby Food" requirement, and forced a brand-correct retry when Yippee snuck into a "Maggi" cart. Up to 2 retries before the cart ships as-is with a `partial_success` flag.
- The website's catalog product images come from `cdn.zeptonow.com`. `next.config.ts` whitelists that origin under `images.remotePatterns`. Many product cards also fall back to `tileTint()` (`lib/theme.ts`) — keyword-derived bg/fg pairs that render as soft tinted placeholders when no image renders.