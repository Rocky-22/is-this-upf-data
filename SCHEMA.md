# Schema

Schema version **2**, stored in `PRAGMA user_version` and `meta.schema_version`. A change to any table or column changes the version.

## `meta`

Key/value text about the build.

| Key | Meaning |
|---|---|
| `db_version` | Open Food Facts snapshot date + rules version, e.g. `20261004-0.4.0` |
| `schema_version` | As above |
| `built_at` | Build time (UTC) |
| `source`, `source_snapshot_date`, `source_snapshot_commit` | The Open Food Facts snapshot (Hugging Face `openfoodfacts/product-database` commit) |
| `ruleset_version` | Version of the classification rules |
| `attribution`, `licence` | Attribution text and licence (`ODbL-1.0`) |
| `product_count` | Number of products |

## `products`

| Column | Notes |
|---|---|
| `id` | Integer key. Ordered by popularity and **not stable between builds**; use `barcode` to refer to a product. |
| `barcode` | As Open Food Facts stores it (EAN-13, EAN-8; UPC-A is zero-padded to 13 digits). Unique. |
| `name`, `generic_name`, `brand`, `quantity` | Display text. `name_is_english` is 0 when only a non-English name exists. |
| `retailer` | Own-label retailer (e.g. `Tesco`), else NULL. |
| `stores` | UK supermarket chains it's recorded at, comma-separated. |
| `nova_group` | 1–4, or NULL when unknown. |
| `confidence` | `high`, `medium` or `low` (NULL when unknown). |
| `unknown_reason` | When `nova_group` is NULL: `no_ingredients`, `not_english`, `unparseable` or `out_of_scope` (alcoholic drinks). |
| `ingredients_text` | The ingredients list as displayed. |
| `popularity` | Open Food Facts unique scans. |
| `off_last_modified` | Unix time of the last Open Food Facts edit. |

## `rules`

The ingredients and additive categories the classification recognises.

| Column | Notes |
|---|---|
| `id` | Integer key used by `product_matches` (not stable between builds). |
| `rule_id` | Stable identifier, e.g. `e471`, `flavouring`. |
| `kind` | `category` (a label category such as "Emulsifier"), `additive` or `ingredient`. |
| `name`, `reason` | Plain-English name and explanation. |
| `marker` | `yes` (marks ultra-processing), `no`, or `contextual` (a marker only when the label declares it under a marker category). |
| `marker_class`, `certainty`, `contested` | Class (e.g. `emulsifier`), `high`/`medium` certainty, and 1 where sources disagree. |
| `e_numbers` | Comma-separated E-numbers. |
| `sources` | JSON array of short source citations. |

## `product_matches`

Why each product got its group.

| Column | Notes |
|---|---|
| `product_id` | → `products.id` |
| `is_marker` | 1 = a marker of ultra-processing (only group 4 products have them). 0 = a recognised non-marker ingredient (salt, sugar, citric acid…), stored for groups 1–3. |
| `position` | Markers: the order to list them in. Recognised: order in the text. |
| `rule`, `declared_as` | → `rules.id`; `declared_as` is the label category it was listed under, if any. |
| `matched_text` | The label's wording. |
| `start`, `end` | Offsets into `ingredients_text` in Unicode code points: `substr(ingredients_text, start + 1, end - start)` = `matched_text`. |

```sql
-- Why is a product ultra-processed?
SELECT r.name, m.matched_text, r.reason
FROM products p
JOIN product_matches m ON m.product_id = p.id AND m.is_marker = 1
JOIN rules r ON r.id = m.rule
WHERE p.barcode = '5000157024671'
ORDER BY m.position;
```

## Search: `products_fts` and `words`

`products_fts` is a contentless FTS5 table (trigram tokenizer, `rowid` = `products.id`) over `name`, `brand` and `generic_name`. It holds a normalised **search key**, not the display text:

- Unicode NFKD with combining marks removed, lower case.
- Apostrophes deleted, `&` → `and`, any other run of non-alphanumerics → one space.
- Then `m and s` → `marks and spencer` and `co op` → `coop`.
- Every word under 3 letters is also stored joined to its neighbour (`pg tips` → `pg tips pgtips`).

Normalise queries the same way and match words of 3+ letters as substrings:

```sql
SELECT p.name, p.brand
FROM products_fts f JOIN products p ON p.id = f.rowid
WHERE products_fts MATCH '"heinz" AND "beans"'
ORDER BY bm25(products_fts, 10.0, 6.0, 1.0) - ln(1 + p.popularity) * 0.6
LIMIT 10;
```

`words` (`word`, `weight`) is the vocabulary of name and brand words, weighted by how many products, and how many scans, use them. The app uses it for spelling correction.
