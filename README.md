# Project-213

Zero-commission food delivery platform monorepo.

This repository is scaffolded for **Phase 1.1 (Foundation)** using a Turborepo-style workspace layout and includes:

- Customer mobile app (`React Native + Expo`)
- Driver mobile app (`React Native + Expo`)
- Restaurant dashboard (`React + Vite`)
- Admin panel (`React + Vite`)
- API package (`Fastify`)
- Shared package (`types/utils/constants`)

## Monorepo Structure

```text
apps/
  customer/
  driver/
  restaurant/
  admin/
packages/
  api/
  shared/
```

## Current Build State

- Phase 1.1 scaffold is complete
- API includes a basic health route: `GET /health`
- No product features are implemented yet

## Requirements

- Node.js 20+ (22 recommended)
- npm 10+ (or another workspace-capable package manager)

## Install Dependencies

From the repo root:

```bash
npm install
```

## Run Commands (Root)

```bash
npm run dev
npm run build
npm run lint
npm run typecheck
```

## Run Individual Packages

API (Fastify):

```bash
cd packages/api
npm install
npm run dev
```

Expected health check response:

```bash
curl http://localhost:3000/health
# {"status":"ok"}
```

Web apps:

```bash
cd apps/restaurant && npm install && npm run dev
cd apps/admin && npm install && npm run dev
```

Expo apps:

```bash
cd apps/customer && npm install && npm run dev
cd apps/driver && npm install && npm run dev
```

## Environment Files

Each app/package includes a starter `.env.example` file. Copy each one to `.env` as needed.

Notable file:

- `packages/api/.env.example` (contains initial backend env keys from the spec)

## Git Setup (SSH Recommended)

If this is a new repo:

```bash
git init -b main
git add .
git commit -m "chore: initial Turborepo scaffold"
git remote add origin git@github.com:<your-user>/<your-repo>.git
git push -u origin main
```

## Next Step

Proceed with **Phase 1.2**:

- Initialize Prisma in `packages/api`
- Add canonical models/enums from `SPEC.md`
- Run first migration
