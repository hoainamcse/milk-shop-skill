---
name: feeding-guide
description: Step-by-step formula preparation guide and daily serving schedule for a Blackmores baby milk product.
user-invocable: true
metadata: {"openclaw": {"emoji": "📋"}}
---

# Feeding Guide

Provide safe preparation instructions and daily serving guidance for a Blackmores formula product.

## When to trigger

- User says "how to prepare", "how to make", "cách pha", "pha sữa như thế nào"
- User asks "how many scoops", "how much water", "how many times a day"
- User wants feeding schedule or serving frequency

## Steps

### 1. Identify the product
Resolve from user context (current product, baby's age, or explicit name).

### 2. Query preparation steps
```bash
sqlite3 /home/nhhnmm/.openclaw/workspace/data/products.db \
  "SELECT step_order, instruction FROM preparation_steps
   WHERE product_id = {id} ORDER BY step_order;"
```

### 3. Query basic product info
```bash
sqlite3 /home/nhhnmm/.openclaw/workspace/data/products.db \
  "SELECT name, age_min_months, age_max_months, description FROM products WHERE id = {id};"
```

## General feeding safety rules (always include)

1. **Always prepare fresh** — never use leftover formula from a previous feeding
2. **Boil water** — always use boiled water cooled to room temperature (or as directed on pack)
3. **Sterilize equipment** — bottles, teats, and measuring spoons must be sterilized before use
4. **Check temperature** — test on inner wrist before feeding; should feel lukewarm, not hot
5. **Follow tin instructions** — always use the correct scoop size from the product tin; do not pack or heap
6. **Discard unused formula** — any prepared formula not consumed within 1 hour at room temperature should be discarded
7. **Store powder correctly** — keep tin sealed, store in a cool dry place, use within the period shown on tin after opening

## Serving frequency reference

| Product | Age | Servings per day |
|---------|-----|-----------------|
| Stage 1 (Newborn) | 0–6 months | On-demand (typically 6–8 feeds/day for newborns, decreasing as baby grows) |
| Stage 2 (Follow-on) | 6–12 months | ~4–5 feeds/day alongside solid foods starting ~6 months |
| Stage 3 (Toddler) | 12+ months | 2–3 cups/day as part of a varied diet |
| JNR Balance+ | 1–3 years | 2 servings/day |
| JNR Balance+ | 4–8 years | 3 servings/day |
| JNR Balance+ | 9–10 years | 4 servings/day |

*Exact amounts per feed: refer to the feeding guide printed on the product tin, as it varies by baby weight and age.*

## Output format

```
📋 Feeding Guide: [Product Name]
For [age range]

**How to prepare:**

1. [Step 1]
2. [Step 2]
...

**Daily serving guide:**
[Age-appropriate serving frequency from reference above]

**Important safety reminders:**
- Always prepare fresh formula for each feeding
- Boil water, cool before use
- Sterilize bottles and equipment
- Check temperature before feeding
- Discard unused formula

💡 For exact scoop-to-water ratios, refer to the feeding guide printed on the product tin — these vary by baby's age and weight.

💊 If your baby has special health needs, consult your pediatrician (bác sĩ nhi khoa) before adjusting feeding amounts.
```

## Important rules

- Always include food safety reminders — preparation hygiene is critical for infant safety
- Be clear that exact scoops/water ratios are on the product tin (we don't have those specific numbers in the database)
- Never encourage overfeeding — reference professional advice for amounts
- For Stage 1 (newborn formula), emphasize on-demand feeding approach
