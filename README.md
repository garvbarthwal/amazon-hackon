# SmartCart API — search-service

The cart-planning engine behind Amazon Picks. Takes a natural-language query (`"movie night for 4 people"`, `"Diwali decorations"`, `"Pav Bhaji"`) and returns a structured, audited cart driven by a deterministic 8-stage retrieval pipeline over a Requirement Graph.

This service is one half of the [`amazon-hackon-main`](..) repo. The [`website/`](../website) calls it over HTTP whenever a user uses Quick Mode or the Conversational hero. SmartCart owns its own Postgres + Redis and never reaches back into the website's database.

- Versioned HTTP API at `/v1/*` — full contract in [`../website/API.md`](../website/API.md)
- Docker-compose deployment for app + Postgres (pgvector/pg16) + Redis
- Bedrock or Gemini for generation; Gemini-only for embeddings (768-d)

---

## Tech stack

| Layer | Choice |
|---|---|
| Runtime | Node 20 (slim Debian image), ESM-only (`"type": "module"`) |
| HTTP | Express 4 |
| Database | Postgres 16 with `vector` (pgvector) and `pg_trgm` extensions |
| ORM | Prisma 5 (the website is Prisma 7; the two are independent) |
| Cache | Redis 7 (sessions, request replay, planner cache) |
| LLM (generation) | AWS Bedrock — Anthropic Claude / Nova via Converse API, OR Gemini 2.5 Flash Lite. Switch via `LLM_PROVIDER` env |
| LLM (embeddings) | Always Gemini `gemini-embedding-001`, 768-d (matches the seeded catalog) |
| API docs | OpenAPI 3 hand-written; served at `/v1/docs` (Swagger UI) and `/v1/openapi.json` |
| TS | TypeScript 5.6, `module: NodeNext` (imports use `.js` suffixes from `.ts` sources) |

---

## Folder structure

```
search-service/
├── README.md                       ← this file
├── Dockerfile                      Multi-stage build (compile TS → run dist/)
├── docker-compose.yml              app + postgres (pgvector/pg16) + redis (7-alpine)
├── package.json                    name: "smartcart-api" v5.0.0
├── tsconfig.json                   ES2022 / NodeNext / strict / outDir=dist
├── .dockerignore
├── .gitignore
│
├── prisma/
│   ├── schema.prisma               Product, SubCategory, MissionKB,
│   │                               RequirementCache, Brand, CartRequestLog
│   └── migrations/
│       ├── 20260615000000_init/                ← initial schema
│       ├── 20260615120000_v2_mission_kb/       ← MissionKB table
│       ├── 20260615180000_v35_brand_cache/     ← Brand stats / brandScore
│       ├── 20260616000000_v45_domains/         ← Product.domain column
│       ├── 20260616120000_v5_cart_request_log/ ← per-request audit log
│       └── migration_lock.toml
│
└── src/
    ├── server.ts                   Express bootstrap, /v1/health, Swagger UI
    │
    ├── api/                        HTTP layer — no business logic here
    │   ├── cart.controller.ts      POST /v1/cart/plan, GET /v1/cart/status/:id
    │   ├── openapi.ts              Hand-written OpenAPI 3 spec object
    │   └── openapi.types.ts        Helper types for the spec
    │
    ├── lib/
    │   ├── infra/                  Cross-cutting infrastructure
    │   │   ├── db.ts               Shared Prisma client singleton
    │   │   ├── redis.ts            Redis client + cacheGet/cacheSet helpers
    │   │   ├── cache.ts            Postgres-backed RequirementCache (30-day TTL)
    │   │   ├── llm.ts              Provider switch (Bedrock vs Gemini for gen)
    │   │   ├── gemini.ts           embedTexts/embedOne/toPgVector — embeddings
    │   │   └── providers/
    │   │       ├── bedrock.ts      Bedrock Converse API w/ tool-call forcing
    │   │       └── gemini.ts       Gemini generateContent in JSON mode
    │   │
    │   ├── middleware/
    │   │   ├── auth.ts             Bearer-token gate (SMARTCART_API_KEY)
    │   │   └── requestId.ts        UUID per HTTP request + JSON access log
    │   │
    │   ├── pipeline/               The 8-stage cart-planning pipeline
    │   │   ├── classifier.ts       Stage 1 — query → queryType (LLM, JSON)
    │   │   ├── planner.ts          Stage 2 — Requirement Graph (LLM, JSON)
    │   │   ├── domains.ts          ProductDomain rules + festival keywords
    │   │   ├── constraints.ts      Per-requirement allow/block + name regex
    │   │   ├── resolver.ts         6-tier strict retrieval chain
    │   │   ├── coverage.ts         fulfilled / total_required ratio
    │   │   ├── substitution.ts     LLM picks subs from same parent category
    │   │   ├── cart.ts             Composer (essentials/recommended/premium)
    │   │   ├── auditor.ts          Strict LLM reviewer w/ 2 retries
    │   │   └── orchestrator.ts     Stitches all stages into one pipeline run
    │   │
    │   ├── services/
    │   │   ├── cartPlanning.service.ts   Service entry — drives the pipeline
    │   │   └── responseMapper.ts          Internal → public CartResponse mapping
    │   │
    │   └── types/
    │       └── cart.types.ts        Frozen public response shape
    │
    └── scripts/                    One-shot CLI scripts (after `npm run build`)
        ├── seedProducts.ts          Loads catalog JSON → Product
        ├── seedCategories.ts        Builds SubCategory vocabulary + embeddings
        ├── seedMissions.ts          Loads MissionKB (dishes / missions / festivals)
        ├── extractBrands.ts         Computes Brand stats / brandScore
        ├── extractDomains.ts        Tags every Product with a domain
        ├── embedProducts.ts         Generates 768-d Gemini embeddings
        └── eval.ts                  Offline accuracy harness
```

---

## What it does (the pipeline)

`POST /v1/cart/plan` runs the same 8-stage chain every time. Source: [`src/lib/pipeline/orchestrator.ts`](src/lib/pipeline/orchestrator.ts).

```
{ query, parameters }
        │
        ▼
  1. Classify          classifier.ts
        │              queryType ∈ { product | brand | ingredient | dish |
        │                            mission | festival | category | unknown }
        │              + confidence + slug + brand + ingredient + categories
        ▼
  2. Plan              planner.ts
        │              Build a Requirement[] (essentials / recommended / premium).
        │              Each has type (exact_product / brand / name / subcategory),
        │              nameMatch, hints, brand, priority, quantity. Cached in
        │              RequirementCache for 30 days (Postgres).
        ▼
  3. Constrain         constraints.ts + domains.ts
        │              Per-requirement: allowed/blocked domains, festival key,
        │              and parameter-driven nameBoost / nameExclude regexes
        │              (taste, vegetarian, vegan, glutenFree, dairyFree,
        │              lowSugar, highProtein, organic, healthy, …).
        ▼
  4. Resolve           resolver.ts — 6-tier strict chain
        │                1. Exact product (nameMatch + optional sub-cat fence)
        │                2. Brand (exact, then pg_trgm fuzzy)
        │                3. Sub-category (hints ∩ Product.subCategory)
        │                4. Category broaden (parent of hints)
        │                5. Synonym (pg_trgm on SubCategory.name)
        │                6. Embedding (pgvector cosine on synthetic text)
        │              First tier with hits wins; constraint filters BEFORE rank.
        │              Score = 0.30·rating + 0.25·log(reviews) + 0.20·popularity
        │                      + 0.25·brandScore + bonuses.
        │              Critical invariant: nameMatch miss → []  (does NOT fall
        │              through to embedding). Atta never replaces Pav.
        ▼
  5. Coverage          coverage.ts
        │              cov = fulfilled_required / total_required
        │              ≥ 0.90 → pass (run substitution for any gaps)
        │              <  0.90 → needs_user_input  (caller's responsibility)
        ▼ (only when there are gaps)
  6. Substitute        substitution.ts
        │              LLM picks 1-2 in-category subs per missing essential
        │              (e.g. Pav → Bread Roll / Burger Bun).
        │              "diaper / formula / baby food / incontinence" are
        │              never auto-substituted.
        ▼
  7. Compose           cart.ts
        │              Top fresh per requirement, no cross-tier dupes.
        │              recommended = top 2 by score.
        │              premium = top 3 by composite:
        │                0.40·complementary + 0.30·rating
        │                + 0.20·mission_relevance + 0.10·margin_proxy
        ▼ (loops back to 7 up to 2 retries)
  8. Audit             auditor.ts
        │              Strict LLM reviewer flags productIds to remove for:
        │              domain mismatch, off-topic items, wrong-brand on a brand
        │              query, festival rivalry (Christmas in a Diwali cart),
        │              baby-food masquerading as adult food. NEVER suggests
        │              replacements — orchestrator drops the bans, recomposes,
        │              re-audits. Up to 2 retries before shipping as
        │              `partial_success`.
        ▼
   final CartResponse   responseMapper.ts → frozen v5 envelope
```

Every successful response is **persisted to `CartRequestLog`** (forever) and **cached in Redis** at `request:{requestId}` for 24h. `GET /v1/cart/status/:requestId` replays from either.

`?debug=1` (or header `X-Debug: 1`) attaches a `debug` block with classifier confidence, every resolverPath taken, every constraint that fired, all substitutions, the auditor verdict, and a notes log.

---

## HTTP API

| Method | Path | Auth | Purpose |
|---|---|---|---|
| GET  | `/v1/health` | public | Liveness — `{ status, version, uptimeSec, llmProvider }` |
| GET  | `/v1/openapi.json` | public | OpenAPI 3 spec |
| GET  | `/v1/docs` | public | Swagger UI |
| POST | `/v1/cart/plan` | bearer | Single-shot, stateless plan |
| GET  | `/v1/cart/status/:requestId` | bearer | Replay a stored response (24h Redis / forever Postgres) |

The OpenAPI spec ([`src/api/openapi.ts`](src/api/openapi.ts)) also documents `POST /v1/cart/chat` for multi-turn flows. The actual website bundled in this repo does **not** call `/v1/cart/chat` — it generates clarifying questions on its own and sends the answered query to `/v1/cart/plan` once. The chat endpoint remains documented for other callers.

Bearer auth is **off** when `SMARTCART_API_KEY` is unset — the boot log says `Auth: DISABLED`. Set it to a real string and clients must send `Authorization: Bearer <key>` on every `/v1/cart/*` call. `/v1/health`, `/v1/openapi.json`, `/v1/docs` are always public.

Full request/response contract, error codes, and client recipes (curl / Node / Python / multi-turn): [`../website/API.md`](../website/API.md).

---

## Data model (Prisma)

All models live in [`prisma/schema.prisma`](prisma/schema.prisma).

| Model | Role |
|---|---|
| `Product` | Catalog row — `id`, `name`, `image`, `price`, `rating`, `reviews`, `quantity`, `subCategory`, `category`, `brand`, `domain`, `inStock`, `syntheticText`. The `embedding vector(768)` column is added via raw SQL migration since Prisma has no native vector type. |
| `SubCategory` | Vocabulary — `name`, `category`, `description`, `productCount`, plus a 768-d `embedding` for synonym-style lookups via raw SQL. |
| `MissionKB` | Source of truth for *what does this query need?* — `slug` (PK) keyed entries with type=`mission` or `dish`, free-form `aliases`, plus JSON `essentials / recommended / premium` arrays. Loaded by `seedMissions`. |
| `RequirementCache` | Per-query planner output — `(query, queryType)` unique key, 30-day TTL enforced at read time. |
| `Brand` | Aggregated brand stats — `productCount`, `avgRating`, `totalReviews`, `brandScore` (0..1 normalized for ranking). |
| `CartRequestLog` | One row per `/v1/cart/plan` call: `requestId` (PK), full `response` JSON, latency, status, coverage, optional `sessionId`. Supports `/v1/cart/status/:requestId`. |

The two `vector(768)` columns + the `pg_trgm` similarity operators all require those Postgres extensions — both are shipped by the `pgvector/pgvector:pg16` image used in `docker-compose.yml`.

---

## Configuration

`.env` (loaded by `dotenv/config` from `server.ts`):

| Var | Required | Notes |
|---|---|---|
| `DATABASE_URL` | yes | Postgres connection string. Compose overrides this to the in-cluster `postgres` service. |
| `REDIS_URL` | yes | Compose overrides to `redis://redis:6379`. Defaults to `redis://localhost:6380` outside compose. |
| `LLM_PROVIDER` | yes | `bedrock` (default) or `gemini`. |
| `GEMINI_API_KEY` | always | Needed for embeddings even when `LLM_PROVIDER=bedrock`. |
| `GEMINI_LLM_MODEL` | no | Defaults to `gemini-2.5-flash-lite`. |
| `GEMINI_EMBED_MODEL` | no | Defaults to `gemini-embedding-001`. |
| `AWS_BEARER_TOKEN_BEDROCK` | if Bedrock | Bedrock long-term API key (picked up by the SDK). |
| `BEDROCK_REGION` | no | Defaults to `us-east-1`. |
| `BEDROCK_MODEL_ID` | no | Defaults to `us.amazon.nova-lite-v1:0`. |
| `SMARTCART_API_KEY` | recommended | Bearer token for `/v1/cart/*`. Empty = auth disabled (dev only). |
| `PORT` | no | Defaults to `3000`. |

---

## Running it

### A. Docker compose (recommended)

```bash
cp .env.example .env
# fill GEMINI_API_KEY, AWS_BEARER_TOKEN_BEDROCK (if Bedrock), SMARTCART_API_KEY

docker compose up -d --build
# app waits for pg + redis health, runs `prisma migrate deploy`, starts server
curl http://localhost:3000/v1/health
```

Open Swagger UI at <http://localhost:3000/v1/docs>.

### B. Local Node (Postgres + Redis already running)

```bash
npm install
npx prisma migrate deploy
npm run build       # prisma generate + tsc
npm start           # node dist/server.js
```

### Seed data (one-time, after the stack is up)

The seed scripts run *inside* the `app` container so they share the in-cluster DB hostname.

```bash
docker compose exec app npm run seed:products      # raw catalog → Product
docker compose exec app npm run embed:products     # Gemini 768-d → Product.embedding
docker compose exec app npm run seed:categories    # SubCategory vocab + embeddings
docker compose exec app npm run seed:missions      # MissionKB
docker compose exec app npm run extract:brands     # Brand stats + brandScore
docker compose exec app npm run extract:domains    # tags Product.domain
```

Order matters only loosely: products must exist before embeddings/brands/domains; missions and sub-categories are independent.

### Day-to-day docker compose

```bash
docker compose logs -f app          # tail logs (one JSON line per request)
docker compose restart app          # after .env changes
docker compose down                 # stop, keep volumes
docker compose down -v              # stop + wipe pg/redis volumes
```

---

## npm scripts

```
npm run build            # prisma generate + tsc → dist/
npm start                # node dist/server.js
npm run db:migrate       # prisma migrate deploy
npm run db:generate      # prisma generate

npm run seed:products    # tableConvert.com_*.json → Product
npm run seed:categories  # SubCategory vocab + embeddings
npm run seed:missions    # MissionKB

npm run embed:products   # Gemini 768-d embeddings → Product.embedding
npm run extract:brands   # Brand aggregates + brandScore
npm run extract:domains  # Product.domain via rules + LLM fallback
```

Every script targets the *compiled* `dist/scripts/*.js` — run `npm run build` first.

---

## Putting it behind a domain

The app listens on `0.0.0.0:3000` inside the container, exposed as `${APP_PORT:-3000}` on the host. To put it behind TLS:

1. **Reverse proxy** (Caddy / Nginx / Traefik) on the host, terminating TLS and forwarding `your.domain → http://localhost:3000`. Caddy:

   ```caddyfile
   api.example.com {
     reverse_proxy localhost:3000
   }
   ```

2. **DNS** — point an A record at the host's public IP.
3. **API key** — set `SMARTCART_API_KEY` in `.env` and restart `app`. Clients send `Authorization: Bearer <key>`.

The server doesn't need to know its public hostname — every route is relative.

---

## Operational notes

- **Sessions** live in Redis at `session:{sessionId}` with a 30-minute idle TTL.
- **Response cache** lives at `request:{requestId}` for 24 hours.
- **Planner cache** (Postgres `RequirementCache`) is 30 days — repeat queries skip the planner LLM call entirely.
- **Logs** are one JSON line per HTTP request:
  ```
  {"ts":"...","requestId":"...","method":"POST","path":"/v1/cart/plan","status":200,"latencyMs":1704}
  ```
  Plus a `[cart-service]` line per pipeline run with `queryType`, `coverage`, `latency`.
- **Latency budget** today: 1.5–3s per cold call; cached repeats are <100ms.
- **CORS** is intentionally not wired in. Browsers calling from another origin will be rejected — proxy through your own backend, or add `cors()` middleware to `server.ts` if you really want direct browser calls.
- **Migrations are idempotent** and run automatically on container start (`prisma migrate deploy`).

---

## Where to change things

| Concern | File |
|---|---|
| HTTP routes | `src/api/cart.controller.ts` |
| Service entry point | `src/lib/services/cartPlanning.service.ts` |
| Frozen response shape | `src/lib/types/cart.types.ts` |
| Internal → public mapping | `src/lib/services/responseMapper.ts` |
| Pipeline orchestration | `src/lib/pipeline/orchestrator.ts` |
| Per-stage logic | `src/lib/pipeline/{classifier,planner,resolver,…}.ts` |
| Bearer auth | `src/lib/middleware/auth.ts` |
| Request ID + access log | `src/lib/middleware/requestId.ts` |
| OpenAPI spec | `src/api/openapi.ts` |
| Redis client + helpers | `src/lib/infra/redis.ts` |
| Provider switch | `src/lib/infra/llm.ts` (and `providers/{bedrock,gemini}.ts`) |
| Embeddings | `src/lib/infra/gemini.ts` |