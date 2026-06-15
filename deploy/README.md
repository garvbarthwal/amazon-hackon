# SmartCart API — Deploy bundle

Self-contained docker-compose deployment of the SmartCart backend. Spins up the API, Postgres (with `pgvector` + `pg_trgm`), and Redis on a single host.

## What's in here

- `src/` — backend TypeScript (no frontend)
- `prisma/` — schema + migrations (vector columns added via SQL migrations)
- `tableConvert.com_epc695.json` — raw product catalog used by `seed:products`
- `Dockerfile` — multi-stage build, compiles TS → `dist/`, runs `node dist/server.js`
- `docker-compose.yml` — `app` + `postgres` (pgvector/pg16) + `redis` (7-alpine)
- `.env.example` — copy to `.env` and fill in real keys

## First-time setup

```bash
cp .env.example .env
# edit .env: set GEMINI_API_KEY, AWS_BEARER_TOKEN_BEDROCK, SMARTCART_API_KEY

docker compose up -d --build
# app waits for pg + redis health, then runs `prisma migrate deploy` and starts the server.
```

Health check: `curl http://localhost:3000/v1/health`

## Seed data (one-time, after the stack is up)

The seed scripts run inside the `app` container so they share the in-cluster DB hostname.

```bash
# Products (reads tableConvert.com_epc695.json)
docker compose exec app npm run seed:products

# Embeddings (calls Gemini — requires GEMINI_API_KEY)
docker compose exec app npm run embed:products

# Sub-category vocabulary
docker compose exec app npm run seed:categories

# Mission / dish knowledge base
docker compose exec app npm run seed:missions

# Brand stats + domain extraction
docker compose exec app npm run extract:brands
docker compose exec app npm run extract:domains
```

## Day-to-day

```bash
docker compose logs -f app          # tail logs
docker compose restart app          # restart after .env changes
docker compose down                 # stop everything (data persists in volumes)
docker compose down -v              # stop + wipe pg/redis volumes
```

## Adding a domain later

The app listens on `0.0.0.0:3000` inside the container, exposed as `${APP_PORT:-3000}` on the host. To put it behind a domain:

1. **Reverse proxy** (Caddy / Nginx / Traefik) on the host, terminating TLS and forwarding `your.domain → http://localhost:3000`. Caddy example:
   ```
   api.example.com {
     reverse_proxy localhost:3000
   }
   ```
2. **DNS** — point an A record for `api.example.com` at the host's public IP.
3. **API key** — set `SMARTCART_API_KEY` in `.env` and restart `app`. Clients then send `Authorization: Bearer <key>`.

The server doesn't need to know the domain — all routes are relative.

## Endpoints

- `GET  /v1/health` — public, no auth
- `GET  /v1/openapi.json` — public, full spec
- `GET  /v1/docs` — Swagger UI
- `POST /v1/cart/plan`, `POST /v1/cart/chat`, `GET /v1/cart/status/:requestId` — Bearer-auth gated when `SMARTCART_API_KEY` is set

## Env reference

| Var | Required | Notes |
|---|---|---|
| `DATABASE_URL` | yes | Compose overrides this to hit the `postgres` service. |
| `REDIS_URL` | yes | Compose overrides this to `redis://redis:6379`. |
| `LLM_PROVIDER` | yes | `bedrock` or `gemini`. |
| `GEMINI_API_KEY` | always | Needed for embeddings even on Bedrock. |
| `AWS_BEARER_TOKEN_BEDROCK` | if `LLM_PROVIDER=bedrock` | Bedrock API key. |
| `BEDROCK_REGION`, `BEDROCK_MODEL_ID`, `BEDROCK_EMBED_MODEL_ID` | if Bedrock | Defaults provided. |
| `SMARTCART_API_KEY` | recommended | Bearer-token auth. Empty = auth disabled (dev only). |
| `PORT` | no | Defaults to `3000`. |

## Notes

- The Prisma schema needs `vector` and `pg_trgm` extensions; the `pgvector/pgvector:pg16` image ships both.
- Migrations run automatically on container start (`prisma migrate deploy`). Idempotent.
- TS is compiled at build time; runtime is plain `node dist/server.js`.
