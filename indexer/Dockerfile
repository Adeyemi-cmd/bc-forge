FROM node:20-alpine

# Install openssl for Prisma engine support on Alpine. libc6-compat provides
# the glibc loader shim some Prisma native binaries probe for; harmless on
# musl and required for the engines to start on both amd64 and arm64 (#1049).
# node:20-alpine itself is multi-arch, so Buildx can produce linux/amd64 and
# linux/arm64 from this Dockerfile under one manifest list.
RUN apk add --no-cache openssl libc6-compat

# This Dockerfile uses the repository root as its build context
# (docker build -f indexer/Dockerfile .), because @bc-forge/indexer depends
# on the @bc-forge/sdk workspace via a symlink. Building with `indexer/` as
# the context cannot resolve that dependency from the npm registry (the SDK
# is not published), so the workspace manifests are copied first and
# installed with the root lockfile for reproducible, multi-arch builds.
WORKDIR /app

# Copy workspace manifests first for layer caching.
COPY package.json package-lock.json ./
COPY sdk/package.json ./sdk/package.json
COPY cli/package.json ./cli/package.json
COPY react/package.json ./react/package.json
COPY indexer/package.json ./indexer/package.json

# Install indexer workspace dependencies (resolves @bc-forge/sdk via the
# local symlink, not the registry).
RUN npm ci --workspace @bc-forge/indexer --include-workspace-root

# Copy sources needed for Prisma generation and TypeScript builds.
COPY sdk ./sdk
COPY indexer/prisma ./indexer/prisma
COPY indexer/tsconfig.json ./indexer/tsconfig.json
COPY indexer/src ./indexer/src

WORKDIR /app/indexer

# Generate Prisma client. The schema's binaryTargets vendor engines for both
# linux/amd64 and linux/arm64, and this RUN executes per-platform under
# Buildx/QEMU, so each arch image carries a working native engine (#1049).
RUN npx prisma generate

# Build TypeScript source
RUN npm run build

# Expose default HTTP port
EXPOSE 3000

# Default port environment variable
ENV PORT=3000

# Start compiled server
CMD ["npm", "start"]
