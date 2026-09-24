# Relationship & Cardinality Profile

## Purpose

This document evaluates how the Olist source tables relate to one another, including relationship type, foreign-key coverage, cardinality, and potential modeling risks.

The goal is to verify the structure of the source data before designing the dimensional warehouse.

---

## Relationship Summary

| Parent Table | Child Table | Join Key | Expected Relationship | Status |
|---|---|---|---|---|
| `customers` | `orders` | `customer_id` | 1:1 at the `customer_id` level | To validate |
| `orders` | `order_items` | `order_id` | 1:M | To validate |
| `orders` | `order_payments` | `order_id` | 1:M | To validate |
| `orders` | `order_reviews` | `order_id` | Mostly 1:1, possible 1:M exceptions | To validate |
| `products` | `order_items` | `product_id` | 1:M | To validate |
| `sellers` | `order_items` | `seller_id` | 1:M | To validate |
| `customer_unique_id` | `customer_id` | `customer_unique_id` | 1:M | To validate |
| `product_category_translation` | `products` | `product_category_name` | 1:M | To validate |

---

## Orders → Order Items

**Join Key:** `order_id`

**Expected Relationship:** One-to-many

### Validation Questions

- Does every order item match a valid order?
- How many orders contain more than one item?
- What is the average number of items per order?
- What is the maximum number of items in a single order?
- Are there any duplicate `order_id + order_item_id` combinations?

### Findings

_To be completed after profiling._

### Modeling Implication

Revenue and freight metrics originate at the order-item grain.

Because one order can contain multiple items, the order-items table should remain separate from order-level facts unless it is aggregated first.

---

## Orders → Payments

**Join Key:** `order_id`

**Expected Relationship:** One-to-many

### Validation Questions

- Does every payment record match a valid order?
- How many orders contain multiple payment records?
- What is the maximum number of payment records associated with one order?
- Are there any duplicate `order_id + payment_sequential` combinations?

### Findings

_To be completed after profiling._

### Modeling Implication

Payments should remain at their own grain.

Joining payments directly to order items could multiply records when an order contains both multiple items and multiple payment records.

---

## Orders → Reviews

**Join Key:** `order_id`

**Expected Relationship:** Mostly one-to-one

### Validation Questions

- Does every review match a valid order?
- How many orders have no review?
- How many orders have more than one review?
- Is `review_id` unique?
- Can a single `review_id` be associated with more than one order?

### Findings

_To be completed after profiling._

### Modeling Implication

Reviews may require their own fact table if multiple review records exist for individual orders.

Aggregating review metrics before joining them to order-level or item-level data will help prevent duplicate records.

---

## Customers → Orders

**Join Key:** `customer_id`

**Expected Relationship:** One customer record per order-level customer identifier

### Validation Questions

- Does every order have a matching `customer_id`?
- Is `customer_id` unique in the customers table?
- Can one `customer_id` appear in multiple orders?

### Findings

_To be completed after profiling._

### Important Customer Identity Note

The Olist dataset contains both:

- `customer_id`
- `customer_unique_id`

These fields represent different concepts.

`customer_id` is the identifier associated with an order, while `customer_unique_id` is intended to identify the same underlying customer across multiple purchases.

This distinction is important for repeat-purchase and customer-level analysis.

---

## Customer Unique ID → Customer ID

**Join Key:** `customer_unique_id`

**Expected Relationship:** One-to-many

### Validation Questions

- How many unique customers made more than one purchase?
- What is the maximum number of `customer_id` records associated with a single `customer_unique_id`?
- What percentage of customers appear only once?

### Findings

_To be completed after profiling._

### Modeling Implication

Customer analytics should use `customer_unique_id` when measuring repeat purchasing, customer lifetime behavior, or purchase frequency.

---

## Products → Order Items

**Join Key:** `product_id`

**Expected Relationship:** One-to-many

### Validation Questions

- Does every order-item product match a product record?
- Are there products that never appear in an order?
- How many unique products appear in transactions?
- Is `product_id` unique in the products table?

### Findings

_To be completed after profiling._

---

## Sellers → Order Items

**Join Key:** `seller_id`

**Expected Relationship:** One-to-many

### Validation Questions

- Does every order-item seller match a valid seller record?
- Are there sellers with no recorded sales?
- How many sellers fulfill multiple orders?
- Is `seller_id` unique in the sellers table?

### Findings

_To be completed after profiling._

---

## Product Categories → Products

**Join Key:** `product_category_name`

**Expected Relationship:** One-to-many

### Validation Questions

- Does every product category have an English translation?
- Are there translated categories that do not appear in the products table?
- How many products have missing category values?

### Findings

_To be completed after profiling._

---

## Foreign-Key Integrity Checks

The following relationships should be tested for orphaned records:

- `orders.customer_id` → `customers.customer_id`
- `order_items.order_id` → `orders.order_id`
- `order_items.product_id` → `products.product_id`
- `order_items.seller_id` → `sellers.seller_id`
- `order_payments.order_id` → `orders.order_id`
- `order_reviews.order_id` → `orders.order_id`
- `products.product_category_name` → `product_category_translation.product_category_name`

### Findings

_To be completed after profiling._

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

could produce 6 rows if the two tables were joined directly at the order level.

This could incorrectly inflate measures such as:

- Revenue
- Freight cost
- Payment value
- Item counts
- Review counts

The warehouse design should preserve separate fact-table grains and aggregate measures appropriately before combining them.

---

## Next Step

Run cardinality and foreign-key integrity checks against each relationship.

The verified results from those checks will be added to this document and used to guide the dimensional warehouse design.
