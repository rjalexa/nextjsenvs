
# Next.js 14 Multi‑Environment Deployment Guide

## Overview
This guide covers best practices for developing Next.js 14 applications that must run in three scenarios:

* **Local Development** – all services on localhost
* **Containerised Development** – services split between localhost and Docker
* **Production** – all services containerised

---


## Project Structure
```text
my-nextjs-app/
├── src/
│   ├── app/
│   │   └── api/          # API routes (TypeScript)
│   ├── lib/
│   │   └── api-client.ts # Centralised API client
│   └── types/
│       ├── api.ts        # API response types
│       └── env.d.ts      # Environment variable types
├── docker/
│   ├── Dockerfile.dev
│   ├── Dockerfile.prod
│   ├── docker-compose.yml
│   ├── docker-compose.dev.yml
│   └── docker-compose.prod.yml
├── .env.local
├── .env.development
├── .env.production
├── next.config.mjs
├── package.json
└── pnpm-lock.yaml
```


---

## Environment Configuration Strategy

### 1 Environment variables

* Client‑side variables use the `NEXT_PUBLIC_` prefix.  
* Server‑only variables have no prefix.  
* Declare all variables in `src/types/env.d.ts` for type‑safety.

```typescript
declare namespace NodeJS {
  interface ProcessEnv {
    INTERNAL_API_BASE_URL?: string;
    INTERNAL_USER_SERVICE_URL?: string;
    INTERNAL_AUTH_SERVICE_URL?: string;
    DATABASE_URL: string;

    NEXT_PUBLIC_API_BASE_URL: string;
    NEXT_PUBLIC_USER_SERVICE_URL: string;
    NEXT_PUBLIC_AUTH_SERVICE_URL: string;

    NODE_ENV: 'development' | 'production' | 'test';
  }
}
```

Create a separate `.env.*` file for each environment:


**Why are there two places to define them?**

| File | Purpose | Checked by TypeScript? | Loaded at runtime? |
|------|---------|------------------------|--------------------|
| `src/types/env.d.ts` | *Declaration only* – lets the TypeScript compiler know each variable exists and what type it has. | ✅ Yes | ❌ No |
| `.env.*` files | *Values* – provide the actual key = value pairs for each environment. | ❌ No | ✅ Yes |

Add or rename a variable → update **both** the declaration file *and* every relevant `.env.*` file.
* `.env.local` – localhost values  
* `.env.development` – containerised‑dev; server variables use Docker DNS names  
* `.env.production` – production; server variables use Docker DNS, client variables use public URLs

**Add `.env` files to Git ignore**

```gitignore
# Local and per‑environment secrets
.env.local
.env.development
.env.production
```

Commit a **sanitised** `\.env.example` to document required variables while keeping real credentials out of version control.

### 2 Build‑time vs run‑time values

`next.config.mjs` executes only during `next build`. Any environment variable used
inside `rewrites()`, `headers()` or elsewhere in that file is **baked into the build**.
If a value must differ between deployments without rebuilding, place the proxy
logic in a reverse‑proxy (e.g. Nginx) or use `next.config.runtime`.

---

## next.config.mjs
```javascript
/** @type {{import('next').NextConfig}} */
const nextConfig = {
  typescript: { ignoreBuildErrors: false },
  eslint: { ignoreDuringBuilds: false },

  experimental: {
    typedRoutes: true,
  },

  async rewrites() {
    return [
      {
        source: '/api/proxy/cached/:path*',
        destination: `${process.env.INTERNAL_API_BASE_URL || process.env.NEXT_PUBLIC_API_BASE_URL}/:path*`,
      },
      {
        source: '/api/proxy/live/:path*',
        destination: `${process.env.INTERNAL_API_BASE_URL || process.env.NEXT_PUBLIC_API_BASE_URL}/:path*`,
      },
      {
        source: '/api/users/:path*',
        destination: `${process.env.INTERNAL_USER_SERVICE_URL || process.env.NEXT_PUBLIC_USER_SERVICE_URL}/:path*`,
      },
      {
        source: '/api/auth/:path*',
        destination: `${process.env.INTERNAL_AUTH_SERVICE_URL || process.env.NEXT_PUBLIC_AUTH_SERVICE_URL}/:path*`,
      },
    ];
  },

  async headers() {
    return [
      {
        source: '/api/proxy/cached/:path*',
        headers: [
          { key: 'Cache-Control', value: 'public, s-maxage=300, stale-while-revalidate=600' },
          { key: 'Vary', value: 'Accept-Encoding' },
        ],
      },
      {
        source: '/api/proxy/live/:path*',
        headers: [
          { key: 'Cache-Control', value: 'no-cache, no-store, must-revalidate' },
        ],
      },
      {
        source: '/api/auth/:path*',
        headers: [
          { key: 'Cache-Control', value: 'no-cache, no-store, must-revalidate' },
          { key: 'Pragma', value: 'no-cache' },
          { key: 'Expires', value: '0' },
        ],
      },
    ];
  },

  images: {
    remotePatterns: [
      { protocol: 'https', hostname: '**.amazonaws.com' },
      { protocol: 'https', hostname: '**.cloudinary.com' },
    ],
  },

  output: 'standalone',
};

export default nextConfig;
```

---

## Docker Configuration


### Why three Docker Compose files?

| File | Role |
|------|------|
| **docker/docker-compose.yml** | *Base layer* – defines shared infrastructure (e.g., the database volume). |
| **docker/docker-compose.dev.yml** | Development overrides: mounts source code, exposes extra ports, uses dev images. |
| **docker/docker-compose.prod.yml** | Production overrides: runs the optimised images, enables an Nginx reverse‑proxy, detached mode by default. |

`docker-compose` lets you stack files with `-f file1 -f file2 …`; later files override or extend keys from earlier ones. That makes it possible to **build once** and **run with environment‑specific tweaks** without duplicating the entire compose file.


### docker/Dockerfile.dev
```dockerfile
FROM node:20-alpine AS base

WORKDIR /app

RUN npm install -g pnpm
ENV PNPM_HOME=/pnpm
RUN mkdir -p /pnpm && pnpm config set store-dir /pnpm

COPY package.json pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile --strict-peer-dependencies

COPY . .

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s   CMD wget -qO- http://localhost:3000/api/health || exit 1

CMD ["pnpm","dev"]
```

### docker/Dockerfile.prod
```dockerfile
FROM node:20-alpine AS base
RUN npm install -g pnpm

FROM base AS deps
RUN apk add --no-cache libc6-compat
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile --prod --strict-peer-dependencies

FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN pnpm build

FROM base AS runner
WORKDIR /app
ENV NODE_ENV=production
RUN addgroup --system --gid 1001 nodejs  && adduser --system --uid 1001 nextjs
COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs
EXPOSE 3000
ENV PORT=3000 HOSTNAME="0.0.0.0"

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s   CMD wget -qO- http://localhost:3000/api/health || exit 1

CMD ["node","server.js"]
```

---

### docker-compose.yml (base)
```yaml
services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

### docker-compose.dev.yml
```yaml
services:
  nextjs-app:
    build:
      context: .
      dockerfile: docker/Dockerfile.dev
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
    env_file:
      - .env.development
    volumes:
      - ./src:/app/src
      - ./public:/app/public
      - ./next.config.mjs:/app/next.config.mjs
      - ./tsconfig.json:/app/tsconfig.json
      - ./tailwind.config.ts:/app/tailwind.config.ts
      - pnpm_store:/pnpm
    depends_on:
      - postgres

  api-service:
    image: your-api-service:latest
    ports:
      - "3001:3001"
    environment:
      - DATABASE_URL=postgresql://user:pass@postgres:5432/mydb
    depends_on:
      - postgres

  user-service:
    image: your-user-service:latest
    ports:
      - "3002:3002"
    depends_on:
      - postgres

  postgres:
    extends:
      file: docker/docker-compose.yml
      service: postgres

volumes:
  postgres_data:
  pnpm_store:
```

### docker-compose.prod.yml
```yaml
services:
  nextjs-app:
    build:
      context: .
      dockerfile: docker/Dockerfile.prod
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
    env_file:
      - .env.production
    depends_on:
      - api-service
      - user-service
      - postgres

  api-service:
    image: your-api-service:latest
    environment:
      - DATABASE_URL=postgresql://user:pass@postgres:5432/mydb
    depends_on:
      - postgres

  user-service:
    image: your-user-service:latest
    depends_on:
      - postgres

  postgres:
    extends:
      file: docker/docker-compose.yml
      service: postgres

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - nextjs-app

volumes:
  postgres_data:
```

---

## API Integration

*(The API client, cache utilities, route examples and React components are unchanged from the original guide and remain valid; include them here as‑is.)*

---

## Package.json
```jsonc
{
  "name": "my-nextjs-app",
  "version": "0.1.0",
  "private": true,
  "packageManager": "pnpm@8.0.0",
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint",
    "type-check": "tsc --noEmit",
    "docker:dev": "docker-compose -f docker/docker-compose.yml -f docker/docker-compose.dev.yml up",
    "docker:prod": "docker-compose -f docker/docker-compose.yml -f docker/docker-compose.prod.yml up -d",
    "docker:build": "docker-compose -f docker/docker-compose.yml -f docker/docker-compose.prod.yml build"
  },
  "dependencies": {
    "next": "14.2.0",
    "react": "^18",
    "react-dom": "^18"
  },
  "devDependencies": {
    "typescript": "^5",
    "@types/node": "^20",
    "@types/react": "^18",
    "@types/react-dom": "^18",
    "eslint": "^8",
    "eslint-config-next": "14.2.0",
    "tailwindcss": "^3.4.0",
    "autoprefixer": "^10.0.1",
    "postcss": "^8"
  },
  "engines": {
    "node": ">=20.0.0",
    "pnpm": ">=8.0.0"
  }
}
```

---

## Deployment Commands

```bash
# Local development
pnpm dev

# Local with Docker services
docker-compose -f docker/docker-compose.yml -f docker/docker-compose.dev.yml up postgres api-service
pnpm dev

# Full containerised development
docker-compose -f docker/docker-compose.yml -f docker/docker-compose.dev.yml up

# Production
docker-compose -f docker/docker-compose.yml -f docker/docker-compose.prod.yml up -d
```
