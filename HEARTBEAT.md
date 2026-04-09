# HEARTBEAT.md

## Product database checks

- **DB health:** Run `sqlite3 data/products.db "SELECT COUNT(*) FROM products"` — should return `4`. If not, database may be missing or corrupted.
- **Unanswered questions?** Check `memory/` files for any parent questions from last session that were left unresolved.
- **Product data fresh?** Note any need to update product info (e.g. new products added, prices, pack size changes) — flag to user if so.
