# Next.js production image (fixed: native build tools + force build of lightningcss)
FROM node:20-bullseye AS deps
WORKDIR /app

# Install native build tools needed by native modules (lightningcss, etc.)
RUN apt-get update && apt-get install -y \
    build-essential \
    python3 \
    make \
    g++ \
    libc6 \
    wget \
    ca-certificates \
  && rm -rf /var/lib/apt/lists/*

# Copy package files and install dependencies (with unsafe-perm for native installs)
COPY client/package.json client/package-lock.json* ./
# Ensure any native modules are compiled here
RUN npm ci --unsafe-perm --prefer-offline --no-audit \
  && npm rebuild lightningcss --build-from-source --update-binary || true

FROM node:20-bullseye AS builder
WORKDIR /app
ENV NODE_ENV=production

# Copy dependencies (already built with native bindings in deps)
COPY --from=deps /app/node_modules ./node_modules

# Copy project files
COPY client/ ./

# Safety: attempt rebuild/update binary again in builder env (covers ABI/platform differences)
RUN npm rebuild lightningcss --build-from-source --update-binary || npm rebuild || true

# Build Next.js
RUN npm run build

FROM node:20-bullseye AS runner
WORKDIR /app
ENV NODE_ENV=production

# Create non-root user
RUN groupadd -r nodegrp && useradd -r -g nodegrp nodeusr

# Copy build output and runtime files
COPY --from=builder /app/.next ./.next
COPY --from=builder /app/public ./public
COPY --from=builder /app/package.json ./package.json
COPY --from=builder /app/node_modules ./node_modules

# Prune devDependencies to reduce final image size
RUN npm prune --omit=dev || true

ENV PORT=3000
EXPOSE 3000

# Healthcheck
HEALTHCHECK --interval=30s --timeout=5s --start-period=20s --retries=3 \
  CMD wget -qO- http://127.0.0.1:3000/ || exit 1

USER nodeusr
CMD ["npm", "run", "start"]
