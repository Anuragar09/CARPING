# Workspace

## Overview

pnpm workspace monorepo using TypeScript. Each package manages its own dependencies.

## Stack

- **Monorepo tool**: pnpm workspaces
- **Node.js version**: 24
- **Package manager**: pnpm
- **TypeScript version**: 5.9
- **API framework**: Express 5
- **Database**: PostgreSQL + Drizzle ORM
- **Validation**: Zod (`zod/v4`), `drizzle-zod`
- **API codegen**: Orval (from OpenAPI spec)
- **Build**: esbuild (CJS bundle)

## CarPing App Features (artifacts/carping)

### Auth Flow
- Permissions screen (first-launch) → Splash → Landing (Sign Up / Sign In)
- Sign Up: phone → random OTP (shown as in-app toast notification) → profile details (name, email, age, mask toggle)
- Sign In: phone or email → random OTP → user lookup → login
- Duplicate phone detection on sign up (redirect to Sign In)
- `carping_auth_user`, `carping_has_registered`, `carping_permissions_shown` localStorage keys

### Vehicles
- All 7 vehicle types: Bike, Scooter, Car, Auto, Bus, Truck, Tractor
- Each type shows relevant fuel options (Petrol/Diesel/EV/CNG)
- Per-vehicle edit (pre-filled form) and delete (with confirmation modal)
- Privacy settings (hide photo, mask number) per vehicle

### Alerts
- Vehicle-specific alert types:
  - Bike/Scooter: Headlight, Theft Alert, Accident, No Plate, Other
  - Car: Door Open, Wrong Parking, Lights ON, Mirror Missing, Other
  - Auto/Bus/Truck/Tractor: Overloaded, Accident, Wrong Parking, Missing Person, Other
- Received / Sent tabs
- Photo proof (camera or gallery + crop)
- `lib/alerts.ts` — `getSentAlerts` / `saveSentAlert` utility functions

### Profile
- Editable name, photo (with crop modal), privacy toggles
- Settings drawer with sub-panels: Privacy Policy, Help Center, What's New, About CarPing, Contact Support
- Switch Account / Log Out

### Backend (artifacts/api-server)
- Express + PostgreSQL + Drizzle (users, vehicles, alerts tables)
- `email` column on users table
- `/users/lookup` GET (by phone or email)
- `PUT /vehicles/:vehicleId` update route
- `DELETE /vehicles/:vehicleId` delete route
- Body limit: 25MB (base64 photo support)
- OpenAPI spec → Orval codegen for all hooks
- REST-based call signaling: `POST/GET/PATCH/DELETE /api/calls` + `/api/calls/pending`
- `call_sessions` table: stores pending/active calls with SDP + ICE candidates in DB

### Call System Architecture
- **Reliability**: DB-backed call sessions (call_sessions table) so callers are notified even when offline
- **Polling**: Receiver polls `GET /api/calls/pending?userId=X` every 2s when idle; caller polls `GET /api/calls/:id` every 2s for accept/reject
- **WebRTC signaling via REST**: Offer SDP stored in DB on POST /api/calls; ICE candidates appended via POST /api/calls/:id/ice (role: caller|receiver); answer stored on PATCH /respond
- **WS bonus path**: WebSocket (`/api/ws`) is still used for real-time notifications when both parties are online simultaneously
- **Timeout**: 45s outgoing timeout; 60s DB TTL per call session
- **Browser notifications**: Shown when receiver has app in background tab

## Structure

```text
artifacts-monorepo/
├── artifacts/              # Deployable applications
│   └── api-server/         # Express API server
├── lib/                    # Shared libraries
│   ├── api-spec/           # OpenAPI spec + Orval codegen config
│   ├── api-client-react/   # Generated React Query hooks
│   ├── api-zod/            # Generated Zod schemas from OpenAPI
│   └── db/                 # Drizzle ORM schema + DB connection
├── scripts/                # Utility scripts (single workspace package)
│   └── src/                # Individual .ts scripts, run via `pnpm --filter @workspace/scripts run <script>`
├── pnpm-workspace.yaml     # pnpm workspace (artifacts/*, lib/*, lib/integrations/*, scripts)
├── tsconfig.base.json      # Shared TS options (composite, bundler resolution, es2022)
├── tsconfig.json           # Root TS project references
└── package.json            # Root package with hoisted devDeps
```

## TypeScript & Composite Projects

Every package extends `tsconfig.base.json` which sets `composite: true`. The root `tsconfig.json` lists all packages as project references. This means:

- **Always typecheck from the root** — run `pnpm run typecheck` (which runs `tsc --build --emitDeclarationOnly`). This builds the full dependency graph so that cross-package imports resolve correctly. Running `tsc` inside a single package will fail if its dependencies haven't been built yet.
- **`emitDeclarationOnly`** — we only emit `.d.ts` files during typecheck; actual JS bundling is handled by esbuild/tsx/vite...etc, not `tsc`.
- **Project references** — when package A depends on package B, A's `tsconfig.json` must list B in its `references` array. `tsc --build` uses this to determine build order and skip up-to-date packages.

## Root Scripts

- `pnpm run build` — runs `typecheck` first, then recursively runs `build` in all packages that define it
- `pnpm run typecheck` — runs `tsc --build --emitDeclarationOnly` using project references

## Packages

### `artifacts/api-server` (`@workspace/api-server`)

Express 5 API server. Routes live in `src/routes/` and use `@workspace/api-zod` for request and response validation and `@workspace/db` for persistence.

- Entry: `src/index.ts` — reads `PORT`, starts Express
- App setup: `src/app.ts` — mounts CORS, JSON/urlencoded parsing, routes at `/api`
- Routes: `src/routes/index.ts` mounts sub-routers; `src/routes/health.ts` exposes `GET /health` (full path: `/api/health`)
- Depends on: `@workspace/db`, `@workspace/api-zod`
- `pnpm --filter @workspace/api-server run dev` — run the dev server
- `pnpm --filter @workspace/api-server run build` — production esbuild bundle (`dist/index.cjs`)
- Build bundles an allowlist of deps (express, cors, pg, drizzle-orm, zod, etc.) and externalizes the rest

### `lib/db` (`@workspace/db`)

Database layer using Drizzle ORM with PostgreSQL. Exports a Drizzle client instance and schema models.

- `src/index.ts` — creates a `Pool` + Drizzle instance, exports schema
- `src/schema/index.ts` — barrel re-export of all models
- `src/schema/<modelname>.ts` — table definitions with `drizzle-zod` insert schemas (no models definitions exist right now)
- `drizzle.config.ts` — Drizzle Kit config (requires `DATABASE_URL`, automatically provided by Replit)
- Exports: `.` (pool, db, schema), `./schema` (schema only)

Production migrations are handled by Replit when publishing. In development, we just use `pnpm --filter @workspace/db run push`, and we fallback to `pnpm --filter @workspace/db run push-force`.

### `lib/api-spec` (`@workspace/api-spec`)

Owns the OpenAPI 3.1 spec (`openapi.yaml`) and the Orval config (`orval.config.ts`). Running codegen produces output into two sibling packages:

1. `lib/api-client-react/src/generated/` — React Query hooks + fetch client
2. `lib/api-zod/src/generated/` — Zod schemas

Run codegen: `pnpm --filter @workspace/api-spec run codegen`

### `lib/api-zod` (`@workspace/api-zod`)

Generated Zod schemas from the OpenAPI spec (e.g. `HealthCheckResponse`). Used by `api-server` for response validation.

### `lib/api-client-react` (`@workspace/api-client-react`)

Generated React Query hooks and fetch client from the OpenAPI spec (e.g. `useHealthCheck`, `healthCheck`).

### `scripts` (`@workspace/scripts`)

Utility scripts package. Each script is a `.ts` file in `src/` with a corresponding npm script in `package.json`. Run scripts via `pnpm --filter @workspace/scripts run <script>`. Scripts can import any workspace package (e.g., `@workspace/db`) by adding it as a dependency in `scripts/package.json`.
