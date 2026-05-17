# Scaffold a Dockerfile

Pure file generation — no API calls. Produces a Dockerfile, `.dockerignore`, and optional healthcheck route that play well with Dokploy's defaults (Traefik routing, `EXPOSE 3000` convention, healthcheck-aware ingress).

## When to use

- The user wants to deploy a new app to Dokploy and doesn't have a Dockerfile yet.
- The user wants to "make this Dokploy-friendly" / "add a healthcheck".

## Inputs you need

- Stack: **Bun**, **Node/TS**, or **Next.js**. If unclear, ask.
- Target port (default: `3000`).
- Path to write files (default: current working directory).

## Stack templates

### Bun

`Dockerfile`:

```dockerfile
# syntax=docker/dockerfile:1.7
FROM oven/bun:1-slim AS deps
WORKDIR /app
COPY package.json bun.lockb ./
RUN bun install --frozen-lockfile

FROM oven/bun:1-slim AS runtime
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
ENV NODE_ENV=production
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD wget -qO- http://localhost:3000/ >/dev/null || exit 1
CMD ["bun", "run", "start"]
```

`.dockerignore`:

```
.git
.gitignore
node_modules
.env
.env.local
.env.*.local
*.log
.DS_Store
.vscode
.idea
dist
build
coverage
README.md
```

### Node/TS

`Dockerfile`:

```dockerfile
# syntax=docker/dockerfile:1.7
FROM node:20-slim AS deps
WORKDIR /app
COPY package.json package-lock.json* pnpm-lock.yaml* bun.lockb* ./
RUN \
  if   [ -f bun.lockb ];      then npm i -g bun && bun install --frozen-lockfile; \
  elif [ -f pnpm-lock.yaml ]; then npm i -g pnpm && pnpm install --frozen-lockfile; \
  elif [ -f package-lock.json ]; then npm ci; \
  else npm install; fi

FROM node:20-slim AS build
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

FROM node:20-slim AS runtime
WORKDIR /app
ENV NODE_ENV=production
COPY --from=build /app/dist ./dist
COPY --from=build /app/node_modules ./node_modules
COPY package.json ./
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD wget -qO- http://localhost:3000/ >/dev/null || exit 1
CMD ["node", "dist/index.js"]
```

`.dockerignore`: same as Bun, plus `dist` (since we build inside the image).

### Next.js

Requires `next.config.js` to set `output: 'standalone'`. Tell the user to add it if missing:

```js
// next.config.js
module.exports = {
  output: 'standalone',
};
```

`Dockerfile`:

```dockerfile
# syntax=docker/dockerfile:1.7
FROM node:20-slim AS deps
WORKDIR /app
COPY package.json package-lock.json* pnpm-lock.yaml* bun.lockb* ./
RUN \
  if   [ -f bun.lockb ];      then npm i -g bun && bun install --frozen-lockfile; \
  elif [ -f pnpm-lock.yaml ]; then npm i -g pnpm && pnpm install --frozen-lockfile; \
  elif [ -f package-lock.json ]; then npm ci; \
  else npm install; fi

FROM node:20-slim AS build
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
ENV NEXT_TELEMETRY_DISABLED=1
RUN npm run build

FROM node:20-slim AS runtime
WORKDIR /app
ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1
COPY --from=build /app/.next/standalone ./
COPY --from=build /app/.next/static ./.next/static
COPY --from=build /app/public ./public
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=5s --start-period=15s --retries=3 \
  CMD wget -qO- http://localhost:3000/api/health >/dev/null || exit 1
CMD ["node", "server.js"]
```

`.dockerignore`:

```
.git
.gitignore
node_modules
.env
.env.local
.env.*.local
.next
out
*.log
.DS_Store
.vscode
.idea
README.md
```

Optional healthcheck route — write to `app/api/health/route.ts` (App Router) or `pages/api/health.ts` (Pages Router):

```ts
// app/api/health/route.ts
export const GET = () => new Response('ok', { status: 200 });
```

## Dokploy notes

- Dokploy reads `EXPOSE` and the `HEALTHCHECK` directive when wiring Traefik. Keep both.
- Default port `3000` matches Dokploy's UI default for `Port` in the application settings. If you change it, update the Dokploy UI too.
- `.dockerignore` is critical — without it, `COPY . .` will copy `node_modules` and `.env` into the image (slow, leaky).

## Follow-ups

- "Now deploy it" → [`create-app-from-git.md`](./create-app-from-git.md).
- "Set env vars first" → [`update-env.md`](./update-env.md).

## See also

- [`create-app-from-git.md`](./create-app-from-git.md) to deploy the scaffolded project
