# Relationship & Cardinality Profile

## Purpose

This document evaluates how the Olist source tables relate to one another, including relationship type, foreign-key coverage, cardinality, and potential modeling risks.

The goal is to verify the structure of the source data before designing the dimensional warehouse.

---

## Relationship Summary

| Parent Table | Child Table | Join Key | Verified Relationship | Key Finding |
|---|---|---|---|---|
| `customers` | `orders` | `customer_id` | 1:1 | Each `customer_id` is associated with one order |
| `orders` | `order_items` | `order_id` | 1:M | 9,803 orders contain multiple item records |
| `orders` | `order_payments` | `order_id` | 1:M | 2,961 orders contain multiple payment records |
| `orders` | `order_reviews` | `order_id` | Mostly 1:1, with 1:M exceptions | 547 orders contain multiple reviews |
| `products` | `order_items` | `product_id` | 1:M | 14,834 products appear in multiple order-item rows |
| `sellers` | `order_items` | `seller_id` | 1:M | 2,524 sellers are associated with multiple orders |
| `customer_unique_id` | `customer_id` | `customer_unique_id` | 1:M | 2,997 unique customers map to multiple `customer_id` records |
| `product_category_translation` | `products` | `product_category_name` | 1:M with incomplete translation coverage | 2 product categories lack English translations |

---

## Orders → Order Items

**Join Key:** `order_id`

**Verified Relationship:** One-to-many

### Findings

- 98,666 orders contain at least one item record.
- 9,803 orders contain multiple items.
- Orders contain an average of 1.14 items.
- The maximum number of items in a single order is 21.
- 775 orders have no corresponding item records.
- No orphaned order-item records were found.
- No duplicate `order_id + order_item_id` combinations were found.

### Modeling Implication

Revenue and freight metrics originate at the order-item grain.

Because one order can contain multiple items, the order-items table should remain separate from order-level facts unless it is aggregated first.

`order_id + order_item_id` is a valid composite key candidate for the order-items table.

---

## Orders → Payments

**Join Key:** `order_id`

**Verified Relationship:** One-to-many

### Findings

- 99,440 orders contain at least one payment record.
- 2,961 orders contain multiple payment records.
- The maximum number of payment records associated with one order is 29.
- Only 1 order has no corresponding payment record.
- No orphaned payment records were found.
- No duplicate `order_id + payment_sequential` combinations were found.

### Modeling Implication

Payments should remain at their own grain.

Joining payments directly to order items could multiply records when an order contains both multiple items and multiple payment records.

`order_id + payment_sequential` is a valid composite key candidate for the payments table.

---

## Orders → Reviews

**Join Key:** `order_id`

**Verified Relationship:** Mostly one-to-one, with one-to-many exceptions

### Findings

- 98,673 orders contain at least one review record.
- 768 orders have no corresponding review record.
- 547 orders contain multiple review records.
- The maximum number of reviews associated with one order is 3.
- No orphaned review records were found.
- `review_id` is not unique.
- 814 duplicate `review_id` occurrences were identified.
- 789 `review_id` values are associated with more than one order.

### Modeling Implication

The reviews table does not follow a strict one-review-per-order structure.

`review_id` should not be treated as a standalone primary key.

Reviews should likely remain at their own fact grain, and review metrics may need to be aggregated before joining to order-level or item-level data.

---

## Customers → Orders

**Join Key:** `customer_id`

**Verified Relationship:** One-to-one

### Findings

- `customer_id` is unique in the customers table.
- Every order has a matching customer record.
- No `customer_id` values are associated with multiple orders.
- The maximum number of orders associated with one `customer_id` is 1.

### Modeling Implication

At the `customer_id` level, the customers and orders tables behave as a one-to-one relationship.

However, `customer_id` should not be used to identify repeat customers because the same underlying customer may receive a different `customer_id` for different purchases.

---

## Customer Unique ID → Customer ID

**Join Key:** `customer_unique_id`

**Verified Relationship:** One-to-many

### Findings

- The dataset contains 96,096 unique customers based on `customer_unique_id`.
- 2,997 unique customers are associated with multiple `customer_id` records.
- The maximum number of `customer_id` records linked to one unique customer is 17.
- 93,099 customers are represented only once.
- 96.88% of unique customers are represented by only one `customer_id`.

### Modeling Implication

Customer analytics should use `customer_unique_id` when measuring:

- Repeat purchasing
- Customer lifetime behavior
- Purchase frequency
- Customer retention

Using `customer_id` alone would incorrectly treat repeat purchases from the same underlying customer as separate customers.

---

## Products → Order Items

**Join Key:** `product_id`

**Verified Relationship:** One-to-many

### Findings

- `product_id` is unique in the products table.
- Every product referenced in the order-items table has a matching product record.
- All 32,951 products in the products table appear in order-item data.
- 14,834 products appear in multiple order-item rows.
- The most frequently occurring product appears in 527 order-item rows.
- No orphaned product references were found.

### Modeling Implication

The products table is a strong candidate for a product dimension.

Because each product may appear in many transactions, product attributes should be stored once in the dimension and referenced from transactional fact tables.

---

## Sellers → Order Items

**Join Key:** `seller_id`

**Verified Relationship:** One-to-many

### Findings

- `seller_id` is unique in the sellers table.
- Every seller referenced in order items has a matching seller record.
- All 3,095 sellers appear in transactional order-item data.
- 2,524 sellers are associated with multiple orders.
- The most active seller is associated with 1,854 distinct orders.
- No sellers exist without corresponding order-item activity.
- No orphaned seller references were found.

### Modeling Implication

The sellers table is a strong candidate for a seller dimension.

Seller-level analysis can be performed by linking the seller dimension to order-item facts.

---

## Product Categories → Products

**Join Key:** `product_category_name`

**Verified Relationship:** One-to-many, with incomplete translation coverage

### Findings

- 610 products are missing a product category.
- 73 distinct categories appear in the products table.
- The translation table contains 71 categories.
- 13 products have a populated category but no English translation.
- Those 13 products belong to 2 untranslated categories:
  - `pc_gamer`
  - `portateis_cozinha_e_preparadores_de_alimentos`
- All translation-table categories are used by at least one product.

### Modeling Implication

The translation table provides nearly complete category coverage but requires a documented fallback rule for the two untranslated categories.

Products with missing category values should be retained and assigned an appropriate placeholder such as `Unknown` rather than dropped.

---

## Foreign-Key Integrity Checks

The following relationships were tested for orphaned records:

- `orders.customer_id` → `customers.customer_id`
- `order_items.order_id` → `orders.order_id`
- `order_items.product_id` → `products.product_id`
- `order_items.seller_id` → `sellers.seller_id`
- `order_payments.order_id` → `orders.order_id`
- `order_reviews.order_id` → `orders.order_id`

### Findings

No orphaned records were found across any of the major source relationships.

This indicates strong referential integrity across the core Olist source data.

---

## Key Modeling Risk: Grain Mismatch

Several source tables exist at different levels of detail:

- Orders: one row per order
- Order items: one row per item within an order
- Payments: one row per payment sequence
- Reviews: one row per review record

Directly joining multiple one-to-many tables can create row multiplication.

For example, an order containing:

- 3 order items
- 2 payment records

could produce 6 rows if order items and payments were joined directly at the order level.

This could incorrectly inflate measures such as:

- Revenue
- Freight cost
- Payment value
- Item counts
- Review counts

The warehouse design should preserve separate fact-table grains and aggregate measures appropriately before combining them.

---

## Key Conclusions

The profiling results support several important warehouse-design decisions:

- Preserve separate fact grains for orders, order items, payments, and reviews.
- Use `customer_unique_id` for customer-level behavior and repeat-purchase analysis.
- Use `order_id + order_item_id` as the order-items composite key.
- Use `order_id + payment_sequential` as the payments composite key.
- Do not use `review_id` as a standalone primary key.
- Retain missing and untranslated product categories using documented cleaning rules.
- Avoid direct joins between multiple one-to-many fact tables unless measures are aggregated first.

These findings will guide the next phase of the project: dimensional warehouse design.
