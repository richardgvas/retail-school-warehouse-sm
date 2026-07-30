---
name: warehouse-uniform-app
description: >
  Project standards, architecture, and context for a MERN-stack warehouse
  management app for a small business that sells uniforms to schools.
  Use this skill whenever working on any part of this project — backend routes,
  frontend components, database models, auth, inventory logic, or deployment.
  Trigger on any mention of: warehouse, inventory, uniforms, schools, stock,
  SKU, orders, barcode, roles, turborepo, docker, TDD, or any file path
  referencing this project.
---

# Warehouse Uniform Management App — Project Skill

## 1. Project Context & Business Brief

A small business sells school uniforms to multiple schools (clients). They
operate a physical warehouse managed by non-technical staff. The app runs on
a **local area network (LAN)** — there is no public internet exposure.

### Key business entities
- **School / Client** — the buyer; each school orders specific uniform items
- **Uniform Item** — a product defined by type, school, size, color, SKU, and supplier
- **Stock Movement** — any change in inventory (in, out, return, damaged)
- **Order** — a school's request for items, linked to stock movements
- **User** — a person with a role (admin, staff, owner)

### Core workflows
1. Receiving stock from a supplier → stock-in movement
2. Fulfilling a school order → stock-out movement linked to an order
3. Handling returns or damaged items → dedicated movement types
4. Monitoring low stock and setting reorder points
5. Scanning barcodes / QR codes to identify items quickly
6. Managers and owners reviewing audit logs and reports

---

## 2. Architecture Standards

### Stack
| Layer | Technology |
|---|---|
| Monorepo | Turborepo |
| Frontend | React (Vite) + Tailwind CSS + shadcn/ui |
| Backend | Node.js + Express |
| Database | MongoDB + Mongoose |
| Auth | JWT (access token) + HTTP-only refresh token |
| Testing | Jest (backend) + Vitest + React Testing Library (frontend) |
| CI | Docker + Docker Compose |
| LAN deployment | Docker Compose on a single LAN server |

### Project structure (Turborepo monorepo)
```
/                                # Turborepo root
├── apps/
│   ├── web/                     # React frontend (Vite)
│   │   ├── src/
│   │   │   ├── api/             # Axios instances and API calls
│   │   │   ├── components/      # Shared UI components
│   │   │   ├── features/        # Feature folders (inventory, orders, users...)
│   │   │   │   └── inventory/
│   │   │   │       ├── components/
│   │   │   │       ├── hooks/
│   │   │   │       └── pages/
│   │   │   ├── hooks/           # Global shared hooks
│   │   │   ├── layouts/         # Page shell layouts (RoleLayout, etc.)
│   │   │   ├── router/          # React Router config
│   │   │   ├── store/           # Zustand global state
│   │   │   └── utils/
│   │   ├── vite.config.js
│   │   └── package.json
│   │
│   └── api/                     # Express backend
│       ├── src/
│       │   ├── config/          # DB connection, env, constants
│       │   ├── controllers/     # Route handlers (thin — logic in services)
│       │   ├── middleware/      # Auth, role guard, error handler, audit logger
│       │   ├── models/          # Mongoose models
│       │   ├── routes/          # Express routers
│       │   ├── services/        # Business logic
│       │   └── utils/           # Helpers (barcode gen, pagination, etc.)
│       ├── tests/               # Jest tests mirroring src/ structure
│       │   ├── unit/
│       │   └── integration/
│       ├── app.js
│       └── package.json
│
├── packages/
│   └── shared/                  # Shared constants, validation schemas, enums
│       └── package.json
│
├── docker/
│   ├── Dockerfile.web
│   ├── Dockerfile.api
│   └── docker-compose.yml       # LAN deployment config
│
├── turbo.json
├── .env.example
└── package.json                 # Root — Turborepo pipeline scripts
```

### Turborepo pipeline (turbo.json)
```json
{
  "pipeline": {
    "build":   { "dependsOn": ["^build"], "outputs": ["dist/**"] },
    "test":    { "dependsOn": ["^build"] },
    "lint":    {},
    "dev":     { "cache": false, "persistent": true }
  }
}
```

Root scripts: `turbo run dev`, `turbo run test`, `turbo run build`, `turbo run lint`.

### Architecture rules
- **No business logic in controllers.** Controllers call services; services
  contain all rules, validations, and DB interactions.
- Every write operation (stock movement, order update, user change) must
  produce an **audit log entry** via the audit middleware.
- Barcode/QR logic lives in `apps/api/src/utils/barcode.js` and
  `apps/web/src/features/inventory/hooks/useScanner.js`.
- Shared enums and constants (sizes, movement types, roles) live in
  `packages/shared/` and are imported by both apps.

---

## 3. Data Models

See `references/models.md` for full Mongoose schemas.

### Summary of collections
| Collection | Purpose |
|---|---|
| `users` | Auth, roles, profile |
| `schools` | Client/school master data |
| `suppliers` | Supplier master data |
| `items` | Uniform product catalog |
| `stockMovements` | Every inventory change (immutable log) |
| `orders` | School orders, linked to stockMovements |
| `auditLogs` | System-wide action log |

### Item identity
An item is uniquely identified by: `itemType + school + size + color + supplier`

SKU/barcode is generated from this combination and stored on the item document.
Never allow duplicate SKUs.

### Stock quantity rule
**Never store a `quantity` field directly on an item.** Stock level is always
computed by summing `stockMovements` for that item:
```
currentStock = SUM(movements where type IN ['stock_in','return'])
             - SUM(movements where type IN ['stock_out','damaged'])
```

### Movement types
| type | Effect | Notes |
|---|---|---|
| `stock_in` | + stock | Receiving from supplier |
| `stock_out` | - stock | Fulfilling an order |
| `return` | + stock | Customer return |
| `damaged` | - stock | Item written off |

---

## 4. Auth & Roles

### Roles
| Role | Permissions |
|---|---|
| `owner` | Full access including user management and all reports |
| `admin` | Manage inventory, orders, schools, suppliers. Cannot manage users |
| `staff` | Create stock movements, view inventory. Cannot edit catalog or reports |
| `viewer` | Read-only across all sections |

### Auth flow
- Login returns a short-lived **JWT access token** (15 min) and sets an
  **HTTP-only refresh token cookie** (7 days).
- The frontend stores the access token in memory only (never localStorage).
- A silent refresh runs automatically before expiry.
- Every protected route uses `middleware/auth.js` → `middleware/roleGuard.js`.

### Role guard pattern
```js
router.post('/items', auth, roleGuard(['admin', 'owner']), createItem);
```

---

## 5. Code Standards

### General
- **Language:** JavaScript (ES2022+). No TypeScript for now.
- **Async:** always `async/await`. No raw `.then()` chains.
- **Error handling:** all async route handlers wrapped in `asyncHandler()`.
  Never swallow errors silently.
- **Env vars:** all config via `.env`. Never hardcode IPs, ports, or secrets.
- **Comments:** comment the *why*, not the *what*.

### Backend (Node/Express)
- Controllers are thin — max ~20 lines, no direct DB calls.
- Service functions return plain objects (not Mongoose documents).
- Use Mongoose `lean()` on read queries for performance.
- Validate all request bodies with **express-validator** before the service.
- Pagination is mandatory on all list endpoints: `?page=1&limit=20`

### Frontend (React + Tailwind)
- **Styling:** Tailwind CSS utility classes only. No custom CSS files except
  for `tailwind.config.js` theme extensions (colors, fonts).
- **UI components:** shadcn/ui exclusively. Do not introduce other component libs.
- **State:** Zustand for global state. React Query for all server state.
- **No direct fetch calls in components.** All API calls via `apps/web/src/api/`.
- Feature-first folder structure. Functional components only. No class components.
- Prop validation via PropTypes.

### Naming conventions
| Thing | Convention | Example |
|---|---|---|
| Files (JS) | camelCase | `stockMovement.js` |
| React components | PascalCase | `ItemCard.jsx` |
| Test files | same name + `.test.js` | `stockService.test.js` |
| MongoDB collections | camelCase plural | `stockMovements` |
| API routes | kebab-case | `/api/stock-movements` |
| Env vars | SCREAMING_SNAKE | `MONGO_URI` |
| Tailwind custom tokens | kebab-case | `warehouse-green` |

---

## 6. Testing Standards (TDD)

**All code is written test-first.** No feature or bug fix ships without a
failing test written before the implementation.

### TDD cycle
1. **Red** — write a failing test that describes the expected behaviour
2. **Green** — write the minimum code to make it pass
3. **Refactor** — clean up without breaking the test

### Backend (Jest)
- **Unit tests** — test service functions in isolation; mock Mongoose models.
- **Integration tests** — test API routes against a real MongoDB test DB
  (use `mongodb-memory-server` — no separate test DB required).
- Test file mirrors source: `src/services/stockService.js` →
  `tests/unit/stockService.test.js`

```js
// Example: unit test for stock level computation
describe('stockService.getStockLevel', () => {
  it('returns 0 when no movements exist', async () => { ... });
  it('adds stock_in and return movements', async () => { ... });
  it('subtracts stock_out and damaged movements', async () => { ... });
  it('never returns a negative number', async () => { ... });
});
```

### Frontend (Vitest + React Testing Library)
- Test behaviour, not implementation. Query by role/label, not by class name.
- Every component that handles user interaction must have a test.
- Use `msw` (Mock Service Worker) to mock API calls in component tests.

```jsx
// Example: component test
it('shows low stock badge when stock is below reorder point', () => {
  render(<StockBadge currentStock={5} reorderPoint={10} />);
  expect(screen.getByText(/low/i)).toBeInTheDocument();
});
```

### Coverage targets
| Area | Minimum coverage |
|---|---|
| Services (backend) | 90% |
| Controllers (backend) | 80% |
| React components | 70% |

### Running tests
```bash
turbo run test              # all workspaces
turbo run test --filter=api # backend only
turbo run test --filter=web # frontend only
```

---

## 7. Technical Standards

### API design
- RESTful. Standard verbs: GET / POST / PATCH / DELETE.
- All responses follow this envelope:
```json
{
  "success": true,
  "data": {},
  "message": "Optional human-readable message",
  "pagination": { "page": 1, "limit": 20, "total": 134 }
}
```
- Errors: `{ "success": false, "error": "message" }` + correct HTTP status.
- No 200 responses for errors.

### Docker & CI

See `references/docker.md` for full Dockerfile and Compose config.

Summary:
- `Dockerfile.api` — Node 20 Alpine, runs Express.
- `Dockerfile.web` — Node 20 Alpine build stage → Nginx to serve static files.
- `docker-compose.yml` — three services: `api`, `web`, `mongo`.
- A CI pipeline runs on every push:
  1. `turbo run lint`
  2. `turbo run test` (all workspaces, with `mongodb-memory-server`)
  3. `turbo run build`
  4. `docker compose build`
- All four steps must pass before merging to `main`.

### Barcode / QR scanning
- SKUs generated server-side via deterministic hash of item attributes.
- QR codes generated with the `qrcode` npm package.
- Frontend uses `@zxing/browser` for camera-based scanning.

### Low stock alerts
- Each item has a configurable `reorderPoint` field (default: 10).
- `node-cron` job runs hourly, checks computed stock vs reorderPoint.
- Frontend polls `/api/stock-alerts/active` on the dashboard.

### Audit log
- Every POST / PATCH / DELETE writes an audit entry via middleware:
  `{ user, action, resource, resourceId, before, after, timestamp }`
- Audit logs are **never deleted**. Read-only paginated endpoint only.

### LAN deployment
- `docker compose up -d` on the LAN server starts everything.
- App accessible at `http://<server-ip>:3000` (web) and `:4000` (api).
- MongoDB data persisted via a named Docker volume.
- Document the server IP in `DEPLOYMENT.md` for staff reference.

---

## 8. Reference Files

Read these when working on the relevant area:

| File | When to read |
|---|---|
| `references/models.md` | Working on any Mongoose model or DB query |
| `references/api-routes.md` | Designing or implementing any API endpoint |
| `references/roles-matrix.md` | Implementing any auth or permission logic |
| `references/ui-conventions.md` | Building any React component or page |

| `references/docker.md` | Working on Docker, CI, or deployment |
| `references/dashboard.md` | Building or modifying any dashboard view or widget |
