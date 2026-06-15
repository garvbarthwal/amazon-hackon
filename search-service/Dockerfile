# ── Build stage ─────────────────────────────────────────────────────────
FROM node:20-slim AS build

# OpenSSL is required by Prisma engines on slim images.
RUN apt-get update && apt-get install -y --no-install-recommends openssl ca-certificates \
  && rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY package.json package-lock.json* ./
RUN npm ci || npm install

COPY tsconfig.json ./
COPY prisma ./prisma
COPY src ./src

RUN npx prisma generate
RUN npx tsc

# ── Runtime stage ───────────────────────────────────────────────────────
FROM node:20-slim AS runtime

RUN apt-get update && apt-get install -y --no-install-recommends openssl ca-certificates \
  && rm -rf /var/lib/apt/lists/*

WORKDIR /app
ENV NODE_ENV=production

COPY package.json package-lock.json* ./
RUN npm ci --omit=dev || npm install --omit=dev

# Prisma client + compiled JS + migrations + seed data
COPY --from=build /app/node_modules/.prisma ./node_modules/.prisma
COPY --from=build /app/node_modules/@prisma ./node_modules/@prisma
COPY --from=build /app/dist ./dist
COPY prisma ./prisma
COPY tableConvert.com_epc695.json ./tableConvert.com_epc695.json

# Default port; can be overridden via env.
EXPOSE 3000

# Apply pending migrations on startup, then launch the server.
CMD ["sh", "-c", "npx prisma migrate deploy && node dist/server.js"]
