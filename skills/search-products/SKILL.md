---
name: search-products
description: Search the Blackmores product database by criteria — age, stage, benefit, nutrient, allergen, or keyword.
user-invocable: true
metadata: {"openclaw": {"emoji": "🔍"}}
---

# Search Products

Query the product database to find products matching specific criteria.

## When to trigger

- User says "search", "find", "tìm sữa", "list products", "show all products"
- User asks "which products have DHA?", "which is safe for soy allergy?"
- User wants to browse or filter the product catalog

## Steps

### 1. Understand the search intent
Parse the user's request into one of these modes:
- **By age:** "products for 8-month-old"
- **By stage:** "Stage 2 products"
- **By benefit:** "products for immune support", "products for brain development"
- **By nutrient:** "products with probiotics", "products with DHA"
- **By allergen (exclude):** "products without soy", "soy-free formula"
- **All products:** "show everything", "list all"

### 2. Execute the appropriate query

#### All products
```bash
sqlite3 data/products.db \
  "SELECT name, stage, age_min_months, age_max_months, pack_size_g, source_type
   FROM products ORDER BY age_min_months;"
```

#### By age (months)
```bash
sqlite3 data/products.db \
  "SELECT name, stage, age_min_months, age_max_months, pack_size_g
   FROM products
   WHERE age_min_months <= {age} AND (age_max_months IS NULL OR age_max_months >= {age})
   ORDER BY stage NULLS LAST;"
```

#### By benefit keyword
```bash
sqlite3 data/products.db \
  "SELECT DISTINCT p.name, p.stage, b.benefit
   FROM benefits b JOIN products p ON b.product_id = p.id
   WHERE b.benefit LIKE '%{keyword}%' OR b.body_system = '{system}'
   ORDER BY p.stage NULLS LAST;"
```

#### By nutrient keyword
```bash
sqlite3 data/products.db \
  "SELECT DISTINCT p.name, p.stage, n.name as nutrient
   FROM nutrients n JOIN products p ON n.product_id = p.id
   WHERE n.name LIKE '%{keyword}%'
   ORDER BY p.stage NULLS LAST;"
```

#### Products WITHOUT a specific allergen
```bash
sqlite3 data/products.db \
  "SELECT p.name, p.stage, p.age_min_months, p.age_max_months
   FROM products p
   WHERE p.id NOT IN (
     SELECT product_id FROM allergens
     WHERE allergen LIKE '%{allergen}%' AND presence = 'contains'
   )
   ORDER BY p.stage NULLS LAST;"
```

#### All allergens across products
```bash
sqlite3 data/products.db \
  "SELECT p.name, a.allergen, a.presence
   FROM allergens a JOIN products p ON a.product_id = p.id
   ORDER BY p.stage NULLS LAST, a.presence;"
```

## Body system keyword mapping

| User says | body_system value |
|-----------|-------------------|
| brain, cognitive, cognitive development, trí não | cognitive |
| immune, immunity, đề kháng | immune |
| bone, teeth, xương | bone |
| growth, tăng trưởng | growth |
| digestion, gut, tiêu hóa | digestion |

## Post-query: may_contain allergen check

After any allergen exclusion query, run a secondary check on returned products to surface `may_contain` warnings:

```bash
sqlite3 data/products.db \
  "SELECT p.name, a.allergen FROM allergens a JOIN products p ON a.product_id = p.id
   WHERE p.id IN ({returned_ids})
     AND a.allergen LIKE '%{searched_allergen}%'
     AND a.presence = 'may_contain';"
```

If any rows returned → append to the product entry:
`⚠️ [Product] có thể chứa dấu vết của [allergen] — không phù hợp với dị ứng nghiêm trọng.`

## Zero-result handling for allergen exclusions

If an allergen exclusion query returns 0 results:
- **Do NOT** say "broaden the search criteria"
- Instead: "⚠️ Không có sản phẩm Blackmores nào trong danh mục hiện tại hoàn toàn không chứa [allergen]. Tất cả công thức Blackmores đều có thành phần nguồn gốc từ sữa bò và đậu nành. Nếu bé có dị ứng nghiêm trọng, hãy tham khảo bác sĩ nhi khoa để tìm sản phẩm phù hợp hơn nhé."

## Output format

**Web (full markdown — table OK):**
```
🔍 Search results: [criteria]

| Product | Stage | Age | Pack | [relevant column] |
|---------|-------|-----|------|-------------------|
| ... | ... | ... | ... | ... |

[X] product(s) found.

💡 Use /recommend-formula to get a full recommendation for a specific age.
💡 Use /compare-products to see a side-by-side comparison.
💡 Use /nutrition-guide for full nutrient details on a product.
```

**Zalo / Discord (no tables — bullet list):**
```
🔍 Kết quả tìm kiếm: [tiêu chí]

• [Product A] — Stage X — [Age range] — [Pack]g
  [relevant info, e.g. benefit or nutrient]

• [Product B] — Stage Y — ...

[X] sản phẩm phù hợp.

💡 /recommend-formula để được tư vấn theo độ tuổi.
💡 /compare-products để so sánh sản phẩm.
```

## Important rules

- Always confirm the search was run against the database, not guessed from memory
- For allergen exclusion queries: always run the `may_contain` secondary check on returned products
- Zero allergen results = catalog limitation, not a search issue — explain clearly
- Link to other skills for deeper exploration
