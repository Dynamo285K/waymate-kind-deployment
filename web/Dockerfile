# Build context: the spolujazda-sketch repo root (the web build type-checks against apps/api and packages/shared).
FROM oven/bun:1.3.13-slim AS build

WORKDIR /app

COPY package.json bun.lock tsconfig.base.json ./
COPY apps/api/package.json ./apps/api/
COPY apps/web/package.json ./apps/web/
COPY packages/shared/package.json ./packages/shared/
COPY packages/db/package.json ./packages/db/
COPY e2e/package.json ./e2e/

RUN bun install --frozen-lockfile

COPY apps/api apps/api
COPY apps/web apps/web
COPY packages packages

# Vite bakes VITE_* variables into the bundle at build time.
ARG VITE_MAPBOX_ACCESS_TOKEN=pk.local_dummy_token
RUN bun run --cwd apps/web build

# Non-root nginx (listens on 8080). The server config comes from a ConfigMap in the cluster.
FROM nginxinc/nginx-unprivileged:1.29-alpine

COPY --from=build /app/apps/web/dist /usr/share/nginx/html
