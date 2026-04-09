---
name: recommend-formula
description: Recommend the right Blackmores formula for a baby based on age in months. Returns the appropriate stage product with key benefits and allergen info.
user-invocable: true
metadata: {"openclaw": {"emoji": "🍼"}}
---

# Recommend Formula

Given a baby's age, find the appropriate Blackmores formula product and explain why it's the right choice.

## When to trigger

- User says "recommend formula", "which milk for my baby", "sữa nào cho bé", "what formula for X months"
- User provides a baby's age and asks what to use
- User is transitioning between stages and wants guidance

## Steps

### 1. Determine the baby's age
If not already provided, ask: "How old is your baby? (in months)"

### 2. Query matching products from database
```bash
sqlite3 /home/nhhnmm/.openclaw/workspace/data/products.db \
  "SELECT id, name, stage, age_min_months, age_max_months, pack_size_g, description, url
   FROM products
   WHERE age_min_months <= {age} AND (age_max_months IS NULL OR age_max_months >= {age})
   ORDER BY stage NULLS LAST;"
```

### 3. Query benefits for each matching product
```bash
sqlite3 /home/nhhnmm/.openclaw/workspace/data/products.db \
  "SELECT p.name, b.benefit, b.body_system
   FROM benefits b JOIN products p ON b.product_id = p.id
   WHERE p.id IN ({matching_ids})
   ORDER BY p.stage, b.body_system;"
```

### 4. Query allergens for each matching product
```bash
sqlite3 /home/nhhnmm/.openclaw/workspace/data/products.db \
  "SELECT p.name, a.allergen, a.presence
   FROM allergens a JOIN products p ON a.product_id = p.id
   WHERE p.id IN ({matching_ids});"
```

## Stage transition rules

| Baby age | Primary recommendation | Notes |
|----------|----------------------|-------|
| 0–6 months | Stage 1: Newborn Formula | Only if not breastfeeding or supplementing |
| 6–12 months | Stage 2: Follow-on Formula 2 | As solids are introduced (~6 months) |
| 12–36 months | Stage 3: Toddler Milk Drink | Supplementary — solid foods are primary nutrition |
| 1–10 years | JNR Balance+ | Supplement drink for nutritional gaps, not a formula replacement |

**At 12 months, both Stage 3 AND JNR Balance+ may be relevant** — clarify the parent's goal:
- Stage 3 = daily milk drink supplement
- JNR Balance+ = targeted nutritional supplement for picky eaters / malnourished children

## Output format

Start with the primary recommendation, then explain:

```
🍼 For a [X]-month-old baby:

**Recommended:** [Product Name] (Stage X)
- Age range: X–X months
- Pack size: Xg
- Why: [2-3 sentence rationale]

**Key benefits:**
- [benefit 1]
- [benefit 2]
- [benefit 3]

⚠️ Allergens: Contains [milk, soy]. [May contain fish if JNR Balance+]

📦 More info: [URL]

---
💡 Always consult your pediatrician (bác sĩ nhi khoa) before starting or switching formula, especially if your baby has health concerns.
```

## If multiple products match (e.g. age 18 months → Stage 3 + JNR Balance+)

Explain the difference:
- Stage 3 is a daily milk drink (sữa cho trẻ em)
- JNR Balance+ is a nutritional supplement for children with dietary gaps

Ask: "Is your toddler a picky eater or have nutritional concerns, or are you looking for a daily milk drink?"

## Important rules

- Never recommend a Stage 2 or higher product for a baby under 6 months
- Always mention allergens
- Always end with a recommendation to consult a healthcare professional for medical concerns
- Do not invent or guess nutrient amounts — query the database
