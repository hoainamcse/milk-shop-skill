# Skill: order-history 📋

View past orders for the current session. Can show a list of all orders or detail of a specific order by order number.

## Triggers

- User asks about past orders: "lịch sử đơn hàng", "đơn hàng của tôi", "my orders"
- User asks about a specific order: "đơn ORD-20260409-001", "order status", "trạng thái đơn hàng"
- User asks "has my order shipped?", "khi nào giao hàng?"

## Inputs Required

- **Session ID:** from USER.md
- **Order number (optional):** if user mentions a specific order

## Steps

### List all orders (no order number given)
```sql
SELECT o.order_number,
       o.status,
       o.total_vnd,
       o.created_at,
       GROUP_CONCAT(oi.product_name || ' ×' || oi.quantity, ' | ') AS items
FROM orders o
JOIN order_items oi ON o.id = oi.order_id
WHERE o.session_id = '<session_id>'
GROUP BY o.id
ORDER BY o.created_at DESC;
```

### Get specific order detail (order number given)
```sql
SELECT o.order_number, o.status, o.customer_name, o.customer_phone,
       o.delivery_address, o.notes, o.total_vnd, o.created_at, o.updated_at,
       oi.product_name, oi.quantity, oi.unit_price_vnd, oi.subtotal_vnd
FROM orders o
JOIN order_items oi ON o.id = oi.order_id
WHERE o.order_number = '<order_number>';
```

## Output Format — Order List

```
📋 Lịch sử đơn hàng của bạn

┌──────────────────┬─────────────┬────────────┬──────────────────────────────┐
│ Mã đơn hàng      │ Trạng thái  │ Tổng tiền  │ Sản phẩm                     │
├──────────────────┼─────────────┼────────────┼──────────────────────────────┤
│ ORD-20260409-001 │ pending     │ 900,000₫   │ Blackmores Newborn ×2        │
│ ORD-20260408-003 │ delivered   │ 420,000₫   │ Blackmores Toddler ×1        │
└──────────────────┴─────────────┴────────────┴──────────────────────────────┘

Nhập mã đơn hàng để xem chi tiết (ví dụ: ORD-20260409-001).
```

## Output Format — Order Detail

```
📦 Chi tiết đơn hàng: ORD-20260409-001
────────────────────────────────
Trạng thái  : ⏳ Đang chờ xác nhận (pending)
Ngày đặt    : 09/04/2026 14:32

Sản phẩm:
  • Blackmores Newborn Formula × 2
    450,000₫ × 2 = 900,000₫

Tổng cộng   : 900,000₫

Giao hàng đến: 123 Nguyễn Huệ, Quận 1
Liên lạc     : Nguyễn Văn A — 0912 345 678
────────────────────────────────
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

- **No orders found:** "Bạn chưa có đơn hàng nào. Hãy để mình giúp bạn tìm sản phẩm phù hợp!"
- **Order number not found:** "Không tìm thấy đơn hàng [number]. Bạn có thể kiểm tra lại mã đơn hàng không?"
- **No session ID:** "Mình chưa lưu thông tin phiên của bạn. Hãy thêm sản phẩm vào giỏ hàng trước nhé!"
