# Dashboard Reference

Each role sees a tailored dashboard on login. The layout is the same shell
but the widgets, stats, and actions are filtered by role.

---

## Route & component

- Route: `/` (root, redirects here after login)
- Component: `apps/web/src/features/dashboard/pages/DashboardPage.jsx`
- Role-specific widgets are composed from a shared `<DashboardGrid>` using
  the `useAuth()` hook to select which widget set to render.

```jsx
const WIDGETS_BY_ROLE = {
  owner:  ['StockAlerts', 'PendingOrders', 'FulfillmentStats', 'RecentMovements', 'RevenueBySchool'],
  admin:  ['StockAlerts', 'PendingOrders', 'FulfillmentStats', 'RecentMovements'],
  staff:  ['StockAlerts', 'RecentMovements', 'QuickScan'],
  viewer: ['StockAlerts', 'PendingOrders'],
};
```

---

## Widgets by role

### owner
| Widget | Description |
|---|---|
| StockAlerts | Count + list of items below reorder point |
| PendingOrders | Orders with status `pending` or `partial` |
| FulfillmentStats | % of order lines fulfilled this month |
| RecentMovements | Last 10 stock movements across all items |
| RevenueBySchool | Units shipped per school (current month vs last month) |

### admin
Same as owner minus RevenueBySchool (financial view is owner-only).

### staff
| Widget | Description |
|---|---|
| StockAlerts | Alerts only — no resolve button (admin/owner only) |
| RecentMovements | Last 10 movements created by this user |
| QuickScan | Inline barcode/QR scan shortcut to jump to item detail |

### viewer
| Widget | Description |
|---|---|
| StockAlerts | Read-only list of active alerts |
| PendingOrders | Read-only list of pending orders |

---

## Data sources (API endpoints polled by dashboard)

| Widget | Endpoint | Interval |
|---|---|---|
| StockAlerts | `GET /api/stock-alerts/active` | 60s |
| PendingOrders | `GET /api/orders?status=pending,partial&limit=10` | 60s |
| RecentMovements | `GET /api/stock-movements?limit=10` | 30s |
| FulfillmentStats | `GET /api/orders/stats/fulfillment` | 5 min |
| RevenueBySchool | `GET /api/orders/stats/by-school` | 5 min |

Use React Query with `refetchInterval` for polling. No websockets needed.

---

## New API endpoints required

### GET /api/orders/stats/fulfillment
Returns fulfillment rate for the current calendar month.
```json
{
  "success": true,
  "data": {
    "month": "2026-06",
    "totalLines": 340,
    "fulfilledLines": 298,
    "rate": 0.876
  }
}
```
Role: `admin`, `owner`

### GET /api/orders/stats/by-school
Returns units shipped per school, current month vs previous month.
```json
{
  "success": true,
  "data": [
    { "school": { "_id": "...", "name": "St. Mary's" }, "current": 120, "previous": 98 },
    { "school": { "_id": "...", "name": "La Salle" },   "current": 87,  "previous": 104 }
  ]
}
```
Role: `owner` only

---

## UI conventions

- Dashboard uses a **responsive grid**: 2 columns on tablet, 3 on desktop.
- Each widget is a shadcn `<Card>` with a title, metric/list, and optional
  action button (role-gated).
- StockAlerts card uses the same color-coded badge as inventory pages
  (red = out of stock, yellow = low).
- FulfillmentStats shows a simple radial progress (use Recharts `<RadialBarChart>`).
- RevenueBySchool shows a bar chart (Recharts `<BarChart>`), current vs previous
  month side by side, one bar group per school.
- All widgets show a skeleton loader while fetching and an inline error +
  retry on failure.

---

## TDD — dashboard test checklist

Write these tests before implementing:

**DashboardPage**
- Renders correct widget set for each role
- Does not render owner-only widgets for staff or viewer

**StockAlerts widget**
- Shows alert count badge
- Lists item name, current stock, reorder point
- Resolve button visible for admin/owner, hidden for staff/viewer

**PendingOrders widget**
- Lists order reference, school name, status
- Click navigates to order detail

**FulfillmentStats widget**
- Displays percentage correctly
- Shows 0% gracefully when no orders exist this month

**RevenueBySchool widget**
- Renders one bar group per school
- Only accessible/rendered for owner role
