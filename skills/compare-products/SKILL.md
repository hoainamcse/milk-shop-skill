---
name: compare-products
description: Side-by-side comparison of two or more Blackmores baby milk products — nutrients, benefits, allergens, pack size, and age range.
user-invocable: true
metadata: {"openclaw": {"emoji": "📊"}}
---

# Compare Products

Generate a clear side-by-side comparison of two or more Blackmores products.

## When to trigger

- User says "compare", "so sánh", "what's the difference between", "Stage 1 vs Stage 2", "which is better"
- User is deciding between two products and wants to see them side-by-side

## Steps

### 1. Identify which products to compare
If not specified, ask. Products can be referenced by:
- Stage number: "Stage 1 vs Stage 2"
- Name: "Newborn Formula vs Follow-on"
- Age: "formula for 3 months vs 9 months"
- Slug: 'blackmores-newborn-formula', 'blackmores-follow-on-2', 'blackmores-toddler-milk-drink-3', 'blackmores-jnr-balance'

### 2. Query product basics
```bash
sqlite3 data/products.db \
  "SELECT id, name, stage, age_min_months, age_max_months, pack_size_g, source_type, url
   FROM products WHERE id IN ({ids}) ORDER BY stage NULLS LAST;"
```

### 3. Query nutrients for both products
```bash
sqlite3 data/products.db \
  "SELECT product_id, name, category FROM nutrients
   WHERE product_id IN ({ids}) ORDER BY category, name;"
```

### 4. Query benefits for both products
```bash
sqlite3 data/products.db \
  "SELECT product_id, benefit, body_system FROM benefits
   WHERE product_id IN ({ids}) ORDER BY body_system;"
```

### 5. Query allergens for both products
```bash
sqlite3 data/products.db \
  "SELECT product_id, allergen, presence FROM allergens WHERE product_id IN ({ids});"
```

## Output format

**Web (full markdown — table OK):**
```
📊 Product Comparison

| | [Product A] | [Product B] |
|---|---|---|
| **Stage** | Stage 1 | Stage 2 |
| **Age** | 0–6 months | 6–12 months |
| **Pack size** | 900g | 900g |
| **Type** | Infant formula | Follow-on formula |
| **DHA** | ✅ | ✅ |
| **ARA** | ✅ | ✅ |
| **GOS prebiotic** | ✅ | ✅ |
| **Taurine** | ✅ | ✅ |
| **Nucleotides** | ✅ | ✅ |
| **Phosphatidylserine** | ❌ | ❌ |
| **Probiotic** | ❌ | ❌ |
| **Allergens** | Milk, Soy | Milk, Soy lecithin |

**Key differences:**
- [Explain the main practical difference in 2-3 sentences]

**When to choose [Product A]:** [scenario]
**When to choose [Product B]:** [scenario]
```

**Zalo / Discord (no tables — bullet list):**
```
📊 So sánh sản phẩm

[Product A] (Stage X · X–X tháng · Xg)
• DHA: ✅  ARA: ✅  GOS: ✅  Taurine: ✅
• Allergens: Milk, Soy

[Product B] (Stage Y · X–X tháng · Xg)
• DHA: ✅  ARA: ✅  GOS: ✅  Taurine: ✅
• Allergens: Milk, Soy lecithin

Điểm khác biệt chính:
• [Key difference 1]
• [Key difference 2]

Chọn [Product A] khi: [scenario]
Chọn [Product B] khi: [scenario]
```

If comparing a formula (Stage 1–3) with JNR Balance+, highlight that:
- Formulas are designed as a milk source (can be sole nutrition or supplementary)
- JNR Balance+ is a targeted supplement drink (not a milk replacement)

## Important rules

- Pull all nutrient/benefit data from the database — do not rely on memory
- Present nutrients grouped by category (vitamins, minerals, fatty acids, etc.)
- Always include allergen comparison row
- Include product URLs at the bottom for reference
