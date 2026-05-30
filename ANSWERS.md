# ANSWERS.md — QDC Mini Assignment
## Sudharsan Selvaraj · AI Engineer Intern

---

## Q1 — The current `getOrder` returns `{ error: string }` with HTTP 200 when an order is not found. What problems does this cause, and how would you fix it?

### What the code does today

In `orders.controller.ts`, `getOrder` calls `this.ordersService.findOne(id)`, which returns `Order | undefined`. When the order is missing, the controller manually constructs `{ error: \`Order with id ${id} not found\` }` and returns it. NestJS serialises this as HTTP **200 OK** — success — because no exception was thrown and no status code was set.

```typescript
// orders.controller.ts — current behaviour
@Get(':id')
getOrder(@Param('id') id: string): Order | { error: string } {
  const order = this.ordersService.findOne(id);
  if (!order) {
    return { error: `Order with id ${id} not found` }; // ← HTTP 200, wrong
  }
  return order;
}
```

### Why this is a production problem

**1. Client code cannot use HTTP semantics.**
`fetch`, Axios, and every HTTP client library use the status code to branch between success and failure. A 200 with `{ error: "..." }` in the body forces every consumer to inspect the body after every call — there is no standard way to hook this into Axios interceptors or React Query's `onError` callback. Each consumer either silently ignores the error or duplicates a bespoke body-inspection check.

**2. Monitoring and alerting are blind.**
APM tools (Datadog, New Relic, Sentry) classify all 2xx responses as successful. A dry-cleaning store making 500 requests for missing orders per hour shows zero errors in every dashboard. On-call engineers have no signal.

**3. CDN and browser cache poisoning.**
A CDN configured to cache successful responses will cache the 200 error body and serve it to every subsequent request for that order ID — even after the order is created. The window is invisible and sticky.

**4. OpenAPI and generated SDK types break.**
OpenAPI tools assign a single TypeScript type to each HTTP status code. The union `Order | { error: string }` cannot be expressed on a 200 response — generated clients are either wrong or require manual overrides. The return type annotation `Order | { error: string }` is a TypeScript smell that leaks into the HTTP contract.

**5. `findOne` returning `undefined` is the wrong abstraction.**
A service method that returns `undefined` for missing entities puts the not-found logic in the controller, which is the wrong layer. The service should own the "does this exist?" decision.

### The fix

```typescript
// orders.service.ts
import { Injectable, NotFoundException } from '@nestjs/common';

findOne(id: string): Order {
  const order = ORDERS.find((o) => o.id === id);
  if (!order) {
    // NestJS global exception filter converts this to:
    // HTTP 404 { "statusCode": 404, "message": "Order ORD-9999 not found", "error": "Not Found" }
    throw new NotFoundException(`Order ${id} not found`);
  }
  return order;
}
```

```typescript
// orders.controller.ts — simplified after fix
@Get(':id')
getOrder(@Param('id') id: string): Order {
  return this.ordersService.findOne(id); // throws 404 automatically if missing
}
```

The return type becomes simply `Order`. No union. No body inspection. The NestJS global exception filter handles serialisation.

### Trade-offs

This is a **breaking change** for any existing consumer that checks `if (response.error)`. A migration requires: deprecation notice, a versioned endpoint (`/api/v2/orders/:id`), or a coordinated client release. The existing behaviour is preserved in this submission per the "keep the overall structure recognisable" constraint.

### Testing

```typescript
it('GET /api/orders/MISSING returns 404', async () => {
  const res = await request(app.getHttpServer()).get('/api/orders/ORD-9999');
  expect(res.status).toBe(404);
  expect(res.body.message).toContain('ORD-9999');
});
```

### Scalability note

When `ORDERS` moves to PostgreSQL via TypeORM, `findOneOrFail` throws `EntityNotFoundError` which maps directly to `NotFoundException` — the controller never changes. The service layer absorbs the persistence detail.

---

## Q2 — The in-memory `ORDERS` array is fine for a demo. What would a production data layer look like, and what would you change in `OrdersService`?

### What the code does today

`orders.service.ts` declares a module-level constant `ORDERS: Order[]` and operates on it directly inside `OrdersService`. The service is simultaneously a storage engine, a query layer, and a business logic layer. Every restart resets all data. Two running instances of the server hold two independent arrays.

### Why this is a production problem

**Data loss.** A single pod crash or deployment restart wipes every order Alice Johnson and Bob Singh ever placed. That is commercially unacceptable.

**No horizontal scaling.** Behind a load balancer, request A (create order) hits pod 1, request B (list orders) hits pod 2. Pod 2 knows nothing about the order. The UI shows "No active orders" while the garments sit in pod 1's memory.

**No transactions.** Adding a garment and updating an order total must succeed or fail together. An array has no rollback.

**No querying.** The current `findAll` returns everything. Real stores have hundreds of orders per day. You cannot paginate, filter by date, or search by customer name without loading every record.

### Production architecture

Introduce the **Repository pattern**, separating persistence from business logic:

```
OrdersController
      │ depends on
      ▼
OrdersService          ← pure business logic, no storage knowledge
      │ depends on (via NestJS DI)
      ▼
IOrdersRepository      ← interface (abstraction boundary)
      │
  ┌───┴──────────────────────┐
  │                          │
InMemoryOrdersRepository   TypeOrmOrdersRepository
(used in tests / dev)      (used in production)
```

```typescript
// orders.repository.interface.ts
export interface IOrdersRepository {
  findAll(): Promise<Order[]>;
  findById(id: string): Promise<Order | null>;
  findByStatus(status: GarmentStatus): Promise<Order[]>;
  save(order: Order): Promise<Order>;
}

// orders.service.ts — after refactor
@Injectable()
export class OrdersService {
  constructor(
    @Inject('ORDERS_REPOSITORY')
    private readonly repo: IOrdersRepository,
  ) {}

  async findAll(): Promise<Order[]> {
    return this.repo.findAll();
  }

  async findOne(id: string): Promise<Order> {
    const order = await this.repo.findById(id);
    if (!order) throw new NotFoundException(`Order ${id} not found`);
    return order;
  }

  async getGarmentStatusSummary(): Promise<GarmentStatusSummary> {
    const orders = await this.repo.findAll();
    const summary: GarmentStatusSummary = {};
    for (const order of orders) {
      for (const garment of order.garments) {
        summary[garment.status] = (summary[garment.status] ?? 0) + 1;
      }
    }
    return summary;
  }
}
```

The `InMemoryOrdersRepository` continues to exist for unit tests — no database required. The `TypeOrmOrdersRepository` is swapped in by changing one line in the module's `providers` array.

**Database choice for QDC:** PostgreSQL. Orders have garments (one-to-many), garments have status history (one-to-many), customers have billing (one-to-one). These are relational joins, not document queries. TypeORM or Prisma both work — Prisma gives stronger type safety at the schema level, TypeORM keeps everything in TypeScript decorators.

### Trade-offs

| | In-memory (now) | Repository + PostgreSQL |
|---|---|---|
| Setup | Zero | Docker, migrations, connection pool |
| Durability | None | Full ACID |
| Horizontal scale | Broken | Works with connection pooling |
| Test speed | Instant | Requires TestContainers or mocks |
| Query flexibility | None | Full SQL + indexes |

### Testing

- Unit-test `OrdersService` with a mock `IOrdersRepository` — no database, no network.
- Integration-test `TypeOrmOrdersRepository` against a real PostgreSQL container (TestContainers for Node.js).
- E2E-test `OrdersController` with the full NestJS app against an in-memory or test database.

---

## Q3 — The React frontend fetches orders once on mount. What are the limitations of this approach for a real-time dry-cleaning dashboard, and what alternatives would you consider?

### What the code does today

`App.tsx` runs a single `fetchOrders()` call inside `useEffect([], ...)`. The dependency array is empty, so the fetch never re-runs. After the initial load, the UI is a static snapshot. If Alice's Blue Shirt moves from `received` to `in_cleaning`, the dashboard shows `received` until the user refreshes the browser.

```typescript
useEffect(() => {
  fetchOrders(); // runs once, never again
}, []); // ← empty dependency array
```

### Why this matters in a dry-cleaning context

A dry-cleaning store processes dozens of orders per hour. Staff at the front desk need to know the moment a garment is `ready` so they can tell the customer. With stale data they may say "your coat is still being cleaned" when it has been ready for 20 minutes — a direct service failure. The counter staff and the cleaning staff are physically separated; the dashboard is their shared ground truth.

### Alternatives

**Option 1 — Polling**
```typescript
useEffect(() => {
  fetchOrders();
  const interval = setInterval(fetchOrders, 10_000); // every 10 seconds
  return () => clearInterval(interval);
}, []);
```
Trivial to implement. Always stale by up to 10 seconds. N clients × 6 requests/min = unnecessary server load. Does not scale.

**Option 2 — Server-Sent Events (SSE)**
Server pushes events over a persistent HTTP connection when a garment status changes. Works with `EventSource` natively in every browser. One-directional only — the client cannot send messages back. Long-lived connections can be closed by proxies and load balancers without notification.

**Option 3 — WebSockets (recommended for QDC)**
Bi-directional persistent connection. NestJS has first-class support via `@WebSocketGateway` and `socket.io`. When `OrdersService.updateGarmentStatus()` is called, it emits a `garment.status_updated` event. The client receives it and invalidates the relevant query.

```typescript
// server — orders.gateway.ts (new file)
@WebSocketGateway({ cors: true })
export class OrdersGateway {
  @WebSocketServer() server: Server;

  notifyStatusChange(orderId: string, garmentId: string, status: GarmentStatus) {
    this.server.emit('garment.status_updated', { orderId, garmentId, status });
  }
}
```

```typescript
// client — useOrders.ts (new hook)
import { useQuery, useQueryClient } from '@tanstack/react-query';
import { useEffect } from 'react';
import { io } from 'socket.io-client';

export function useOrders() {
  const queryClient = useQueryClient();

  useEffect(() => {
    const socket = io('http://localhost:3001');
    socket.on('garment.status_updated', () => {
      // Invalidate the cache — React Query refetches only what is stale
      queryClient.invalidateQueries({ queryKey: ['orders'] });
    });
    return () => { socket.disconnect(); };
  }, [queryClient]);

  return useQuery({ queryKey: ['orders'], queryFn: fetchOrders });
}
```

**Option 4 — React Query alone (without WebSockets)**
`staleTime` + `refetchInterval` gives background refresh, automatic retry on error, deduplication of concurrent requests, and loading/error/stale states for free. This alone eliminates 80% of the current limitations with minimal infrastructure. Pair it with WebSocket invalidation for true real-time.

### Comparison

| Approach | Latency | Server load | Complexity | Offline |
|---|---|---|---|---|
| Single fetch (now) | Infinite | Minimal | None | After load |
| Polling 10s | ≤10s | O(clients/10s) | Low | After load |
| SSE | ~0 | Low | Medium | No |
| WebSockets | ~0 | Low | Medium-high | No |
| React Query alone | Configurable | Low | Low | Partial |

### Files to change

- **New:** `server/src/orders/orders.gateway.ts` (WebSocket gateway)
- **New:** `server/src/orders/orders.module.ts` (add gateway to providers)
- **New:** `client/src/hooks/useOrders.ts`
- **Modify:** `client/src/App.tsx` → replace `useEffect/fetch` with `useOrders()`

---

## Q4 — `GarmentStatus` is currently a string union. How would you evolve this type as the QDC domain grows to include billing, delivery, and prepaid packages?

### What the code does today

```typescript
// orders.service.ts
export type GarmentStatus = 'received' | 'in_cleaning' | 'ready' | 'delivered';
```

`Garment` has a single `status` field of this type. The four values conflate the cleaning lifecycle with the fulfilment lifecycle.

### Why a flat union breaks under growth

**Invalid states become expressible.** A walk-in order should never have `out_for_delivery` status — there is no delivery. A prepaid package garment should not reach `invoiced`. But a flat union cannot encode these constraints — TypeScript cannot stop you assigning `out_for_delivery` to a walk-in garment.

**Statuses from different dimensions collide.** "Is this garment paid for?" is an orthogonal question to "Is it clean?". Stuffing both into one field means you need status values like `ready_awaiting_payment` or `in_cleaning_paid` — a combinatorial explosion as the domain grows.

**The UI filter in `OrdersList.tsx` becomes unmaintainable.** Today it maps four values. With billing it maps 12, with delivery 20. Every new status requires a UI change, a label update, and a badge colour.

### Production type design

Split into three independent dimensions, each with its own state machine:

```typescript
// The cleaning lifecycle — always present on every garment
type CleaningStatus =
  | 'received'
  | 'in_cleaning'
  | 'quality_check'   // inspector signs off before marking ready
  | 'ready';

// The fulfilment lifecycle — depends on order type
type PickupStatus   = 'awaiting_pickup' | 'collected';
type DeliveryStatus = 'scheduled' | 'out_for_delivery' | 'delivered' | 'delivery_failed';
type FulfilmentStatus = PickupStatus | DeliveryStatus;

// Billing — entirely orthogonal to cleaning
type BillingStatus = 'unpaid' | 'invoiced' | 'paid' | 'refunded';

// Garment — three independent status axes
interface Garment {
  id: string;
  description: string;
  cleaningStatus: CleaningStatus;
  fulfilmentStatus: FulfilmentStatus;
  billingStatus: BillingStatus;
}

// Order — discriminant controls which fulfilment statuses are valid
type Order =
  | { type: 'pickup';   garments: Array<Garment & { fulfilmentStatus: PickupStatus }> }
  | { type: 'delivery'; garments: Array<Garment & { fulfilmentStatus: DeliveryStatus }> };
```

The TypeScript discriminated union on `Order.type` means the compiler prevents `out_for_delivery` ever appearing on a pickup order — the constraint lives in the type system, not in runtime validation.

**Status history** replaces a single mutable field for audit and compliance:

```typescript
interface GarmentStatusEvent {
  at: string;           // ISO timestamp
  by: string;           // staff member ID
  cleaningStatus: CleaningStatus;
}
```

A garment's current cleaning status is `events[events.length - 1].cleaningStatus`. The full history is available for billing disputes ("when did we mark it ready?") and SLA tracking.

**Prepaid packages** add a separate `packageId` reference on the order — billing status is then computed from the package balance rather than the garment, keeping `BillingStatus` on the garment as a denormalised cache for display speed.

### Trade-offs

Richer types require more migration work. The existing `OrdersList.tsx` filter tabs need updating — they would filter by `cleaningStatus` rather than the old flat `status`. The database schema needs three columns (or a JSONB history array). Worth it: the alternative is a 30-value flat union that makes the codebase unintelligible within a year.

### Testing

Each status transition is a state machine edge. Use **XState** to formalise the machine and generate transition tests automatically. Or write them manually:

```typescript
it('does not allow out_for_delivery on a pickup order', () => {
  // TypeScript catches this at compile time — the test documents the intent
  const order: Order = { type: 'pickup', garments: [] };
  // @ts-expect-error — delivery status on pickup order
  order.garments[0].fulfilmentStatus = 'out_for_delivery';
});
```

---

## Q5 — Imagine `OrdersService` methods were initially generated by an AI tool. What specific risks would you look for, and how would you validate the generated code before shipping?

### Context

AI code generation tools produce plausible-looking code at speed. The failure mode is not random garbage — it is subtly wrong logic that passes a casual read, compiles cleanly, and fails only under specific inputs or in production conditions.

### Specific risks in this codebase

**Risk 1 — `|| 0` instead of `?? 0` in the summary accumulator**

The most likely AI mistake in `getGarmentStatusSummary`:

```typescript
// Buggy AI output
summary[garment.status] = (summary[garment.status] || 0) + 1;
```

`||` treats `0` as falsy. If `summary[garment.status]` is `0` (which it never is here, since keys are only added when count > 0, but would be in any implementation that pre-initialises the object), this resets the count to `1` instead of `1`. The fix — and what is implemented — is `?? 0`, which only substitutes for `null` and `undefined`.

**Risk 2 — Route order: `summary` after `:id`**

An AI completing the controller might append `@Get('summary')` after `@Get(':id')`:

```typescript
// Buggy AI output — wrong order
@Get(':id')   getOrder(...)  { ... }
@Get('summary') getSummary() { ... }  // ← never reached
```

NestJS resolves routes in declaration order. `GET /api/orders/summary` matches `:id` first with `id = "summary"`, calls `findOne("summary")`, and returns `{ error: "Order with id summary not found" }` — HTTP 200, no exception, no log line. No compiler error. No runtime error. The endpoint silently does not exist.

**Risk 3 — Missing edge cases**

AI tools rarely generate handling for:
- `ORDERS` array is empty → `getGarmentStatusSummary` should return `{}`, not crash
- An order has zero garments → the outer loop runs, inner loop does nothing, correct
- An unknown status value arrives from a future database migration → `Record<string, number>` vs `Partial<Record<GarmentStatus, number>>` matters here

**Risk 4 — Hallucinated API calls**

AI tools sometimes call NestJS decorators, TypeORM repository methods, or Node.js APIs that do not exist in the installed version. A hallucinated `@Get({ path: 'summary', exact: true })` compiles with `skipLibCheck: true` but fails at runtime.

**Risk 5 — Security gaps**

AI-generated CORS configs often default to `origin: '*'`. The current `main.ts` does exactly this: `NestFactory.create(AppModule, { cors: true })`. In production this allows any domain to make credentialed requests to the API. The fix is `cors: { origin: process.env.CLIENT_ORIGIN }`.

AI-generated `findOne` implementations sometimes interpolate the `id` parameter directly into a database query string — SQL injection. In this in-memory implementation it is harmless; in a TypeORM migration it would be a critical vulnerability if the developer copies the pattern naively.

### Validation process before shipping

**Step 1 — Write tests first, before reading the generated code.**
Tests written after seeing the code are biased by what the code does. Write them from the spec:

```typescript
describe('getGarmentStatusSummary', () => {
  it('returns {} when no orders exist', () => { ... });
  it('returns {} when orders have no garments', () => { ... });
  it('counts each status correctly', () => { ... });
  it('omits statuses with zero count', () => { ... });
  it('counts across multiple orders', () => { ... });
});
```

**Step 2 — Type check with `tsc --noEmit --strict`.**
Catches hallucinated decorators, wrong return types, and missing imports immediately.

**Step 3 — Integration test the HTTP layer with `supertest`.**

```typescript
it('GET /api/orders/summary returns 200 with status counts', async () => {
  const res = await request(app.getHttpServer()).get('/api/orders/summary');
  expect(res.status).toBe(200);
  expect(res.body).toEqual({ received: 1, in_cleaning: 1, ready: 1 });
});

it('GET /api/orders/summary is not shadowed by :id route', async () => {
  const res = await request(app.getHttpServer()).get('/api/orders/summary');
  expect(res.body.error).toBeUndefined(); // would be set if :id matched
});
```

**Step 4 — Code review checklist specific to AI output:**
- Every numeric accumulator uses `?? 0`, not `|| 0`
- Every static route (`summary`) is declared before every wildcard route (`:id`) in the same controller
- Every imported symbol actually exists in the installed package version
- No `origin: '*'` in production CORS config
- No string interpolation into query strings

**Step 5 — Mutation testing with Stryker.**
Stryker modifies the generated code one operator at a time (`?? 0` → `|| 0`, `>` → `>=`, removes a condition) and runs the test suite. If no test fails, the tests are insufficient — the mutation is "survived". This is the only reliable way to know whether the test suite actually guards the logic rather than just calling it.

### Broader principle

AI-generated code should be treated as a confident first draft. The value is in having something concrete to review — not in trusting the review can be skipped. The risk is proportional to how plausible the code looks. Code that is obviously wrong gets caught in seconds. Code that is subtly wrong under edge cases ships to production.

---

## Q6 — How would you add filtering by status on the server side instead of the client, and when would that be the right call?

### What the code does today

`App.tsx` fetches all orders from `GET /api/orders`, stores them in `useState`, and passes them to `OrdersList`. The filter dropdown (added in the frontend implementation task) operates entirely on the already-fetched array in the browser — no second network request is made when the filter changes.

### When client-side filtering is the right call

Right now. The entire dataset is three garments across two orders. The array fits in a few hundred bytes. The filter is instant, works offline after the first fetch, and adds no server load per filter change. Adding server-side filtering at this scale would add a network round-trip per interaction with zero performance benefit.

### When server-side filtering becomes necessary

| Signal | Why server-side is required |
|---|---|
| **Pagination** | The client cannot filter data it does not have. Once `GET /api/orders` returns a page of 50 rather than all orders, client-side filtering is wrong by definition. |
| **Large dataset** | A busy store creates 200 orders/day. Sending 6,000 garments over the wire to filter 50 on the client wastes bandwidth and slows the initial render. |
| **Access control** | A staff member should only see orders assigned to their station. The server must enforce this — a client-side filter is a UI courtesy, not a security control. |
| **Cross-store admin view** | Aggregating across multiple store locations requires a database `WHERE` clause. The client cannot do this. |
| **Full-text search** | Searching by customer name or garment description requires a database index. |

### Implementation

**Backend — add an optional `?status` query parameter:**

```typescript
// orders.controller.ts
import { Controller, Get, Param, Query } from '@nestjs/common';

@Get()
getOrders(@Query('status') status?: GarmentStatus): Order[] {
  return this.ordersService.findAll(status);
}
```

```typescript
// orders.service.ts
findAll(status?: GarmentStatus): Order[] {
  if (!status) return ORDERS;

  return ORDERS
    .map(order => ({
      ...order,
      garments: order.garments.filter(g => g.status === status),
    }))
    .filter(order => order.garments.length > 0);
}
```

**Backend — input validation (important — without this, any string is accepted):**

```typescript
import { IsEnum, IsOptional } from 'class-validator';

class GetOrdersQuery {
  @IsOptional()
  @IsEnum(['received', 'in_cleaning', 'ready', 'delivered'])
  status?: GarmentStatus;
}

@Get()
getOrders(@Query() query: GetOrdersQuery): Order[] {
  return this.ordersService.findAll(query.status);
}
```

**Frontend — build the URL from `selectedStatus`:**

```typescript
// App.tsx — replace the static fetch URL
const url = selectedStatus === 'all'
  ? 'http://localhost:3001/api/orders'
  : `http://localhost:3001/api/orders?status=${selectedStatus}`;

const res = await fetch(url);
```

With React Query:

```typescript
useQuery({
  queryKey: ['orders', selectedStatus],
  queryFn: () => fetchOrders(selectedStatus),
  // queryKey includes selectedStatus → automatic refetch on filter change
});
```

### Trade-offs

| Dimension | Client-side (now) | Server-side |
|---|---|---|
| Filter latency | Zero (instant) | One network round-trip (~50ms local) |
| Bandwidth | All orders every load | Only matching orders |
| Offline | Works after first fetch | Requires connectivity per filter |
| Access control | Cannot enforce | Enforced at data layer |
| Scalability | Breaks with pagination | Required for pagination |
| Implementation cost | Already done | ~20 lines across 2 files |

### Recommended migration path

1. **Now:** Keep client-side filtering. It is correct for this scale.
2. **Add the `?status=` query parameter to `GET /api/orders` now** — as an optional parameter with no breaking change. Existing clients that do not send it get all orders as before.
3. **When pagination is introduced:** The client must send `?status=` because it can no longer filter locally. The parameter is already there — no API change required, only a client change.

This is the correct application of YAGNI: do not build server-side filtering before you need it, but design the API so adding it later is a non-breaking change.

### Files to change (when the time comes)

- `server/src/orders/orders.service.ts` — `findAll(status?: GarmentStatus)`
- `server/src/orders/orders.controller.ts` — `@Query('status') status?: GarmentStatus`
- `client/src/App.tsx` — build URL from `selectedStatus` before fetching
