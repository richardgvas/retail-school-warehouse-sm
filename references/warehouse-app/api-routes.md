# API Routes Reference

Base URL (LAN): `http://<server-ip>:4000/api`

All routes except `/auth/login` and `/auth/refresh` require a valid JWT
in the `Authorization: Bearer <token>` header.

---

## Auth

| Method | Route | Role | Description |
|---|---|---|---|
| POST | `/auth/login` | public | Login, returns access token + sets refresh cookie |
| POST | `/auth/refresh` | public | Silent token refresh via cookie |
| POST | `/auth/logout` | any | Clears refresh cookie |

---

## Users

| Method | Route | Role | Description |
|---|---|---|---|
| GET | `/users` | owner | List all users |
| POST | `/users` | owner | Create user |
| PATCH | `/users/:id` | owner | Update user (role, active status) |
| PATCH | `/users/:id/password` | owner | Reset password |

---

## Schools

| Method | Route | Role | Description |
|---|---|---|---|
| GET | `/schools` | any | List schools |
| POST | `/schools` | admin, owner | Create school |
| PATCH | `/schools/:id` | admin, owner | Update school |

---

## Suppliers

| Method | Route | Role | Description |
|---|---|---|---|
| GET | `/suppliers` | any | List suppliers |
| POST | `/suppliers` | admin, owner | Create supplier |
| PATCH | `/suppliers/:id` | admin, owner | Update supplier |

---

## Items

| Method | Route | Role | Description |
|---|---|---|---|
| GET | `/items` | any | List items (paginated). Query: `?school=&size=&color=&page=&limit=` |
| GET | `/items/:id` | any | Get single item with computed stock level |
| GET | `/items/sku/:sku` | any | Look up item by SKU (used by barcode scanner) |
| POST | `/items` | admin, owner | Create item (SKU auto-generated) |
| PATCH | `/items/:id` | admin, owner | Update item metadata (not SKU) |
| DELETE | `/items/:id` | owner | Soft-delete (sets isActive: false) |

---

## Stock Movements

| Method | Route | Role | Description |
|---|---|---|---|
| GET | `/stock-movements` | admin, owner | List movements. Query: `?item=&type=&from=&to=&page=` |
| POST | `/stock-movements` | staff, admin, owner | Create movement (stock_in, return, damaged) |
| POST | `/stock-movements/fulfill` | staff, admin, owner | Fulfill an order line (creates stock_out movement) |

Movements are immutable — no PATCH or DELETE.

---

## Orders

| Method | Route | Role | Description |
|---|---|---|---|
| GET | `/orders` | any | List orders. Query: `?school=&status=&page=` |
| GET | `/orders/:id` | any | Get order with lines and fulfilment status |
| POST | `/orders` | admin, owner | Create order |
| PATCH | `/orders/:id` | admin, owner | Update order metadata / status |
| DELETE | `/orders/:id` | owner | Cancel order (soft) |

---

## Stock Alerts

| Method | Route | Role | Description |
|---|---|---|---|
| GET | `/stock-alerts/active` | any | List unresolved low-stock alerts |
| PATCH | `/stock-alerts/:id/resolve` | admin, owner | Mark alert resolved |

---

## Audit Logs

| Method | Route | Role | Description |
|---|---|---|---|
| GET | `/audit-logs` | owner | Paginated audit log. Query: `?user=&action=&from=&to=&page=` |

---

## Response envelope (all routes)

```json
{
  "success": true,
  "data": {},
  "message": "Optional",
  "pagination": { "page": 1, "limit": 20, "total": 134 }
}
```

Error:
```json
{ "success": false, "error": "Human-readable message" }
```
