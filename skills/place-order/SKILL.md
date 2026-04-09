# Skill: place-order 📦

Convert the current cart into a confirmed order. Collects customer details, generates an order number, snapshots prices, deducts stock, and clears the cart.

## Triggers

- User confirms they want to place/finalize the order
- "đặt hàng", "xác nhận đơn", "checkout", "place my order", "tôi muốn đặt"
- After reviewing cart, user says "được rồi" / "ok" / "yes, order it"

## Inputs Required

- **Session ID:** from USER.md (`zalo-{userId}` on Zalo, `sess-...` otherwise)
- **Customer name:**
  - **Zalo channel:** auto-filled from USER.md **Zalo displayName** — no need to ask
  - **Other channels:** ask if not already known
- **Customer phone:** ask if not already known
- **Delivery address:** ask conversationally
- **Notes (optional):** special delivery instructions

## Steps

### 1. Validate cart is not empty
```sql
SELECT COUNT(*) FROM cart_items ci
JOIN carts c ON ci.cart_id = c.id
WHERE c.session_id = '<session_id>';
```
If 0 → "Giỏ hàng đang trống, chưa thể đặt hàng."

### 2. Stock check for all items
```sql
SELECT p.name, ci.quantity, p.stock_quantity
FROM cart_items ci
JOIN carts c ON ci.cart_id = c.id
JOIN products p ON ci.product_id = p.id
WHERE c.session_id = '<session_id>'
  AND ci.quantity > p.stock_quantity;
```
If any rows returned → warn parent about which items are out/low stock

### 3. Collect customer info
- **On Zalo:** name is already known from `displayName` (USER.md). Only ask for phone and address.
  - Phone: "Cho mình xin số điện thoại để xác nhận đơn hàng nhé?"
  - Address: "Địa chỉ giao hàng của bạn ở đâu ạ?"
- **Other channels:** ask for any missing fields:
  - Name: "Cho mình biết tên của bạn để điền vào đơn hàng nhé?"
  - Phone: "Số điện thoại liên lạc của bạn là gì?"
  - Address: "Địa chỉ giao hàng của bạn ở đâu ạ?"

### 4. Show order summary for confirmation
Display all items, quantities, prices, and total before finalizing.
Ask: "Bạn xác nhận đặt hàng với thông tin trên không?"

### 5. Generate order number
```sql
SELECT 'ORD-' || strftime('%Y%m%d', 'now') || '-' ||
       printf('%03d', COALESCE(
         (SELECT COUNT(*) FROM orders
          WHERE order_number LIKE 'ORD-' || strftime('%Y%m%d', 'now') || '-%'),
         0) + 1);
```

### 6. Calculate total
```sql
SELECT SUM(ci.quantity * p.price_vnd) AS total
FROM cart_items ci
JOIN carts c ON ci.cart_id = c.id
JOIN products p ON ci.product_id = p.id
WHERE c.session_id = '<session_id>';
```

### 7. Insert order
```sql
INSERT INTO orders (order_number, session_id, customer_name, customer_phone,
                    customer_email, delivery_address, notes, total_vnd)
VALUES ('<order_number>', '<session_id>', '<name>', '<phone>',
        NULL, '<address>', '<notes>', <total>);
```

### 8. Insert order items (snapshot prices)
```sql
INSERT INTO order_items (order_id, product_id, product_name, quantity, unit_price_vnd, subtotal_vnd)
SELECT last_insert_rowid(), p.id, p.name,
       ci.quantity, p.price_vnd, ci.quantity * p.price_vnd
FROM cart_items ci
JOIN carts c ON ci.cart_id = c.id
JOIN products p ON ci.product_id = p.id
WHERE c.session_id = '<session_id>';
```

### 9. Deduct stock
```sql
UPDATE products
SET stock_quantity = stock_quantity - (
  SELECT ci.quantity FROM cart_items ci
  JOIN carts c ON ci.cart_id = c.id
  WHERE c.session_id = '<session_id>' AND ci.product_id = products.id
)
WHERE id IN (
  SELECT ci.product_id FROM cart_items ci
  JOIN carts c ON ci.cart_id = c.id
  WHERE c.session_id = '<session_id>'
);
```

### 10. Clear cart
```sql
DELETE FROM cart_items
WHERE cart_id = (SELECT id FROM carts WHERE session_id = '<session_id>');
```

### 11. Update USER.md
Set **Recent order number** to the new order number.

## Output Format

```
🎉 Đặt hàng thành công!

📋 Xác nhận đơn hàng
────────────────────────────────
Mã đơn hàng : ORD-20260409-001
Trạng thái  : Đang chờ xác nhận (pending)
Ngày đặt    : 09/04/2026

Sản phẩm đặt mua:
  • Blackmores Newborn Formula × 2   →   900,000₫

Tổng cộng   : 900,000₫ (~$36.00 USD)

Giao hàng đến: 123 Nguyễn Huệ, Quận 1, TP.HCM
Liên lạc     : Nguyễn Văn A — 0912 345 678
────────────────────────────────

Mình sẽ liên hệ xác nhận đơn hàng sớm nhé! 
Dùng /order-history để xem lịch sử đơn hàng bất kỳ lúc nào.
```

## Edge Cases

- **Stock runs out between add-to-cart and place-order** → warn, ask to reduce quantity or remove item
- **Cart becomes empty after removing out-of-stock items** → don't place empty order
