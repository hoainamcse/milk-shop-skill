# TOOLS.md - Baby Milk Product Database

## Database

- **Engine:** SQLite
- **Path:** `/home/nhhnmm/.openclaw/workspace/data/products.db`
- **Tool:** `sqlite3` CLI via exec
- **Usage:** `sqlite3 /home/nhhnmm/.openclaw/workspace/data/products.db "<SQL>"`

## Schema

### products
```sql
CREATE TABLE products (
  id INTEGER PRIMARY KEY,
  slug TEXT UNIQUE NOT NULL,       -- URL slug, e.g. 'blackmores-newborn-formula'
  name TEXT NOT NULL,              -- Full product name
  stage INTEGER,                   -- 1=Newborn, 2=Follow-on, 3=Toddler, NULL=JNR/supplement
  age_min_months INTEGER,          -- Minimum age in months
  age_max_months INTEGER,          -- Maximum age in months (NULL = no upper bound)
  pack_size_g INTEGER,             -- Pack size in grams
  description TEXT,                -- Product description
  url TEXT,                        -- Product page URL
  source_type TEXT,                -- 'formula' | 'supplement_drink'
  price_vnd INTEGER,               -- Retail price in Vietnamese Dong
  price_usd REAL,                  -- Retail price in USD
  stock_quantity INTEGER DEFAULT 100  -- Available stock units
);
```

### nutrients
```sql
CREATE TABLE nutrients (
  id INTEGER PRIMARY KEY,
  product_id INTEGER NOT NULL REFERENCES products(id),
  name TEXT NOT NULL,
  category TEXT NOT NULL           -- 'vitamin' | 'mineral' | 'fatty_acid' | 'prebiotic' | 'protein' | 'other'
);
```

### benefits
```sql
CREATE TABLE benefits (
  id INTEGER PRIMARY KEY,
  product_id INTEGER NOT NULL REFERENCES products(id),
  benefit TEXT NOT NULL,
  body_system TEXT                 -- 'immune' | 'cognitive' | 'bone' | 'growth' | 'digestion' | 'general'
);
```

### preparation_steps
```sql
CREATE TABLE preparation_steps (
  id INTEGER PRIMARY KEY,
  product_id INTEGER NOT NULL REFERENCES products(id),
  step_order INTEGER NOT NULL,
  instruction TEXT NOT NULL
);
```

### allergens
```sql
CREATE TABLE allergens (
  id INTEGER PRIMARY KEY,
  product_id INTEGER NOT NULL REFERENCES products(id),
  allergen TEXT NOT NULL,
  presence TEXT NOT NULL           -- 'contains' | 'may_contain'
);
```

## SQL Cookbook

### Find products by baby age (in months)
```sql
SELECT id, name, stage, age_min_months, age_max_months, pack_size_g
FROM products
WHERE age_min_months <= 6 AND (age_max_months IS NULL OR age_max_months >= 6)
ORDER BY stage;
```

### List all products with age range
```sql
SELECT name, stage, age_min_months, age_max_months, pack_size_g, source_type
FROM products ORDER BY age_min_months;
```

### Get all nutrients for a product
```sql
SELECT n.name, n.category
FROM nutrients n
JOIN products p ON n.product_id = p.id
WHERE p.slug = 'blackmores-newborn-formula'
ORDER BY n.category, n.name;
```

### Get benefits by body system
```sql
SELECT b.benefit, b.body_system
FROM benefits b
JOIN products p ON b.product_id = p.id
WHERE p.id = 1
ORDER BY b.body_system;
```

### Get preparation steps for a product
```sql
SELECT ps.step_order, ps.instruction
FROM preparation_steps ps
JOIN products p ON ps.product_id = p.id
WHERE p.slug = 'blackmores-newborn-formula'
ORDER BY ps.step_order;
```

### Get allergens for a product
```sql
SELECT allergen, presence FROM allergens WHERE product_id = 1;
```

### Find products that don't contain a specific allergen
```sql
SELECT p.name FROM products p
WHERE p.id NOT IN (
  SELECT product_id FROM allergens WHERE allergen = 'Soy' AND presence = 'contains'
);
```

### Compare nutrients between two products
```sql
SELECT n.name, n.category,
  CASE WHEN n.product_id = 1 THEN 'Stage 1' ELSE 'Stage 2' END as product
FROM nutrients n
WHERE n.product_id IN (1, 2)
ORDER BY n.category, n.name;
```

### Search products by benefit keyword
```sql
SELECT DISTINCT p.name, b.benefit
FROM benefits b
JOIN products p ON b.product_id = p.id
WHERE b.benefit LIKE '%immune%';
```

### carts
```sql
CREATE TABLE carts (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id TEXT NOT NULL UNIQUE,   -- User session identifier
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);
```

### cart_items
```sql
CREATE TABLE cart_items (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    cart_id INTEGER NOT NULL REFERENCES carts(id) ON DELETE CASCADE,
    product_id INTEGER NOT NULL REFERENCES products(id),
    quantity INTEGER NOT NULL DEFAULT 1,
    added_at TEXT NOT NULL DEFAULT (datetime('now')),
    UNIQUE(cart_id, product_id)        -- One row per product per cart
);
```

### orders
```sql
CREATE TABLE orders (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    order_number TEXT NOT NULL UNIQUE, -- "ORD-YYYYMMDD-NNN"
    session_id TEXT,                   -- Links back to the user session
    customer_name TEXT,
    customer_phone TEXT,
    customer_email TEXT,
    delivery_address TEXT,
    status TEXT NOT NULL DEFAULT 'pending',
    -- status values: pending | confirmed | processing | shipped | delivered | cancelled
    notes TEXT,
    total_vnd INTEGER,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);
```

### order_items
```sql
CREATE TABLE order_items (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    order_id INTEGER NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    product_id INTEGER NOT NULL REFERENCES products(id),
    product_name TEXT NOT NULL,        -- Snapshot of name at time of order
    quantity INTEGER NOT NULL,
    unit_price_vnd INTEGER NOT NULL,   -- Snapshot of price at time of order
    subtotal_vnd INTEGER NOT NULL
);
```

## Products Overview

| ID | Slug | Stage | Age | Pack | Price (VND) | Price (USD) |
|----|------|-------|-----|------|-------------|-------------|
| 1 | blackmores-newborn-formula | 1 | 0–6 mo | 900g | 450,000 | $18.00 |
| 2 | blackmores-follow-on-2 | 2 | 6–12 mo | 900g | 450,000 | $18.00 |
| 3 | blackmores-toddler-milk-drink-3 | 3 | 12+ mo | 500g | 420,000 | $17.00 |
| 4 | blackmores-jnr-balance | — | 1–10 yr | 850g | 380,000 | $15.00 |

### Get or create a cart for a session
```sql
-- Get existing cart
SELECT id FROM carts WHERE session_id = '<session_id>';

-- Create cart if not exists
INSERT OR IGNORE INTO carts (session_id) VALUES ('<session_id>');
SELECT id FROM carts WHERE session_id = '<session_id>';
```

### Add or update item in cart
```sql
-- Upsert: insert or increase quantity
INSERT INTO cart_items (cart_id, product_id, quantity)
VALUES (<cart_id>, <product_id>, <qty>)
ON CONFLICT(cart_id, product_id)
DO UPDATE SET quantity = quantity + excluded.quantity;
```

### View cart with product details
```sql
SELECT p.name, p.stage, p.pack_size_g,
       ci.quantity,
       p.price_vnd AS unit_price,
       ci.quantity * p.price_vnd AS subtotal
FROM cart_items ci
JOIN carts c ON ci.cart_id = c.id
JOIN products p ON ci.product_id = p.id
WHERE c.session_id = '<session_id>';
```

### Update item quantity (set to 0 to remove)
```sql
-- Update
UPDATE cart_items SET quantity = <new_qty>
WHERE cart_id = (SELECT id FROM carts WHERE session_id = '<session_id>')
  AND product_id = <product_id>;

-- Remove if quantity is 0
DELETE FROM cart_items
WHERE cart_id = (SELECT id FROM carts WHERE session_id = '<session_id>')
  AND quantity <= 0;
```

### Generate next order number
```sql
SELECT 'ORD-' || strftime('%Y%m%d', 'now') || '-' ||
       printf('%03d', COALESCE(MAX(id), 0) + 1)
FROM orders
WHERE order_number LIKE 'ORD-' || strftime('%Y%m%d', 'now') || '-%';
```

### Place order (run in sequence)
```sql
-- 1. Insert order
INSERT INTO orders (order_number, session_id, customer_name, customer_phone,
                    customer_email, delivery_address, total_vnd)
VALUES ('<order_number>', '<session_id>', '<name>', '<phone>',
        '<email>', '<address>', <total>);

-- 2. Insert order items (run for each cart item)
INSERT INTO order_items (order_id, product_id, product_name, quantity, unit_price_vnd, subtotal_vnd)
SELECT last_insert_rowid(), p.id, p.name, ci.quantity, p.price_vnd, ci.quantity * p.price_vnd
FROM cart_items ci
JOIN carts c ON ci.cart_id = c.id
JOIN products p ON ci.product_id = p.id
WHERE c.session_id = '<session_id>';

-- 3. Deduct stock
UPDATE products SET stock_quantity = stock_quantity - (
  SELECT ci.quantity FROM cart_items ci
  JOIN carts c ON ci.cart_id = c.id
  WHERE c.session_id = '<session_id>' AND ci.product_id = products.id
)
WHERE id IN (
  SELECT ci.product_id FROM cart_items ci
  JOIN carts c ON ci.cart_id = c.id
  WHERE c.session_id = '<session_id>'
);

-- 4. Clear cart
DELETE FROM cart_items WHERE cart_id = (SELECT id FROM carts WHERE session_id = '<session_id>');
```

### List orders for a session
```sql
SELECT o.order_number, o.status, o.total_vnd, o.created_at,
       GROUP_CONCAT(oi.product_name || ' x' || oi.quantity, ', ') AS items
FROM orders o
JOIN order_items oi ON o.id = oi.order_id
WHERE o.session_id = '<session_id>'
GROUP BY o.id
ORDER BY o.created_at DESC;
```

### Get order detail by order number
```sql
SELECT o.order_number, o.status, o.customer_name, o.customer_phone,
       o.delivery_address, o.total_vnd, o.created_at,
       oi.product_name, oi.quantity, oi.unit_price_vnd, oi.subtotal_vnd
FROM orders o
JOIN order_items oi ON o.id = oi.order_id
WHERE o.order_number = 'ORD-20260409-001';
```

### Update order status
```sql
UPDATE orders
SET status = '<new_status>', updated_at = datetime('now')
WHERE order_number = '<order_number>';
```

## Zalo Channel

OpenClaw handles Zalo integration automatically. When running on Zalo, the platform provides user context directly — no commands needed.

### Identity Detection
If `userId`, `displayName`, or `avatar` are present in session context → you're on Zalo.

| Field | Description | Usage |
|---|---|---|
| `userId` | Stable Zalo user ID | Session ID: `zalo-{userId}` |
| `displayName` | User's Zalo display name | Pre-fill `customer_name` in orders |
| `avatar` | Avatar image URL | Store in USER.md |

### Available Plugin Actions
These are actions OpenClaw can perform on the Zalo channel:

| Action | Purpose |
|---|---|
| `send` | Send a text message |
| `image` | Send an image by URL |
| `link` | Send a link card |
| `friends` | List or search friends |
| `groups` | List groups |
| `me` | Get current user profile (userId, displayName, avatar) |
| `status` | Check plugin auth status |

## openclaw docs

Use `openclaw docs [query...]` to search the live OpenClaw documentation:

```bash
openclaw docs agents         # Agent workspace concepts
openclaw docs skills         # How skills work, SKILL.md format
openclaw docs identity       # IDENTITY.md template
openclaw docs soul           # SOUL.md template
openclaw docs heartbeat      # HEARTBEAT.md template
openclaw docs templates      # All templates reference
```
