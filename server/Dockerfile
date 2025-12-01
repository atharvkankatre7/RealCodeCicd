# Server image — build if needed, run otherwise
# Stage: deps (install node modules / compile native modules)
FROM node:20-bullseye AS deps
WORKDIR /app

# Install native build tools required by node-gyp / native modules
RUN apt-get update && apt-get install -y \
    build-essential \
    python3 \
    make \
    g++ \
    ca-certificates \
  && rm -rf /var/lib/apt/lists/*

# Copy package files and install production deps (unsafe-perm for native installs)
COPY server/package.json server/package-lock.json* ./
RUN npm ci --unsafe-perm --prefer-offline --no-audit

# Stage: builder (optional build step)
FROM node:20-bullseye AS builder
WORKDIR /app
ENV NODE_ENV=production

# Copy installed modules from deps
COPY --from=deps /app/node_modules ./node_modules

# Copy source files
COPY server/ ./

# Try to run a build if it exists. The "|| true" prevents failure if no build script is present.
# If your server *requires* a build (TypeScript), ensure package.json has a "build" script.
RUN if npm run | grep -q " build"; then npm run build; else echo "no build script, skipping"; fi

# Stage: runtime (smaller final image)
FROM node:20-bullseye AS runtime
WORKDIR /app
ENV NODE_ENV=production

# Create non-root user
RUN groupadd -r nodegrp && useradd -r -g nodegrp nodeusr

# Copy runtime node_modules and app files
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app ./

# If your build outputs to "dist", the files will already be present (copied above).
# Ensure your package.json includes a "start" script that runs the server (recommended).
ENV PORT=5002
EXPOSE 5002

# Healthcheck (increase start-period if your boot is slow)
HEALTHCHECK --interval=30s --timeout=5s --start-period=60s --retries=3 \
  CMD curl -fsS http://127.0.0.1:5002/health || exit 1

USER nodeusr

# Default start. This expects a "start" script in server/package.json.
# If your package.json uses a different script, replace this with the correct command,
# e.g. ["node","src/index.js"] or ["npm","run","dev"]
CMD ["npm", "run", "start"]

