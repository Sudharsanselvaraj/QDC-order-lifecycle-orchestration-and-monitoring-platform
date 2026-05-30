# QDC — Order Lifecycle Orchestration & Monitoring Platform

> A full-stack TypeScript submission for the QDC AI Engineer Intern assignment.
> Built on NestJS (backend) and React (frontend), demonstrating clean architecture,
> production-quality API design, and thoughtful engineering reasoning.

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Architecture](#2-architecture)
3. [Repository Structure](#3-repository-structure)
4. [Tech Stack](#4-tech-stack)
5. [Data Model](#5-data-model)
6. [API Reference](#6-api-reference)
7. [Implementation — What Was Built](#7-implementation--what-was-built)
8. [Frontend — UI Design](#8-frontend--ui-design)
9. [Engineering Decisions & Trade-offs](#9-engineering-decisions--trade-offs)
10. [Setup & Running Locally](#10-setup--running-locally)
11. [Docker](#11-docker)
12. [Theoretical Questions Summary](#12-theoretical-questions-summary)

---

## 1. Project Overview

**QDC** (Quick Dry Cleaning) is a B2B point-of-sale and business management platform for retail laundry and dry-cleaning businesses. Staff use it to track orders and garments through a four-stage lifecycle:

```
received → in_cleaning → ready → delivered
```

This submission implements a minimal but production-reasoned slice of that system:

- A **NestJS REST API** serving order and garment data with a new `/summary` analytics endpoint
- A **React + TypeScript dashboard** modelled after Linear's operational UI — dark theme, three-column layout, compact data table, click-to-inspect detail pane, and animated status filter tabs
- Full **ANSWERS.md** with six theory questions answered at senior-engineer depth

---

## 2. Architecture

### High-Level Data Flow

```
┌─────────────────────────────────────────────────────────────┐
│                        Browser                              │
│                                                             │
│  ┌──────────────┐   ┌───────────────────┐   ┌───────────┐   │
│  │   Sidebar    │   │   Main Panel      │   │  Detail   │   │
│  │  (nav/counts)│   │  (filter + table) │   │  Pane     │   │
│  └──────┬───────┘   └────────┬──────────┘   └───────┬───┘   │
│         │                    │  React state         │       │
│         └────────────────────┼──────────────────────┘       │
│                              │ fetch()                      │
└──────────────────────────────┼──────────────────────────────┘
                               │ HTTP/REST
                               ▼
              ┌────────────────────────────────┐
              │     NestJS  (port 3001)        │
              │                                │
              │  OrdersController              │
              │    GET /api/orders             │
              │    GET /api/orders/summary     │  ← new
              │    GET /api/orders/:id         │
              │         │                      │
              │  OrdersService                 │
              │    findAll()                   │
              │    findOne(id)                 │
              │    getGarmentStatusSummary()   │  ← new
              │         │                      │
              │  In-memory ORDERS array        │
              └────────────────────────────────┘
```
### Module Graph

```
AppModule
  └── OrdersModule
        ├── OrdersController   (HTTP routing)
        └── OrdersService      (business logic + data)
```

### Frontend Component Tree

```
<App>                         ← fetch, orders state, selectedStatus state, selectedOrder state
  ├── <Sidebar>               ← nav links with live counts computed from orders
  ├── <MainPanel>
  │     ├── <SearchBar>       ← UI only (no backend search in this slice)
  │     ├── <StatusFilterTabs>← pill buttons, one per GarmentStatus + "All"
  │     └── <OrdersTable>     ← compact table, row click → setSelectedOrder
  └── <DetailPane>            ← renders selected order garments + status breakdown
```

---

## 3. Repository Structure

```
.
├── package.json              # npm workspaces root — runs both server and client
├── Dockerfile                # Multi-stage build: server (dist) + client (build)
├── ANSWERS.md                # Six theory questions answered in depth
│
├── server/
│   ├── package.json
│   ├── tsconfig.json         # strict: true, emitDecoratorMetadata: true
│   └── src/
│       ├── main.ts           # Bootstrap: global prefix /api, CORS enabled, port 3001
│       ├── app.module.ts     # Root module — imports OrdersModule
│       └── orders/
│           ├── orders.module.ts
│           ├── orders.service.ts    # Domain types + in-memory data + all business logic
│           └── orders.controller.ts # HTTP routes — summary BEFORE :id (routing safety)
│
└── client/
    ├── package.json
    ├── tsconfig.json         # strict: true, jsx: react-jsx
    └── src/
        ├── index.tsx         # ReactDOM.createRoot entry point
        ├── App.tsx           # Root component: fetch, state, three-column layout
        └── OrdersList.tsx    # Order table + garment rows + status badges
```

---

## 4. Tech Stack

| Layer | Technology | Version |
|---|---|---|
| Runtime | Node.js | 20 LTS |
| Backend framework | NestJS | ^10.0.0 |
| Backend language | TypeScript | ^5.4.0 |
| HTTP platform | Express (via `@nestjs/platform-express`) | ^4.19.2 |
| Hot reload (dev) | ts-node-dev | ^2.0.0 |
| Frontend framework | React | ^18.2.0 |
| Frontend language | TypeScript | ^5.4.0 |
| Build tool | react-scripts (CRA) | 5.0.1 |
| Monorepo tooling | npm workspaces | built-in |
| Containerisation | Docker | node:20-alpine base |

---

## 5. Data Model

All types live in `server/src/orders/orders.service.ts` and are mirrored in `client/src/App.tsx`.

```typescript
type GarmentStatus = 'received' | 'in_cleaning' | 'ready' | 'delivered';

interface Garment {
  id: string;
  description: string;   // garment name (e.g. "Blue Shirt")
  status: GarmentStatus;
}

interface Order {
  id: string;            // e.g. "ORD-1001"
  customerName: string;
  createdAt: string;     // ISO 8601 date string
  garments: Garment[];
}

// New — added as part of implementation tasks
type GarmentStatusSummary = Partial<Record<GarmentStatus, number>>;
// e.g. { received: 1, in_cleaning: 1, ready: 1 }
// Zero-count statuses are omitted entirely.
// Returns {} when no garments exist.
```

### Seed Data

| Order | Customer | Garments |
|---|---|---|
| ORD-1001 | Alice Johnson | Blue Shirt (`received`), Black Trousers (`in_cleaning`) |
| ORD-1002 | Bob Singh | Wedding Gown (`ready`) |

---

## 6. API Reference

Base URL: `http://localhost:3001/api`

### `GET /orders`

Returns all orders with their garments.

**Response `200 OK`:**
```json
[
  {
    "id": "ORD-1001",
    "customerName": "Alice Johnson",
    "createdAt": "2026-05-30T13:58:52.000Z",
    "garments": [
      { "id": "G-1", "description": "Blue Shirt",     "status": "received"    },
      { "id": "G-2", "description": "Black Trousers", "status": "in_cleaning" }
    ]
  },
  {
    "id": "ORD-1002",
    "customerName": "Bob Singh",
    "createdAt": "2026-05-30T13:58:52.000Z",
    "garments": [
      { "id": "G-3", "description": "Wedding Gown", "status": "ready" }
    ]
  }
]
```

---

### `GET /orders/summary` ⭐ New

Returns a count of garments grouped by status across **all** orders.
Statuses with zero occurrences are **omitted**. Returns `{}` if no garments exist.

> **Routing note:** This route is declared **before** `GET /orders/:id` in the controller.
> If declared after, NestJS would match `"summary"` as an `:id` wildcard and return
> `{ error: "Order not found" }` instead — a silent bug with no compiler warning.

**Response `200 OK`:**
```json
{
  "received": 1,
  "in_cleaning": 1,
  "ready": 1
}
```

---

### `GET /orders/:id`

Returns a single order by ID.

**Response `200 OK` (found):**
```json
{
  "id": "ORD-1001",
  "customerName": "Alice Johnson",
  "createdAt": "2026-05-30T13:58:52.000Z",
  "garments": [ ... ]
}
```

**Response `200 OK` (not found):**
```json
{ "error": "Order with id ORD-9999 not found" }
```

> **Known limitation (discussed in ANSWERS.md Q1):** Not-found returns HTTP 200, not 404.
> The production fix is `throw new NotFoundException(...)`. The existing behaviour is preserved
> here per the "keep the overall structure recognizable" constraint.

---

## 7. Implementation — What Was Built

### Backend Task A — `getGarmentStatusSummary()`

Added to `OrdersService`:

```typescript
// Returns counts of garments per status across all orders.
// ?? 0 is used (not || 0) — the nullish coalescing operator correctly handles
// numeric zero, whereas || 0 would reset a count of 0 back to 0 (a classic
// AI-generation footgun and the exact pattern flagged in ANSWERS.md Q5).
getGarmentStatusSummary(): GarmentStatusSummary {
  const summary: GarmentStatusSummary = {};
  for (const order of this.orders) {
    for (const garment of order.garments) {
      summary[garment.status] = (summary[garment.status] ?? 0) + 1;
    }
  }
  return summary;
}
```

### Backend Task B — `GET /api/orders/summary`

Added to `OrdersController`, **positioned before `@Get(':id')`**:

```typescript
@Get('summary')
getSummary(): GarmentStatusSummary {
  return this.ordersService.getGarmentStatusSummary();
}
```

### Frontend Task A — Status filter

- Added `selectedStatus: 'all' | GarmentStatus` state to `App.tsx`, defaulting to `'all'`
- Rendered as pill-style tab buttons (not a `<select>`) — one per status plus "All", each showing a live count derived from the fetched orders array
- `StatusFilter` type defined as `'all' | GarmentStatus` to reuse the canonical union and avoid duplicating the four literal values

### Frontend Task B — Filtered garment display in `OrdersList`

- Accepts `selectedStatus` prop
- Two-pass filter: first maps each order to a filtered garments array, then checks `totalVisible` before rendering to produce a user-friendly empty-state message rather than empty cards
- Orders with zero visible garments after filtering are hidden entirely

---

## 8. Frontend — UI Design

The dashboard is styled to match **Linear's operational UI** — dark background, tight typographic hierarchy, and a three-column fixed layout.

### Layout

```
┌─────────────────────┬──────────────────────────────┬──────────────┐
│   SIDEBAR  240px    │   MAIN PANEL  flex-1         │ DETAIL 340px │
│                     │                              │              │
│  QDC Ops            │  [search bar]                │  DETAILS     │
│  ─────────          │  All 2  Received 1  Ready 1  │  ──────────  │
│  Inbox         2    │  ─────────────────────────── │  Customer    │
│  My Orders     2    │  ORDERS          2 orders    │  Order Date  │
│  Ready         1    │  ─────────────────────────── │   GARMENTS   │
│  In Cleaning   1    │  ORD-1001  Alice  2  30 May  │  Blue Shirt  │
│  Delivered     0    │  ORD-1002  Bob    1  30 May  │              │
│                     │                              │  STATUS      │
│  [Create Order]     │                              │  In Progress │
│  [Add Customer]     │                              │              │
└─────────────────────┴──────────────────────────────┴──────────────┘
```

### Colour Tokens

| Purpose | Value |
|---|---|
| Page background | `#0f0f10` |
| Surface (panels) | `#161618` |
| Elevated surface | `#1c1c1f` |
| Border | `#2a2a2e` |
| Primary text | `#f1f0ef` |
| Secondary text | `#c9c9d4` |
| Muted text | `#5e5e6e` |
| Accent (purple) | `#7c6af7` |

### Status Badge Colours

| Status | Background | Text |
|---|---|---|
| `received` | `#1e2a1e` | `#4ade80` (green) |
| `in_cleaning` | `#2a2416` | `#f59e0b` (amber) |
| `ready` | `#1a2035` | `#60a5fa` (blue) |
| `delivered` | `#252528` | `#9898aa` (neutral) |

### Animations

- Table row hover: `background-color 80ms ease`
- Detail pane slide-in: `translateX(12px) → 0` at `150ms ease` on order selection
- Filter tab press: `scale(0.97)` on `:active`
- Selected row flash: `rowSelect` keyframe `200ms ease`

---

## 9. Engineering Decisions & Trade-offs

### Why `Partial<Record<GarmentStatus, number>>` for the summary type?

`Record<GarmentStatus, number>` requires all four keys to always be present — impossible when a status has zero garments. `Partial<Record<...>>` makes every key optional, which matches the spec exactly and produces `{}` for the empty case.

### Why `?? 0` and not `|| 0` in the accumulator?

`summary[status] || 0` evaluates to `0` whenever `summary[status]` is falsy — including when it is already `0`. This means a count that reaches zero would never increment correctly. `?? 0` only substitutes for `null` and `undefined`, making it the correct choice for numeric accumulation.

### Why must `@Get('summary')` come before `@Get(':id')`?

NestJS resolves routes in declaration order. If `:id` is declared first, a request to `/api/orders/summary` matches the wildcard with `id = "summary"`, calls `findOne("summary")`, and returns `{ error: "Order with id summary not found" }` with HTTP 200 — no error is thrown, no log entry is produced. This is one of the most common silent bugs in NestJS and has no TypeScript compiler protection.

### Why client-side filtering instead of a `?status=` query parameter?

With the current dataset (two orders, three garments), a network round-trip per filter change would add latency with zero benefit. Client-side filtering is instant and works offline after the first fetch. The `?status=` parameter is designed and explained in ANSWERS.md Q6 as the migration path once pagination is introduced — at which point the query param is already the natural server contract and the client only needs to start sending it.

### Why not use `NotFoundException` from NestJS?

The assignment constraint is "keep the overall structure recognizable." Changing `return { error: ... }` to `throw new NotFoundException(...)` is a breaking change for any consumer that checks `response.error`. The limitation is explicitly documented in ANSWERS.md Q1 with a full production fix.

---

## 10. Setup & Running Locally

### Prerequisites

- **Node.js 20** (LTS recommended — the Dockerfile also uses node:20-alpine)
- npm 10+ (ships with Node 20)

### 1. Clone

```bash
git clone <your-fork-url>
cd QDC-order-lifecycle-orchestration-and-monitoring-platform
```

### 2. Install all dependencies

```bash
npm run install-all
```

This runs `npm install --workspaces`, installing both `server/` and `client/` dependencies from the repo root via npm workspaces.

### 3. Start both servers

```bash
npm run dev
```

This uses `concurrently` to start:

| Process | URL | Description |
|---|---|---|
| NestJS API | http://localhost:3001 | All routes under `/api` |
| React app | http://localhost:3000 | Dashboard UI |

### 4. Verify the API

```bash
# All orders
curl http://localhost:3001/api/orders

# Status summary (new endpoint)
curl http://localhost:3001/api/orders/summary

# Single order
curl http://localhost:3001/api/orders/ORD-1001

# Not found
curl http://localhost:3001/api/orders/ORD-9999
```

### Run servers independently

```bash
# Backend only
npm run server

# Frontend only
npm run client
```

### TypeScript build (server production build)

```bash
cd server
npm run build       # compiles to server/dist/
npm run start       # runs server/dist/main.js
```

---

## 11. Docker

A `Dockerfile` is included at the repository root for containerised deployment.

```bash
# Build
docker build -t qdc-platform .

# Run (exposes port 3001 for API, 3000 for client build served separately)
docker run -p 3001:3001 -p 3000:3000 qdc-platform
```

**Dockerfile stages:**

1. `node:20-alpine` base
2. Copy and install all workspace dependencies
3. Build server TypeScript → `server/dist/`
4. Build React app → `client/build/`
5. `CMD ["npm", "run", "dev"]` starts both processes via concurrently

> **Production note:** For a real deployment, the React build would be served via nginx
> and the NestJS process would run as a standalone node process, not via `ts-node-dev`.
> The Dockerfile is suitable for review/demo purposes.

---

## 12. Theoretical Questions Summary

Full answers are in [`ANSWERS.md`](./ANSWERS.md). Below is a one-line summary of each.

| Question | Core answer |
|---|---|
| **Q1** — HTTP 200 on not-found | Use `throw new NotFoundException()` — returns proper 404, fixes monitoring, OpenAPI, and CDN cache poisoning |
| **Q2** — Production data layer | Repository pattern with `IOrdersRepository` interface; swap `InMemoryRepo` (tests) for `TypeOrmRepo` (prod) with zero controller changes |
| **Q3** — Real-time updates | `useEffect` fetch-on-mount is fine for static data; production needs React Query + WebSocket gateway emitting `garment.status_updated` events |
| **Q4** — Evolving GarmentStatus | Split into `CleaningStatus`, `FulfilmentStatus`, and `BillingStatus` to prevent invalid state combinations (e.g. walk-in order set to `out_for_delivery`) |
| **Q5** — AI-generated code risks | Validate with unit tests written first, `tsc --strict`, integration tests on the HTTP layer, route smoke tests, and mutation testing (Stryker) |
| **Q6** — Server-side filtering | Add optional `?status=` query param now; keep client-side filtering; switch to server-side when pagination is introduced — the param is already the natural contract |

---

*Submitted by Sudharsan Selvaraj — QDC AI Engineer Intern Assignment*
