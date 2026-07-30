# Roles & Permissions Matrix

## Role Definitions

| Role | Intended user | Description |
|---|---|---|
| `owner` | Business owner | Unrestricted access including user management |
| `admin` | Warehouse manager | Manage catalog, orders, schools, suppliers |
| `staff` | Warehouse worker | Day-to-day stock operations only |
| `viewer` | Read-only observer | View everything, change nothing |

---

## Permissions Matrix

| Resource / Action | owner | admin | staff | viewer |
|---|:---:|:---:|:---:|:---:|
| **Users** | | | | |
| View users | ✅ | ❌ | ❌ | ❌ |
| Create / edit / deactivate users | ✅ | ❌ | ❌ | ❌ |
| **Schools & Suppliers** | | | | |
| View | ✅ | ✅ | ✅ | ✅ |
| Create / edit | ✅ | ✅ | ❌ | ❌ |
| **Items (Catalog)** | | | | |
| View items & stock levels | ✅ | ✅ | ✅ | ✅ |
| Create / edit items | ✅ | ✅ | ❌ | ❌ |
| Soft-delete items | ✅ | ❌ | ❌ | ❌ |
| **Stock Movements** | | | | |
| View movements | ✅ | ✅ | ✅ | ✅ |
| Create stock_in / return / damaged | ✅ | ✅ | ✅ | ❌ |
| Fulfill order (stock_out) | ✅ | ✅ | ✅ | ❌ |
| **Orders** | | | | |
| View orders | ✅ | ✅ | ✅ | ✅ |
| Create / edit orders | ✅ | ✅ | ❌ | ❌ |
| Cancel orders | ✅ | ❌ | ❌ | ❌ |
| **Stock Alerts** | | | | |
| View alerts | ✅ | ✅ | ✅ | ✅ |
| Resolve alerts | ✅ | ✅ | ❌ | ❌ |
| **Audit Logs** | | | | |
| View audit logs | ✅ | ❌ | ❌ | ❌ |
| **Reports** | | | | |
| View reports | ✅ | ✅ | ❌ | ❌ |

---

## Middleware implementation

```js
// middleware/roleGuard.js
const roleGuard = (allowedRoles) => (req, res, next) => {
  if (!allowedRoles.includes(req.user.role)) {
    return res.status(403).json({ success: false, error: 'Insufficient permissions' });
  }
  next();
};

// Usage in routes:
router.post('/items', auth, roleGuard(['admin', 'owner']), createItem);
router.delete('/items/:id', auth, roleGuard(['owner']), deleteItem);
```

## UI visibility rules

- Navigation items and action buttons must be conditionally rendered based on role.
- Use the `useAuth()` hook to access `user.role` in components.
- Never rely solely on UI hiding for security — the API enforces permissions too.

```jsx
// Example
const { user } = useAuth();
{['admin','owner'].includes(user.role) && <Button>Create Item</Button>}
```
