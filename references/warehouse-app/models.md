# Data Models Reference

Full Mongoose schema definitions for the warehouse uniform app.

---

## User

```js
const userSchema = new Schema({
  name:         { type: String, required: true, trim: true },
  email:        { type: String, required: true, unique: true, lowercase: true },
  passwordHash: { type: String, required: true },
  role:         { type: String, enum: ['owner','admin','staff','viewer'], required: true },
  isActive:     { type: Boolean, default: true },
}, { timestamps: true });
```

---

## School

```js
const schoolSchema = new Schema({
  name:    { type: String, required: true, trim: true },
  address: { type: String },
  contact: {
    name:  String,
    phone: String,
    email: String,
  },
  isActive: { type: Boolean, default: true },
}, { timestamps: true });
```

---

## Supplier

```js
const supplierSchema = new Schema({
  name:    { type: String, required: true, trim: true },
  contact: {
    name:  String,
    phone: String,
    email: String,
  },
  isActive: { type: Boolean, default: true },
}, { timestamps: true });
```

---

## Item (Uniform Product)

```js
const itemSchema = new Schema({
  itemType:     { type: String, required: true, trim: true }, // e.g. "shirt", "pants"
  school:       { type: ObjectId, ref: 'School', required: true },
  supplier:     { type: ObjectId, ref: 'Supplier', required: true },
  size:         { type: String, enum: ['XS','S','M','L','XL','XXL'], required: true },
  color:        { type: String, required: true, trim: true },
  sku:          { type: String, required: true, unique: true }, // generated server-side
  reorderPoint: { type: Number, default: 10, min: 0 },
  isActive:     { type: Boolean, default: true },
  notes:        { type: String },
}, { timestamps: true });

// Virtual: current stock (populated via aggregation, not stored)
// Use stockService.getStockLevel(itemId) to compute
```

**SKU generation rule:**
```js
// server/utils/barcode.js
const crypto = require('crypto');
function generateSKU({ itemType, schoolId, size, color, supplierId }) {
  const raw = `${itemType}-${schoolId}-${size}-${color}-${supplierId}`.toLowerCase();
  return crypto.createHash('md5').update(raw).digest('hex').slice(0, 12).toUpperCase();
}
```

---

## StockMovement

```js
const stockMovementSchema = new Schema({
  item:      { type: ObjectId, ref: 'Item', required: true },
  order:     { type: ObjectId, ref: 'Order' }, // required if type is stock_out
  type:      { type: String, enum: ['stock_in','stock_out','return','damaged'], required: true },
  quantity:  { type: Number, required: true, min: 1 },
  notes:     { type: String },
  createdBy: { type: ObjectId, ref: 'User', required: true },
}, { timestamps: true });

// Immutable: never update or delete a movement. Corrections = new movement.
stockMovementSchema.pre('save', function(next) {
  if (!this.isNew) return next(new Error('Stock movements are immutable'));
  next();
});
```

---

## Order

```js
const orderSchema = new Schema({
  school:      { type: ObjectId, ref: 'School', required: true },
  reference:   { type: String, trim: true }, // school's own PO number, optional
  status:      { type: String, enum: ['pending','partial','fulfilled','cancelled'], default: 'pending' },
  lines: [{
    item:             { type: ObjectId, ref: 'Item', required: true },
    quantityOrdered:  { type: Number, required: true, min: 1 },
    quantityFulfilled:{ type: Number, default: 0 },
  }],
  notes:       { type: String },
  createdBy:   { type: ObjectId, ref: 'User', required: true },
}, { timestamps: true });
```

---

## AuditLog

```js
const auditLogSchema = new Schema({
  user:       { type: ObjectId, ref: 'User', required: true },
  action:     { type: String, required: true }, // e.g. 'CREATE_MOVEMENT', 'UPDATE_ORDER'
  resource:   { type: String, required: true }, // collection name
  resourceId: { type: ObjectId, required: true },
  before:     { type: Schema.Types.Mixed }, // snapshot before change (null for creates)
  after:      { type: Schema.Types.Mixed }, // snapshot after change
  ip:         { type: String },
}, { timestamps: true });

// Never delete audit logs. No update or delete routes for this collection.
```

---

## StockAlert

```js
const stockAlertSchema = new Schema({
  item:          { type: ObjectId, ref: 'Item', required: true },
  currentStock:  { type: Number, required: true },
  reorderPoint:  { type: Number, required: true },
  resolvedAt:    { type: Date }, // null = still active
}, { timestamps: true });
```
