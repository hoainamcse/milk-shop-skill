# Skill: view-cart 🛒

Display the current session's cart contents with prices and totals.

## Triggers

- User asks to see their cart
- "xem giỏ hàng", "giỏ hàng của tôi", "tôi đang có gì trong giỏ"
- "what's in my cart", "show my cart", "review my order"

## Inputs Required

- **Session ID:** from USER.md

## Steps

1. **Check session ID** — if missing, inform user their cart is empty (no session = no cart)

2. **Query cart** — include `price_usd` from DB for accurate USD totals:
   ```sql
   SELECT p.name, p.stage, p.pack_size_g, p.source_type,
          ci.quantity,
          p.price_vnd AS unit_price,
          p.price_usd AS unit_price_usd,
          ci.quantity * p.price_vnd AS subtotal_vnd,
          ci.quantity * p.price_usd AS subtotal_usd
   FROM cart_items ci
   JOIN carts c ON ci.cart_id = c.id
   JOIN products p ON ci.product_id = p.id
   WHERE c.session_id = '<session_id>'
   ORDER BY p.stage NULLS LAST;
   ```

3. **Calculate grand total** from subtotals

4. **Handle empty cart** — if no rows returned, say cart is empty

## Output Format

**Web (full markdown — table OK):**
```
🛒 Giỏ hàng của bạn

| Sản phẩm                    | SL | Đơn giá   | Thành tiền |
|-----------------------------|----|-----------|-----------|
| Blackmores Newborn Formula  | 2  | 450,000₫  | 900,000₫  |
| (Stage 1 · 900g)            |    |           |           |

**TỔNG CỘNG: 900,000₫** (~$36.00 USD)

Dùng /update-cart để thay đổi số lượng, hoặc /place-order để đặt hàng.

_Giá hiển thị là giá hiện tại. Giá chính thức được chốt tại thời điểm đặt hàng._
```

**Zalo / Discord (no tables — bullet list):**
```
🛒 Giỏ hàng của bạn

• Blackmores Newborn Formula (Stage 1 · 900g)
  2 hộp × 450,000₫ = 900,000₫

Tổng cộng: 900,000₫ (~$36.00 USD)

Nhắn /update-cart để thay đổi, /place-order để đặt hàng.
(Giá hiển thị là giá hiện tại — chốt giá khi xác nhận đặt hàng.)
```

## Edge Cases

- **Empty cart:** "Giỏ hàng của bạn đang trống. Hãy để mình giúp bạn tìm sản phẩm phù hợp cho bé nhé!"
- **Multiple items:** show each product on its own row
