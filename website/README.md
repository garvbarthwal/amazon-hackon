# Amazon Picks — Website

Customer-facing storefront for the Amazon Picks prototype. A Next.js 15 / React 19 quick-commerce front-end with an Amazon-skinned UI, a full product catalog, local cart and checkout, an order history, predictive reorder cards, and two AI-driven cart-building modes (Quick Mode and Conversational) that delegate to the SmartCart service.

The website is one of two services in this repo. It owns the storefront and its catalog/orders database. It calls the [`search-service`](../search-service) over HTTP for AI cart planning. Detailed SmartCart contract: [`API.md`](./API.md).

---

## Tech stack

| Layer | Choice |
|---|---|
| Framework | Next.js 15 (App Router) |
| UI | React 19, Tailwind v4 (`@tailwindcss/postcss`), shadcn-style primitives over Radix Dialog / Radio |
| Animation / toasts | `framer-motion`, `sonner` |
| Icons | `lucide-react` |
| State (client) | Zustand (`zustand/middleware/persist` for cart + predictions) |
| Validation | Zod |
| Database | Postgres via Prisma 7 with the `@prisma/adapter-pg` driver adapter |
| LLM (clarifying questions only) | Gemini 2.0 Flash via `@google/genai` (with a rule-based fallback) |
| Tooling | `tsx` (seed), Prisma migrate / studio, ESLint via `next lint` |

---

## Folder structure

```
website/
├── README.md                      ← this file
├── API.md                         Full SmartCart /v1/* HTTP contract (lives here)
├── package.json                   name: "amazon-picks" v0.1.0, scripts below
├── package-lock.json
├── next.config.ts                 Allows cdn.zeptonow.com images, 2 MB server actions
├── tsconfig.json                  Path alias: "@/*" → "./*"
├── postcss.config.mjs             Tailwind v4 plugin
├── components.json                shadcn config
├── prisma.config.ts
├── seed.ts                        ~8 MB generated catalog dump (consumed by seed-db)
├── .gitignore
│
├── prisma/
│   ├── schema.prisma              Product (catalog), Order
│   └── migrations/
│       └── 20260614140245_init/
│
├── scripts/
│   ├── seed-db.ts                 `npm run seed` — inserts seed.ts into Postgres
│   ├── build-seed.mjs             Builds seed.ts from raw catalog data
│   └── dedupe-seed.mjs            Drops duplicate IDs before seeding
│
├── app/                           Next.js App Router
│   ├── layout.tsx                 Header + Footer + Toaster + both AI modals,
│   │                              fetches categories server-side for the navbar
│   ├── page.tsx                   Home: hero / predictive / category tiles / 3 rows
│   ├── globals.css                Tailwind v4 + Amazon-skinned CSS variables
│   ├── loading.tsx                Route-level loading skeleton
│   │
│   ├── cart/page.tsx              Local cart view
│   ├── checkout/page.tsx          Local checkout view
│   ├── orders/page.tsx            Server-rendered order history (user "garv")
│   ├── order/[id]/page.tsx        Single order detail
│   ├── product/[id]/page.tsx      PDP with related row
│   ├── category/[slug]/page.tsx   Category browse + /category/search?q=… results
│   │
│   └── api/                       Next.js Route Handlers
│       ├── categories/route.ts             GET — list w/ icon, count, slug, hasArt
│       ├── products/route.ts               GET — paginated search/sort/filter
│       ├── products/[id]/route.ts          GET — product + related
│       ├── products/by-ids/route.ts        GET ?ids=a,b,c — batch lookup (≤200)
│       ├── orders/route.ts                 GET / POST — list + place order
│       ├── predictions/eligible/route.ts   GET — recurring-staple products
│       ├── conversation/route.ts           POST — proxy to SmartCart /v1/cart/plan
│       ├── conversation/questions/route.ts POST — Gemini clarifiers + fallback bank
│       └── quick/route.ts                  POST — Quick Mode (one-shot SmartCart)
│
├── components/
│   ├── header.tsx                 Sticky Amazon-style header w/ search + cart count
│   ├── footer.tsx
│   ├── hero-quick.tsx             Home hero — input opens Conversational modal
│   ├── predictive-section.tsx     "Buy it again, right on time" reorder cards
│   ├── product-card.tsx           Grid card (used on category / search)
│   ├── product-row.tsx            Horizontal-scroll row (used on home)
│   ├── product-tile.tsx           Image / tinted-fallback tile
│   ├── product-detail-client.tsx  PDP add-to-cart + qty client island
│   ├── filter-rail.tsx            Left rail: category links + sort
│   ├── cart-view.tsx
│   ├── checkout-view.tsx
│   │
│   ├── conversational/
│   │   └── conversational-modal.tsx   Multi-turn modal — chat → checkout → done
│   │
│   ├── quick-mode/
│   │   ├── quick-mode-button.tsx       Header CTA
│   │   ├── quick-mode-modal.tsx        3-step animated modal shell
│   │   ├── input-step.tsx              Intent textarea + people stepper + chips
│   │   ├── thinking-step.tsx           Animated "decompose / search / compose"
│   │   └── results-step.tsx            Cart preview + "Add all" + delivery ETA
│   │
│   └── ui/                         shadcn-style primitives
│       ├── button.tsx
│       ├── dialog.tsx
│       ├── radio-group.tsx
│       ├── skeleton.tsx
│       └── toaster.tsx              `sonner` re-export
│
├── hooks/
│   └── use-hydrated-cart.ts        SSR-safe Zustand cart hook (avoids hydration jump)
│
└── lib/
    ├── db.ts                       Prisma client (singleton, pg driver adapter)
    ├── format.ts                   formatRupees, discountPct, starPct, slugify…
    ├── utils.ts                    `cn()` Tailwind class merger
    ├── theme.ts                    Vibe palettes, category tile art, product tints
    ├── recurring.ts                Recurring-staple keyword config
    ├── schemas.ts                  Zod request schemas for /api routes
    ├── gemini.ts                   `invokeLLM()` for /api/conversation/questions
    │
    ├── cart-store.ts               Zustand cart (persisted to localStorage)
    ├── conv-store.ts               Zustand conversational-modal state
    ├── prediction-store.ts         Zustand prediction history / snooze / removed
    ├── quick-mode-store.ts         Zustand Quick Mode modal open state
    │
    └── services/
        ├── products.ts             Catalog read API (categories, list, related, …)
        ├── orders.ts               Order create / list / get + line builder
        ├── tier-builder.ts         CartLineItem helper type
        └── smartcart.ts            HTTP client for SmartCart /v1/cart/plan
```

---

## What it does

### Storefront (no AI)
- **Home (`app/page.tsx`)** — Renders the AI hero (`hero-quick.tsx`), the predictive section (`predictive-section.tsx`), a 5-tile category grid (only categories with bespoke art in `lib/theme.ts` are shown — the others would render as generic 🛒/grey, so they're hidden), then "Today's top deals" plus two themed rows whose categories are derived from whatever beverage / snack categories exist in the catalog.
- **Browse (`app/category/[slug]/page.tsx`)** — Same page handles category and search:
  - `/category/snacks-munchies` — slug-resolved category browse
  - `/category/search?q=oreo` — full-text search across name / brand / tags
  - Shared `FilterRail` for category links + sort, and a Zod-validated query (`relevance | price-asc | price-desc | rating | discount`).
- **Product (`app/product/[id]/page.tsx`)** — PDP with breadcrumb, MRP / discount / star bar / stock state, and a related row built from same sub-category (with category fallback), excluding self.
- **Cart / Checkout / Orders** — Local. Orders persist to Postgres under hard-coded `userId="garv"` and a hard-coded Connaught Place 110001 address.

### AI cart-building — two modes

| | Quick Mode | Conversational |
|---|---|---|
| Trigger | Header "Quick Mode" button | Hero input box on home page |
| Flow | One shot — input → spinner → results | Multi-turn chat with up to 2-3 clarifying questions |
| Frontend store | `lib/quick-mode-store.ts` | `lib/conv-store.ts` |
| Endpoint hit | `POST /api/quick` | `POST /api/conversation` (with optional pre-round to `/api/conversation/questions`) |
| Backend call | `planCart({ query: intent })` | `planCart({ query, parameters, sessionId })` |
| Clarifying questions | None | Generated locally (Gemini → fallback bank), NOT via SmartCart `/v1/cart/chat` |
| Output | Vibe-themed cart preview + total + ETA | Streamed messages + cart line items inside the modal |

Both modes ultimately call `lib/services/smartcart.ts` → `POST {SMARTCART_API_URL}/v1/cart/plan`. The conversational flow gathers clarifying answers in `parameters` and sends them along on the *final* plan call — SmartCart stays stateless from this app's perspective.

The clarifier (`app/api/conversation/questions/route.ts`) routes the user query to one of ten topic buckets (baby / pet / cleaning / personal-care / healthcare / beverage / breakfast / party / movie / meal / default). With `GEMINI_API_KEY` set it generates 2-3 query-specific questions through Gemini in JSON mode; otherwise it serves a hand-curated bank.

### "Buy it again, right on time"
`components/predictive-section.tsx` reads `/api/predictions/eligible`, which scans the catalog for products whose name / brand / tags match a recurrence keyword in `lib/recurring.ts`:

| Keyword | Cadence |
|---|---|
| milk | every 2 days |
| bisleri / mineral water | every 2 days |
| bread | every 3 days |
| badam / kool badam | every 3 days |
| egg | every 4 days |
| maggi / noodle | every 5 days |
| butter | every 7 days |
| corn flakes / kellogg | every 10 days |

On first load it auto-seeds 4 prior orders per match in `prediction-store.ts` so the demo immediately shows "Due now / Due tomorrow" cards. Per card the user can add to cart (records a synthetic order), skip the cycle, snooze 24h, or permanently remove from prediction.

---

## Data model

```prisma
model Product {
  id           String   @id
  name         String
  brand        String
  category     String
  subCategory  String
  size         String
  price        Int
  mrp          Int
  rating       Float
  ratingCount  Int
  deliveryMin  Int
  status       String          // "Available" / etc.
  tags         String[]
  rankScore    Float    @default(0)
  img          String

  @@index([category])
  @@index([brand])
  @@index([subCategory])
}

model Order {
  id            String   @id @default(cuid())
  userId        String   @default("garv")
  items         Json            // OrderItemSnapshot[]
  subtotal      Int
  savings       Int
  total         Int
  paymentMethod String           // "upi" | "card" | "cod"
  zoneCode      String   @default("110001")
  deliveryMin   Int
  address       Json
  status        String   @default("placed")
  createdAt     DateTime @default(now())

  @@index([userId, createdAt(sort: Desc)])
}
```

This is *not* the same Postgres as `search-service`. The two databases are independent — SmartCart returns its own product fields inline (name, image, price, rating, brand, sub-category) so the website never has to join SmartCart's `productId` against its own catalog.

---

## API routes

All under `app/api/*`. Schemas live in `lib/schemas.ts` (Zod).

| Method | Path | Body / query | Returns |
|---|---|---|---|
| GET | `/api/categories` | — | `CategoryDTO[]` (name, slug, count, icon, bg, hasArt) |
| GET | `/api/products?q&category&sort&page&pageSize` | `ProductsQuerySchema` | `{ items, total, page, pageSize }` |
| GET | `/api/products/[id]` | — | `{ product, related }` |
| GET | `/api/products/by-ids?ids=a,b,c` | up to 200 ids | `{ items: ProductDTO[] }` |
| GET | `/api/orders` | — | recent orders for user `"garv"` |
| POST | `/api/orders` | `CreateOrderSchema` | `{ orderId, deliveryMin }` |
| GET | `/api/predictions/eligible` | — | `{ items: { product, intervalDays }[] }` |
| POST | `/api/quick` | `QuickRequestSchema` (intent / groupSize / zoneCode) | Cart preview shape (`vibe_category`, `shopping_list`, `cart`) |
| POST | `/api/conversation` | `{ query, parameters?, sessionId? }` | `{ status, reply, questions, items, sessionId? }` |
| POST | `/api/conversation/questions` | `{ query }` | `{ questions: ConvQuestionDTO[], usedFallback }` |

`/api/quick` and `/api/conversation` proxy through to SmartCart — so they need `SMARTCART_API_URL` set and (if SmartCart has auth on) `SMARTCART_API_KEY`.

---

## Environment

`.env.local`:

| Var | Required | Notes |
|---|---|---|
| `DATABASE_URL` | yes | Postgres for catalog + orders |
| `SMARTCART_API_URL` | yes | Base URL of `search-service`, default `http://localhost:3001` |
| `SMARTCART_API_KEY` | when SmartCart has auth on | Bearer token forwarded on every `/v1/cart/plan` call |
| `GEMINI_API_KEY` | recommended | Powers `/api/conversation/questions`. Without it, falls back to a topic-routed question bank |
| `GOOGLE_API_KEY` | alternate | Accepted as a synonym for `GEMINI_API_KEY` |
| `GEMINI_MODEL_ID` | no | Defaults to `gemini-2.0-flash` |
| `NODE_ENV` | no | Controls Prisma log verbosity |

---

## Running locally

### Prerequisites
- Node 20+, npm 10+
- A Postgres 14+ database for this app's catalog + orders
- A reachable `search-service` instance (see [`../search-service/README.md`](../search-service/README.md)) — the AI cart modes won't work without it, but everything else (browse / cart / orders / predictions) does.

### First time

```bash
# from website/
npm install

# create .env.local with the vars above, then
npm run db:migrate      # applies prisma/migrations
npm run seed            # tsx scripts/seed-db.ts — loads seed.ts (~8 MB) into Product
```

### Day-to-day

```bash
npm run dev             # next dev — http://localhost:3000
npm run build           # next build
npm run start           # next start (production server)
npm run lint            # next lint
npm run db:generate     # prisma generate
npm run db:studio       # prisma studio
```

### Port note
Both this app and `search-service` default to `PORT=3000`. The simplest fix is to publish SmartCart on `3001` and set `SMARTCART_API_URL=http://localhost:3001` here.

---

## Things worth knowing

- **Force-dynamic everywhere data is read** — `app/page.tsx`, `app/product/[id]/page.tsx`, `app/category/[slug]/page.tsx`, `app/orders/page.tsx` all set `export const dynamic = "force-dynamic"`. No static caching of catalog data.
- **Single hard-coded user** — every order is owned by `userId="garv"` with the Connaught Place 110001 address. There is no auth layer; this is a demo.
- **Product images** — many cards render real images from `cdn.zeptonow.com` (whitelisted in `next.config.ts`). When a product has no usable image, `lib/theme.ts → tileTint()` returns a keyword-derived bg/fg pair (e.g. cola → `#f4e2dc / #9c2f1c`, lemon → `#e9f2cf / #566b14`) and the tile renders as a tinted placeholder.
- **Hydration-safe cart** — `useCart` is persisted to `localStorage` under `ap_cart_v1`. Components that read it use `hooks/use-hydrated-cart.ts` so the header count doesn't flicker between SSR and CSR.
- **No SmartCart `/v1/cart/chat`** — clarifying questions are produced by *this app* (Gemini → fallback bank). The website always calls SmartCart's stateless `/v1/cart/plan` once it has the answers, packaged as `parameters`.
- **Tailwind v4** — config-less; styles flow through PostCSS via `@tailwindcss/postcss`. The Amazon-skinned palette and CSS variables live in `app/globals.css`.
- **Path alias** — `@/*` maps to the project root (`tsconfig.json`), so imports look like `@/components/header`, `@/lib/db`.