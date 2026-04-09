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

0. **Baby age gate** — if USER.md `Baby's age` is empty:
   - Ask: "Để mình kiểm tra sản phẩm phù hợp, bé nhà mình bao nhiêu tháng tuổi ạ?"
   - Normalize answer to integer months (see Age Normalization below), save to USER.md
   - If parent insists on skipping ("just add it"): warn once that age check was skipped, then proceed

1. **Resolve product** — query by name/stage, including age bounds:
   ```sql
   SELECT id, name, stage, price_vnd, stock_quantity, age_min_months, age_max_months
   FROM products
   WHERE name LIKE '%<keyword>%' OR slug = '<slug>' OR stage = <stage>;
   ```
   - If 0 rows → list all products and ask parent to clarify:
     ```
     Bạn muốn thêm sản phẩm nào ạ? Các sản phẩm hiện có:
     • Blackmores Newborn Formula (Stage 1 — 0–6 tháng)
     • Blackmores Follow-on Formula 2 (Stage 2 — 6–12 tháng)
     • Blackmores Toddler Milk Drink (Stage 3 — 12+ tháng)
     • Blackmores JNR Balance+ (1–10 tuổi)
     ```
   - If multiple rows → show list and ask parent to pick one

2. **Check stock** — account for quantity already in cart:
   ```sql
   SELECT p.stock_quantity,
          COALESCE(ci.quantity, 0) AS current_cart_qty,
          p.stock_quantity - COALESCE(ci.quantity, 0) AS remaining_available
   FROM products p
   LEFT JOIN cart_items ci ON ci.product_id = p.id
     AND ci.cart_id = (SELECT id FROM carts WHERE session_id = '<session_id>')
   WHERE p.id = <product_id>;
   ```
   - If `(current_cart_qty + qty_to_add) > stock_quantity`:
     - Warn: "Sản phẩm này chỉ còn [remaining_available] hộp khả dụng (bạn đã có [current_cart_qty] trong giỏ). Mình thêm [remaining_available] hộp cho bạn nhé?"
     - Cap quantity at `remaining_available`; abort if 0

3. **Age safety check** — compare baby's age against product's actual age bounds:
   - If `baby_age_months < product.age_min_months`:
     - Warn: "⚠️ [Product] được khuyến dùng từ [age_min_months] tháng tuổi. Bé hiện [baby_age] tháng tuổi — sản phẩm này chưa phù hợp. Bạn có muốn xem sản phẩm phù hợp hơn không?"
     - **Do NOT add to cart without explicit parent confirmation**
   - If `product.age_max_months IS NOT NULL AND baby_age_months > product.age_max_months`:
     - Warn: "⚠️ [Product] dành cho bé dưới [age_max_months] tháng tuổi. Bé đã [baby_age] tháng — có thể đã đến lúc chuyển giai đoạn tiếp theo."
     - Still allow adding (parent may be buying for a younger sibling)

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

6. **Show confirmation** — display what was added + cart total + allergens:
   ```sql
   -- Cart summary
   SELECT p.name, ci.quantity, p.price_vnd,
          ci.quantity * p.price_vnd AS subtotal
   FROM cart_items ci
   JOIN carts c ON ci.cart_id = c.id
   JOIN products p ON ci.product_id = p.id
   WHERE c.session_id = '<session_id>';

   -- Allergens for the added product
   SELECT allergen, presence FROM allergens WHERE product_id = <product_id>;
   ```

## Age Normalization

Convert all age inputs to integer months before any SQL or check:
- "X tuổi" → X × 12
- "X năm Y tháng" → X × 12 + Y
- "X tháng" → X
- "X tuần" → 0 (treat as newborn)
- "X ngày" → 0 (treat as newborn)
- Invalid or negative → ask again: "Bé bao nhiêu tháng tuổi ạ?"
- Save normalized integer to USER.md `Baby's age` field

## Output Format

**Web / full markdown:**
```
✅ Đã thêm vào giỏ hàng!

  Blackmores Newborn Formula (Stage 1, 900g)
  Số lượng: 2 hộp × 450,000₫ = 900,000₫
  ⚠️ Allergens: Chứa Milk, Soy.

Tổng giỏ hàng: 900,000₫

Gõ /view-cart để xem toàn bộ giỏ hàng, hoặc /place-order để đặt hàng.
```

**Zalo / Discord (no tables):**
```
✅ Đã thêm vào giỏ hàng!

• Blackmores Newborn Formula (Stage 1 · 900g)
  2 hộp × 450,000₫ = 900,000₫
  ⚠️ Allergens: Chứa Milk, Soy.

Tổng giỏ hàng: 900,000₫
Nhắn /view-cart để xem giỏ, /place-order để đặt hàng.
```

## Side Effects

- Creates a new cart row if none exists for the session
- Updates USER.md Session ID if it was newly generated (format: `sess-YYYYMMDD-XXXXXX`, 6 random `[a-z0-9]` chars)
- Updates USER.md Baby's age (integer months) if newly collected
- Updates USER.md Preferred language if first detection
