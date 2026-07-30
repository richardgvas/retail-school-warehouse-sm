# UI Conventions Reference

## Design Principles

The primary users are **non-technical warehouse staff**. UI must be:
- **Simple and unambiguous** — one clear action per screen where possible
- **Fast to operate** — keyboard and barcode scanner friendly
- **Forgiving** — confirm destructive actions, never silently fail
- **Legible** — large touch targets, good contrast, readable fonts

---

## Tech

- **shadcn/ui** components + **Tailwind CSS** — do not introduce other UI libs
- **React Query** for all server state
- **Zustand** for global client state (auth, toast/alert queue)
- **React Router v6** for navigation

---

## Layout

```
┌─────────────────────────────────────────┐
│  Sidebar (desktop) / Bottom nav (mobile)│
│  Logo | Nav links | User badge + logout │
├─────────────────────────────────────────┤
│  Page header: Title + primary action btn│
├─────────────────────────────────────────┤
│  Page content                           │
│  (table / form / detail view)           │
└─────────────────────────────────────────┘
```

- Sidebar collapses to icon-only on narrow screens.
- On tablets/phones: sticky bottom nav bar with icons + labels.

---

## Page patterns

### List page
- Filterable, paginated table (shadcn `<DataTable>`)
- Search bar at top right
- Filters (dropdowns) inline with the table header
- Row click → detail/edit drawer or page
- Primary action button (e.g. "Add Item") top right of header

### Detail / Edit page
- Read view with an "Edit" button (role-gated)
- Edit opens an inline form or a side drawer — not a separate page
- Cancel always discards and returns to read view

### Create form
- Full page form for complex entities (Item, Order)
- Side drawer for simpler ones (School, Supplier)
- Always show validation errors inline under the relevant field
- Submit button disabled while request is in flight

### Confirmation dialogs
- Use for: delete, cancel order, resolve alert, any irreversible action
- Always state what will happen: "This will permanently remove SKU #ABC123."
- Two buttons: destructive action (red) + "Cancel"

---

## Barcode / QR scanning

- A persistent "Scan" button lives in the nav bar (or a floating action button).
- Clicking it opens a camera modal (`@zxing/browser`).
- On successful scan: resolve SKU → navigate to item detail page.
- On failed scan: toast error "SKU not found."
- Also support manual SKU text input as fallback in the same modal.

---

## Stock level display

Always show stock as a badge with color coding:

| Stock level | Badge color | Label |
|---|---|---|
| > reorderPoint | green | e.g. "42 units" |
| ≤ reorderPoint, > 0 | yellow | e.g. "8 units – Low" |
| 0 | red | "Out of stock" |

---

## Toast notifications

Use Zustand + shadcn `<Toaster>` for all feedback:

```js
// store/toastStore.js
addToast({ type: 'success' | 'error' | 'warning', message: '' })
```

- Success: green, auto-dismiss 3s
- Error: red, stays until dismissed
- Warning: yellow, auto-dismiss 5s

---

## Role-gated UI

```jsx
import { useAuth } from '@/hooks/useAuth';

const { user } = useAuth();
const canEdit = ['admin', 'owner'].includes(user.role);

{canEdit && <Button>Edit</Button>}
```

Never show actions to roles that can't perform them. Hiding is UX; the API
enforces actual security.

---

## Loading & error states

- All data-fetching components show a skeleton loader (shadcn `<Skeleton>`).
- Errors show an inline alert with a retry button — never a blank page.
- Use React Query's `isLoading`, `isError`, `refetch` for this consistently.

---

## Naming conventions (frontend)

| Thing | Convention |
|---|---|
| Page components | `ItemsPage.jsx`, `OrderDetailPage.jsx` |
| Feature hooks | `useItems.js`, `useCreateMovement.js` |
| API modules | `items.api.js`, `orders.api.js` |
| Zustand stores | `authStore.js`, `toastStore.js` |
| Route paths | `/items`, `/items/:id`, `/orders/new` |
