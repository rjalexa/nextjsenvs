
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

### 1 Type-safe Response Models

`src/types/api.ts`

```typescript
// User
export interface User {
  id: string;
  name: string;
  email: string;
  createdAt: string;
  updatedAt: string;
}

export interface CreateUserRequest {
  name: string;
  email: string;
  password: string;
}

// Product
export interface Product {
  id: string;
  name: string;
  description: string;
  price: number;
  categoryId: string;
  imageUrl?: string;
  inStock: boolean;
}

// Auth
export interface LoginRequest {
  email: string;
  password: string;
}

export interface AuthResponse {
  token: string;
  user: User;
  expiresAt: string;
}

// Generic wrappers
export interface ApiError {
  message: string;
  code?: string;
  details?: Record<string, any>;
}

export interface ApiResponse<T> {
  data: T;
  success: boolean;
  message?: string;
}
```

---

### 2 Centralised API Client (`src/lib/api-client.ts`)

* Chooses **internal** URLs on the server and **public** URLs on the client.
* Implements intelligent caching (Next 14 `revalidate`, tag-based invalidation).
* Callers can force `cache: 'no-store'`.

```typescript
const isServer = typeof window === 'undefined';

const getBaseUrl = (svc: string) =>
  isServer
    ? {
        api:  process.env.INTERNAL_API_BASE_URL  ?? process.env.NEXT_PUBLIC_API_BASE_URL!,
        user: process.env.INTERNAL_USER_SERVICE_URL ?? process.env.NEXT_PUBLIC_USER_SERVICE_URL!,
        auth: process.env.INTERNAL_AUTH_SERVICE_URL ?? process.env.NEXT_PUBLIC_AUTH_SERVICE_URL!,
      }[svc]!
    : {
        api:  process.env.NEXT_PUBLIC_API_BASE_URL!,
        user: process.env.NEXT_PUBLIC_USER_SERVICE_URL!,
        auth: process.env.NEXT_PUBLIC_AUTH_SERVICE_URL!,
      }[svc]!;

const cfg = {
  cacheable: {
    user: { revalidate: 3600 },
    products: { revalidate: 1800 },
    categories: { revalidate: 7200 },
    config: { revalidate: 86400 },
  },
  noCacheEndpoints: ['/auth', '/orders', '/payments', '/analytics', '/notifications', '/real-time'],
  noCacheServices: ['auth', 'payment', 'analytics'],
};

const shouldCache = (svc: string, ep: string) =>
  !cfg.noCacheServices.includes(svc) && !cfg.noCacheEndpoints.some((p) => ep.includes(p));

const cacheOpt = (svc: string, ep: string, m = 'GET') =>
  m !== 'GET' || !shouldCache(svc, ep)
    ? { cache: 'no-store' as const }
    : { next: { revalidate: cfg.cacheable[svc]?.revalidate ?? 300, tags: [`${svc}-${ep.split('/')[1] || 'default'}`] } };

interface Opt { cache?: boolean }

export const apiClient = {
  async get<T>(svc: string, ep: string, o?: Opt): Promise<T> {
    const res = await fetch(`${getBaseUrl(svc)}${ep}`, {
      method: 'GET',
      headers: { 'Content-Type': 'application/json' },
      ...(o?.cache === false ? { cache: 'no-store' } : cacheOpt(svc, ep)),
    });
    if (!res.ok) throw new Error(`GET ${ep} failed ${res.status}`);
    return res.json();
  },
  async post<T>(svc: string, ep: string, body: any): Promise<T> {
    const res = await fetch(`${getBaseUrl(svc)}${ep}`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(body),
      cache: 'no-store',
    });
    if (!res.ok) throw new Error(`POST ${ep} failed ${res.status}`);
    return res.json();
  },
  async put<T>(svc: string, ep: string, body: any): Promise<T> {
    const res = await fetch(`${getBaseUrl(svc)}${ep}`, {
      method: 'PUT',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(body),
      cache: 'no-store',
    });
    if (!res.ok) throw new Error(`PUT ${ep} failed ${res.status}`);
    return res.json();
  },
  async delete<T>(svc: string, ep: string): Promise<T> {
    const res = await fetch(`${getBaseUrl(svc)}${ep}`, {
      method: 'DELETE',
      headers: { 'Content-Type': 'application/json' },
      cache: 'no-store',
    });
    if (!res.ok) throw new Error(`DELETE ${ep} failed ${res.status}`);
    return res.json();
  },
};
```

---

### 3 Cache Invalidation Helpers (`src/lib/cache-utils.ts`)

```typescript
import { revalidateTag, revalidatePath } from 'next/cache';

export const cacheUtils = {
  revalidateService: (svc: string, ep?: string) =>
    revalidateTag(ep ? `${svc}-${ep.split('/')[1] || 'default'}` : `${svc}-default`),
  revalidateUserData: () =>
    ['user-users', 'user-profile', 'user-preferences'].forEach(revalidateTag),
  revalidateProducts: () =>
    ['products-list', 'products-categories', 'categories-default'].forEach(revalidateTag),
  revalidatePagePath: (p: string) => revalidatePath(p),
};
```

---

### 4 API Route vs Rewrite — decision matrix

| Use an **API Route** when … | Use a **Rewrite** when … |
|-----------------------------|--------------------------|
| Data transformation / merging is required | Backend already returns the needed payload |
| Auth or role checks are needed | Backend handles auth itself |
| You need cache tags / invalidation | Backend sets cache headers |
| You want full TypeScript typing in the route | Contract already typed downstream |

---

### 5 Example cached route

```typescript
// src/app/api/users/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { apiClient } from '@/lib/api-client';
import type { User, CreateUserRequest, ApiResponse } from '@/types/api';

export async function GET(): Promise<NextResponse<ApiResponse<User[]>>> {
  const users = await apiClient.get<User[]>('user', '/users');
  return NextResponse.json({ data: users, success: true });
}

export async function POST(req: NextRequest): Promise<NextResponse<ApiResponse<User>>> {
  const dto = (await req.json()) as CreateUserRequest;
  const newUser = await apiClient.post<User>('user', '/users', dto);
  const { cacheUtils } = await import('@/lib/cache-utils');
  cacheUtils.revalidateUserData();
  return NextResponse.json({ data: newUser, success: true });
}
```

---

### 6 Example non‑cached route

```typescript
// src/app/api/auth/login/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { apiClient } from '@/lib/api-client';
import type { LoginRequest, AuthResponse, ApiResponse } from '@/types/api';

export async function POST(req: NextRequest): Promise<NextResponse<ApiResponse<AuthResponse>>> {
  const credentials = (await req.json()) as LoginRequest;
  const auth = await apiClient.post<AuthResponse>('auth', '/login', credentials);
  return NextResponse.json({ data: auth, success: true });
}
```

---

### 7 Simple client‑side usage

```tsx
'use client';
import { useEffect, useState } from 'react';
import type { User, ApiResponse } from '@/types/api';

export default function UserList() {
  const [users, setUsers] = useState<User[]>([]);
  useEffect(() => {
    fetch('/api/users')
      .then((r) => r.json())
      .then((j: ApiResponse<User[]>) => j.success && setUsers(j.data));
  }, []);
  return <ul>{users.map((u) => <li key={u.id}>{u.name}</li>)}</ul>;
}
```


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
