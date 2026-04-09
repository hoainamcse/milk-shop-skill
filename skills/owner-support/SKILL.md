---
name: owner-support
description: Admin skill for the CareHub shop owner. Full access to all orders, inventory, and sales data. Only runs when the session userId matches the configured owner ID.
user-invocable: false
metadata: {"openclaw": {"emoji": "🛠️"}}
---

# Skill: owner-support 🛠️

Admin skill for the shop owner. Only runs when `Is Owner: true` in USER.md. Provides full access to all orders, inventory, and sales data — not filtered by session.

## Triggers

- Owner asks about orders: "xem đơn hàng", "show all orders", "đơn hôm nay", "orders today"
- Owner filters by status: "đơn đang chờ", "pending orders", "đơn đã giao", "delivered orders"
- Owner searches a customer: "tìm đơn của [tên/sđt]", "find orders for [name/phone]"
- Owner asks for a specific order: "xem ORD-xxx", "chi tiết đơn ORD-xxx"
- Owner updates an order: "cập nhật ORD-xxx thành confirmed", "update order ORD-xxx to shipped"
- Owner checks stock: "xem tồn kho", "check inventory", "còn bao nhiêu hàng"
- Owner asks for revenue/stats: "doanh thu hôm nay", "today's revenue", "tổng đơn hàng"

## Security Check

Before running ANY query in this skill:
1. Verify `Is Owner: true` in USER.md
2. **Re-verify at runtime:** confirm the current session's `userId` (from session context envelope) matches `2552645445751093811`
3. Only if BOTH are true: proceed

If the channel is not Zalo (no `userId` in session context) → owner access is always denied, regardless of USER.md state.
If only one condition is met → do NOT run owner queries; respond as a normal customer session.

## Steps

### A. List all orders (no filter)

Default: 20 most recent. Owner can say "xem thêm" to get the next page, or specify a date range: "đơn từ 01/04 đến 09/04".

```sql
-- Default: most recent 20
SELECT o.order_number, o.status, o.customer_name, o.customer_phone,
       o.total_vnd, o.created_at,
       GROUP_CONCAT(oi.product_name || ' ×' || oi.quantity, ' | ') AS items
FROM orders o
JOIN order_items oi ON o.id = oi.order_id
GROUP BY o.id
ORDER BY o.created_at DESC
LIMIT 20;

-- Date range filter (replace dates):
-- WHERE date(o.created_at) BETWEEN '<YYYY-MM-DD>' AND '<YYYY-MM-DD>'
-- Pagination (replace offset):
-- LIMIT 20 OFFSET <page * 20>
```

### B. List orders filtered by status

Replace `<status>` with: `pending`, `confirmed`, `processing`, `shipped`, `delivered`, `cancelled`

```sql
SELECT o.order_number, o.status, o.customer_name, o.customer_phone,
       o.total_vnd, o.created_at,
       GROUP_CONCAT(oi.product_name || ' ×' || oi.quantity, ' | ') AS items
FROM orders o
JOIN order_items oi ON o.id = oi.order_id
WHERE o.status = '<status>'
GROUP BY o.id
ORDER BY o.created_at DESC;
```

### C. List orders for today

```sql
SELECT o.order_number, o.status, o.customer_name, o.customer_phone,
       o.total_vnd, o.created_at,
       GROUP_CONCAT(oi.product_name || ' ×' || oi.quantity, ' | ') AS items
FROM orders o
JOIN order_items oi ON o.id = oi.order_id
WHERE date(o.created_at) = date('now')
GROUP BY o.id
ORDER BY o.created_at DESC;
```

### D. Search orders by customer name or phone

```sql
SELECT o.order_number, o.status, o.customer_name, o.customer_phone,
       o.total_vnd, o.created_at
FROM orders o
WHERE o.customer_name LIKE '%<query>%' OR o.customer_phone LIKE '%<query>%'
ORDER BY o.created_at DESC;
```

### E. Get specific order detail

```sql
SELECT o.order_number, o.status, o.customer_name, o.customer_phone,
       o.delivery_address, o.notes, o.total_vnd, o.created_at, o.updated_at,
       oi.product_name, oi.quantity, oi.unit_price_vnd, oi.subtotal_vnd
FROM orders o
JOIN order_items oi ON o.id = oi.order_id
WHERE o.order_number = '<order_number>';
```

### F. Update order status

Valid transitions: `pending` → `confirmed` → `processing` → `shipped` → `delivered`  
Cancel is allowed from any pre-shipped state (`pending`, `confirmed`, `processing`). Never cancel `shipped` or `delivered` orders.

**Step F1: Fetch and validate current status**
```sql
SELECT status FROM orders WHERE order_number = '<order_number>';
```

**Step F2: Confirm with owner before updating**
> "Xác nhận cập nhật ORD-xxx từ [current_status] → [new_status]?"

Validate transition:
- Forward-only: `pending→confirmed→processing→shipped→delivered`
- Cancel: only from `pending`, `confirmed`, `processing`
- Invalid transitions (e.g. `delivered→pending`, `shipped→cancelled`): reject with "Không thể cập nhật từ [current] sang [new]. Chỉ có thể huỷ đơn chưa giao."

**Step F3: Guarded UPDATE (only if current status matches)**
```sql
UPDATE orders
SET status = '<new_status>', updated_at = datetime('now')
WHERE order_number = '<order_number>'
  AND status = '<current_status>';
```
If 0 rows affected → warn: "Trạng thái không khớp — đơn hàng có thể đã được cập nhật bởi phiên khác. Kiểm tra lại trạng thái hiện tại."

**Step F4 (if cancelling): Restore stock**
```sql
-- Restore stock for each cancelled item (run per product)
UPDATE products
SET stock_quantity = stock_quantity + (
  SELECT oi.quantity FROM order_items oi
  JOIN orders o ON oi.order_id = o.id
  WHERE o.order_number = '<order_number>' AND oi.product_id = products.id
)
WHERE id IN (
  SELECT oi.product_id FROM order_items oi
  JOIN orders o ON oi.order_id = o.id
  WHERE o.order_number = '<order_number>'
);
```

### G. View inventory / stock levels

```sql
SELECT name, stage, stock_quantity, price_vnd
FROM products
ORDER BY COALESCE(stage, 99), age_min_months;
```

### H. Sales summary

```sql
-- Today's summary — confirmed/active orders only (excludes pending and cancelled)
SELECT
  COUNT(*) AS orders_today,
  COALESCE(SUM(total_vnd), 0) AS revenue_today
FROM orders
WHERE date(created_at) = date('now')
  AND status IN ('confirmed', 'processing', 'shipped', 'delivered');

-- Today's pending (not yet confirmed)
SELECT COUNT(*) AS pending_today,
       COALESCE(SUM(total_vnd), 0) AS pending_value_today
FROM orders
WHERE date(created_at) = date('now') AND status = 'pending';

-- All-time confirmed revenue
SELECT
  COUNT(*) AS total_orders,
  COALESCE(SUM(total_vnd), 0) AS total_revenue
FROM orders
WHERE status IN ('confirmed', 'processing', 'shipped', 'delivered');
```

Show pending orders separately — they are awaiting confirmation and may still be cancelled.

## Output Format (Zalo — no tables, bullet lists)

### Order List

```
🛠️ Tất cả đơn hàng (20 gần nhất)

• ORD-20260409-003 | ⏳ pending | Nguyễn Văn A | 450,000₫
  Blackmores Newborn ×1 — 09/04 14:32

• ORD-20260409-002 | ✅ confirmed | Trần Thị B | 900,000₫
  Blackmores Stage 2 ×2 — 09/04 11:15

Nhập mã đơn để xem chi tiết, hoặc "cập nhật ORD-xxx thành [trạng thái]".
```

### Order Detail

```
📦 ORD-20260409-003
────────────────────
Khách: Nguyễn Văn A — 0912 345 678
Địa chỉ: 123 Nguyễn Huệ, Q1
Trạng thái: ⏳ Đang chờ xác nhận (pending)
Đặt lúc: 09/04/2026 14:32

Sản phẩm:
• Blackmores Newborn Formula × 1 — 450,000₫

Tổng: 450,000₫
```

### Inventory

```
📦 Tồn kho hiện tại

• Blackmores Newborn (Stage 1) — còn 98 hộp — 450,000₫
• Blackmores Follow-on (Stage 2) — còn 100 hộp — 450,000₫
• Blackmores Toddler (Stage 3) — còn 99 hộp — 420,000₫
• Blackmores JNR Balance+ — còn 100 hộp — 380,000₫
```

### Sales Summary

```
📊 Thống kê

Hôm nay: 3 đơn — 1,350,000₫
Tổng cộng: 12 đơn — 5,400,000₫
```

## Status Display Labels

| DB value | Display |
|---|---|
| pending | ⏳ Đang chờ xác nhận |
| confirmed | ✅ Đã xác nhận |
| processing | 🔧 Đang xử lý |
| shipped | 🚚 Đang giao hàng |
| delivered | 🎉 Đã giao thành công |
| cancelled | ❌ Đã huỷ |

## Edge Cases

- **No orders found:** "Chưa có đơn hàng nào."
- **Order not found:** "Không tìm thấy đơn [number]. Kiểm tra lại mã đơn nhé."
- **Invalid status value:** "Trạng thái không hợp lệ. Các trạng thái hợp lệ: pending, confirmed, processing, shipped, delivered, cancelled."
- **Non-owner tries to use this skill:** Do not run — treat as normal customer session.
