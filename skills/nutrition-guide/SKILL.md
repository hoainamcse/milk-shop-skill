---
name: nutrition-guide
description: Full nutritional breakdown for a Blackmores baby milk product — vitamins, minerals, fatty acids, prebiotics, and what each nutrient does for the baby.
user-invocable: true
metadata: {"openclaw": {"emoji": "🌿"}}
---

# Nutrition Guide

Provide a complete nutritional profile for a specific Blackmores product, explaining what each key nutrient does.

## When to trigger

- User says "what's in [product]", "nutrition", "ingredients", "dinh dưỡng", "thành phần"
- User asks about a specific nutrient: "does it have DHA?", "what vitamins does it have?"
- User wants to understand the benefits of a product in detail

## Steps

### 1. Identify the product
Resolve from user input:
- By name: "Newborn", "Follow-on", "Toddler", "JNR Balance+"
- By stage: "Stage 1", "Stage 2", "Stage 3"
- By age: infer stage from baby's age

```bash
sqlite3 /home/nhhnmm/.openclaw/workspace/data/products.db \
  "SELECT id, name, stage, age_min_months, age_max_months, description FROM products
   WHERE slug = '{slug}' OR name LIKE '%{keyword}%';"
```

### 2. Query all nutrients grouped by category
```bash
sqlite3 /home/nhhnmm/.openclaw/workspace/data/products.db \
  "SELECT name, category FROM nutrients WHERE product_id = {id} ORDER BY category, name;"
```

### 3. Query all benefits grouped by body system
```bash
sqlite3 /home/nhhnmm/.openclaw/workspace/data/products.db \
  "SELECT benefit, body_system FROM benefits WHERE product_id = {id} ORDER BY body_system;"
```

### 4. Query allergens
```bash
sqlite3 /home/nhhnmm/.openclaw/workspace/data/products.db \
  "SELECT allergen, presence FROM allergens WHERE product_id = {id};"
```

## Nutrient explanation reference

Use these explanations when presenting nutrients to parents:

| Nutrient | Why it matters |
|----------|---------------|
| DHA | Omega-3 fatty acid essential for brain and eye development in infancy |
| ARA | Omega-6 fatty acid, works with DHA for brain development |
| GOS prebiotic | Promotes healthy gut bacteria — similar to oligosaccharides in breast milk |
| FOS prebiotic | Another prebiotic fiber; supports digestive health alongside GOS |
| Taurine | Amino acid found naturally in breast milk; supports brain development |
| Nucleotides | Building blocks for DNA/RNA; support immune function and gut development |
| Phosphatidylserine (PS) | Supports memory, learning, and cognitive function in children |
| B. lactis HN019 (probiotic) | Clinically studied probiotic; supports digestive and immune health |
| Iron | Critical for cognitive development and preventing anaemia |
| Iodine | Essential for thyroid function and brain development |
| Zinc | Key for immune function and growth |
| Calcium | Builds strong bones and teeth |
| Vitamin D | Needed to absorb calcium; supports bone health and immunity |
| Vitamin A | Immune function and vision |
| Vitamin C | Antioxidant; helps absorb iron; immune support |

## Output format

```
🌿 Nutritional Profile: [Product Name]
For babies aged [X–X months]

**Overview:**
[1-2 sentence product description]

**Fatty Acids:**
- DHA — [explanation]
- ARA — [explanation]

**Prebiotics & Probiotics:**
- GOS — [explanation]
- [others]

**Vitamins:**
- Vitamin A, B-complex (B1–B12, Folate), C, D, E, K — [brief note on collective role]

**Minerals:**
- Iron, Zinc, Calcium, Magnesium, Iodine, and more — [brief note]

**Benefits summary:**
| Body System | Benefit |
|-------------|---------|
| Brain & cognition | ... |
| Immune | ... |
| Bones & teeth | ... |
| Digestion | ... |
| Growth | ... |

⚠️ Allergens: [list]

📦 Product page: [URL]
```

## Important rules

- Always use database data — do not invent nutrient amounts or percentages not in the database
- Explain nutrients in plain language parents can understand
- Group nutrients logically (fatty acids → prebiotics → vitamins → minerals)
- If a parent asks about a specific nutrient not in the database, acknowledge the limitation honestly
