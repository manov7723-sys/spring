# syntax=docker/dockerfile:1

############################
# Stage 1 — build the Vite bundle
############################
FROM --platform=linux/amd64 node:22-slim AS build

WORKDIR /app

# pnpm pinned to the major matching lockfileVersion 9.0
RUN npm install -g pnpm@9

# Install deps first for layer caching. --include=optional-equivalent:
# pnpm installs optional deps (per-platform native binaries like esbuild/SWC)
# by default; they are materialised into node_modules at build time.
COPY package.json pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile

# VITE_* values are baked into the bundle at build time (framework-prefixed
# vars only — runtime secrets must NEVER be passed here).
ARG VITE_BACKEND_URL
ARG VITE_API_URL
ARG VITE_FRONTEND_URL
ENV VITE_BACKEND_URL=${VITE_BACKEND_URL} \
    VITE_API_URL=${VITE_API_URL} \
    VITE_FRONTEND_URL=${VITE_FRONTEND_URL}

COPY . .
RUN pnpm run build

############################
# Stage 2 — static file server
############################
FROM --platform=linux/amd64 node:22-slim

ENV NODE_ENV=production \
    PORT=8080

# `serve` is materialised into /usr/local at build time — nothing is
# downloaded at runtime (root fs is read-only; only /tmp is writable).
RUN npm install -g serve@14 && npm cache clean --force

WORKDIR /app
COPY --from=build /app/dist ./dist

# Numeric uid required by Kubernetes runAsNonRoot
USER 1000

EXPOSE 8080

# Base image is slim (no curl/wget) — probe with node itself.
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD node -e "fetch('http://127.0.0.1:'+(process.env.PORT||8080)).then(r=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"

# sh -c + exec so ${PORT} from the pod spec is honoured (never hardcode).
CMD ["sh", "-c", "exec serve -s dist -l tcp://0.0.0.0:${PORT}"]
