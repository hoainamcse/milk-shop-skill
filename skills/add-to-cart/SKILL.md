# Skill: add-to-cart 🛒

Add a product to the current session's cart.

## Triggers

- User says they want to buy, order, or purchase a product
- User says "thêm vào giỏ", "tôi muốn mua", "đặt hàng sản phẩm này"
- After a recommendation, user confirms they want it

## Inputs Required

- **Product:** name or stage of product (resolve to product_id via SQL)
- **Quantity:** default 1 if not specified
- **Session ID:** from USER.md — derived as follows:
  - **Zalo channel:** `zalo-{userId}` (from USER.md **Zalo userId**, populated at session startup)
  - **Other channels:** generate `sess-YYYYMMDD-XXXX` on first cart action, save to USER.md

## Steps

1. **Resolve product** — query by name/stage:
   ```sql
   SELECT id, name, price_vnd, stock_quantity FROM products
   WHERE name LIKE '%<keyword>%' OR stage = <stage>;
   ```

2. **Check stock:**
   ```sql
   SELECT stock_quantity FROM products WHERE id = <product_id>;
   ```
   - If `stock_quantity < quantity` → warn parent and abort

3. **Age safety check** — if USER.md has baby age, warn if product stage doesn't match:
   - Stage 1: 0–6 months, Stage 2: 6–12 months, Stage 3: 12+ months

4. **Get or create cart:**
   ```sql
   INSERT OR IGNORE INTO carts (session_id) VALUES ('<session_id>');
   SELECT id FROM carts WHERE session_id = '<session_id>';
   ```

5. **Upsert item:**
   ```sql
   INSERT INTO cart_items (cart_id, product_id, quantity)
   VALUES (<cart_id>, <product_id>, <qty>)
   ON CONFLICT(cart_id, product_id)
   DO UPDATE SET quantity = quantity + excluded.quantity;
   ```

6. **Show confirmation** — display what was added + cart running total:
   ```sql
   SELECT p.name, ci.quantity, p.price_vnd,
          ci.quantity * p.price_vnd AS subtotal
   FROM cart_items ci
   JOIN carts c ON ci.cart_id = c.id
   JOIN products p ON ci.product_id = p.id
   WHERE c.session_id = '<session_id>';
   ```

## Output Format

```
✅ Đã thêm vào giỏ hàng!

  Blackmores Newborn Formula (Stage 1, 900g)
  Số lượng: 2 hộp × 450,000₫ = 900,000₫

Tổng giỏ hàng: 900,000₫

Gõ /view-cart để xem toàn bộ giỏ hàng, hoặc /place-order để đặt hàng.
```

## Side Effects

- Creates a new cart row if none exists for the session
- Updates USER.md Session ID if it was newly generated
