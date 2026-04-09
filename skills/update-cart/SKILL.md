# Skill: update-cart ✏️

Update item quantity or remove an item from the cart.

## Triggers

- User wants to change quantity: "đổi số lượng", "change quantity", "tôi muốn 3 hộp"
- User wants to remove an item: "xóa", "bỏ ra", "remove", "I don't want X anymore"
- User wants to clear entire cart: "xóa hết", "clear cart", "bỏ tất cả"

## Inputs Required

- **Session ID:** from USER.md
- **Product:** which product to update (resolve to product_id)
- **New quantity:** 0 means remove the item

## Steps

1. **Resolve product** — find product_id from user's description:
   ```sql
   SELECT id, name FROM products WHERE name LIKE '%<keyword>%' OR stage = <stage>;
   ```

2. **If quantity > 0 — update:**
   ```sql
   -- Check stock first
   SELECT stock_quantity FROM products WHERE id = <product_id>;

   -- Update quantity
   UPDATE cart_items SET quantity = <new_qty>
   WHERE cart_id = (SELECT id FROM carts WHERE session_id = '<session_id>')
     AND product_id = <product_id>;
   ```

3. **If quantity = 0 — remove:**
   ```sql
   DELETE FROM cart_items
   WHERE cart_id = (SELECT id FROM carts WHERE session_id = '<session_id>')
     AND product_id = <product_id>;
   ```

4. **Clear entire cart (if requested):**
   ```sql
   DELETE FROM cart_items
   WHERE cart_id = (SELECT id FROM carts WHERE session_id = '<session_id>');
   ```

5. **Show updated cart** — re-run the view-cart query to display current state

## Output Format

After update:
```
✅ Đã cập nhật giỏ hàng!

[show updated cart table — same format as /view-cart]
```

After removal:
```
🗑️ Đã xóa Blackmores Newborn Formula khỏi giỏ hàng.

[show remaining cart or "Giỏ hàng đang trống" if last item removed]
```

## Edge Cases

- Product not in cart → "Sản phẩm này không có trong giỏ hàng của bạn."
- New quantity exceeds stock → warn and set to max available
