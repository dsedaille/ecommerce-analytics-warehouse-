# Source Data Profile

This document summarizes the structure, grain, key fields, and data-quality characteristics of the raw Olist e-commerce source files.

## Source Table Summary

| Table | Rows | Grain | Primary Key Candidate | Important Relationships | Data Quality Notes |
|---|---:|---|---|---|---|
| `olist_orders_dataset` | 99,441 | One row per order | `order_id` | `customer_id` → customers | Missing approval and delivery timestamps |
| `olist_order_items_dataset` | 112,650 | One row per item within an order | `order_id + order_item_id` | `order_id` → orders, `product_id` → products, `seller_id` → sellers | No nulls or exact duplicates |
| `olist_customers_dataset` | 99,441 | One row per customer-order identifier | `customer_id` | `customer_id` → orders | No nulls or exact duplicates |
| `olist_products_dataset` | 32,951 | One row per product | `product_id` | `product_id` → order items | Missing category and product attribute values |
| `olist_order_payments_dataset` | 103,886 | One row per payment sequence within an order | `order_id + payment_sequential` | `order_id` → orders | No nulls or exact duplicates |
| `olist_order_reviews_dataset` | 99,224 | One row per review record | `review_id` | `order_id` → orders | Large number of missing review titles and messages |
| `olist_sellers_dataset` | 3,095 | One row per seller | `seller_id` | `seller_id` → order items | No nulls or exact duplicates |
| `olist_geolocation_dataset` | 1,000,163 | One row per recorded geolocation observation | No single unique field | ZIP prefixes connect to customer and seller geography | 261,831 exact duplicate rows |
| `product_category_name_translation` | 71 | One row per translated category | `product_category_name` | Joins to products | No nulls or exact duplicates |

## Orders

**File:** `olist_orders_dataset.csv`

**Rows:** 99,441

**Grain:** One row represents one order.

### Key Fields

- `order_id` — unique order identifier
- `customer_id` — links the order to the customers table
- `order_status` — current/final order status
- `order_purchase_timestamp` — purchase date and time
- `order_approved_at` — payment/order approval timestamp
- `order_delivered_carrier_date` — date handed to carrier
- `order_delivered_customer_date` — actual customer delivery date
- `order_estimated_delivery_date` — promised delivery date

### Data Quality

Missing values:

- `order_approved_at`: 160
- `order_delivered_carrier_date`: 1,783
- `order_delivered_customer_date`: 2,965

No exact duplicate rows were found.

The missing delivery timestamps may be legitimate for orders that were canceled, unavailable, or otherwise not completed.

---

## Order Items

**File:** `olist_order_items_dataset.csv`

**Rows:** 112,650

**Grain:** One row represents one item line within an order.

### Key Fields

- `order_id`
- `order_item_id`
- `product_id`
- `seller_id`
- `shipping_limit_date`
- `price`
- `freight_value`

A likely composite primary key is:

`order_id + order_item_id`

No null values or exact duplicate rows were found.

This table will likely serve as the primary source for product-level sales and marketplace value analysis.

---

## Customers

**File:** `olist_customers_dataset.csv`

**Rows:** 99,441

**Grain:** One row per `customer_id`.

### Key Fields

- `customer_id`
- `customer_unique_id`
- `customer_zip_code_prefix`
- `customer_city`
- `customer_state`

No null values or exact duplicate rows were found.

### Important Modeling Note

`customer_id` and `customer_unique_id` represent different concepts.

`customer_id` is associated with an order, while `customer_unique_id` can be used to identify the same underlying customer across multiple orders.

This distinction will be important for repeat-customer analysis and customer lifetime metrics.

---

## Products

**File:** `olist_products_dataset.csv`

**Rows:** 32,951

**Grain:** One row per product.

**Primary Key:** `product_id`

### Data Quality

Missing values include:

- `product_category_name`: 610
- `product_name_lenght`: 610
- `product_description_lenght`: 610
- `product_photos_qty`: 610
- `product_weight_g`: 2
- `product_length_cm`: 2
- `product_height_cm`: 2
- `product_width_cm`: 2

No exact duplicate rows were found.

The category field is stored in Portuguese and can be translated using the category translation table.

---

## Payments

**File:** `olist_order_payments_dataset.csv`

**Rows:** 103,886

**Grain:** One row per payment sequence within an order.

### Key Fields

- `order_id`
- `payment_sequential`
- `payment_type`
- `payment_installments`
- `payment_value`

A likely composite primary key is:

`order_id + payment_sequential`

No null values or exact duplicate rows were found.

Because an order may have multiple payment records, this table should not be joined directly to order-item detail without considering potential row multiplication.

---

## Reviews

**File:** `olist_order_reviews_dataset.csv`

**Rows:** 99,224

**Grain:** One row per review record.

### Key Fields

- `review_id`
- `order_id`
- `review_score`
- `review_comment_title`
- `review_comment_message`
- `review_creation_date`
- `review_answer_timestamp`

### Data Quality

Missing values:

- `review_comment_title`: 87,656
- `review_comment_message`: 58,247

No exact duplicate rows were found.

Missing review text is expected because customers may provide a numeric score without leaving written feedback.

---

## Sellers

**File:** `olist_sellers_dataset.csv`

**Rows:** 3,095

**Grain:** One row per seller.

**Primary Key:** `seller_id`

### Fields

- `seller_id`
- `seller_zip_code_prefix`
- `seller_city`
- `seller_state`

No null values or exact duplicate rows were found.

---

## Geolocation

**File:** `olist_geolocation_dataset.csv`

**Rows:** 1,000,163

**Grain:** One row per geolocation observation associated with a ZIP-code prefix.

### Fields

- `geolocation_zip_code_prefix`
- `geolocation_lat`
- `geolocation_lng`
- `geolocation_city`
- `geolocation_state`

### Data Quality

The table contains:

**261,831 exact duplicate rows**

This will require cleaning before the table is used in the warehouse.

ZIP-code prefixes are not unique in this source, so the geolocation table cannot be treated as a simple one-row-per-ZIP dimension without transformation.

A future cleaning step may aggregate or select a representative latitude/longitude for each ZIP-code prefix.

---

## Product Category Translation

**File:** `product_category_name_translation.csv`

**Rows:** 71

**Grain:** One row per product category translation.

### Fields

- `product_category_name`
- `product_category_name_english`

No null values or exact duplicate rows were found.

This table will be used to translate Portuguese product categories into English for reporting.

---

## Key Modeling Risks Identified

Several source tables have different grains.

For example:

- Orders: one row per order
- Order items: one row per item within an order
- Payments: one row per payment sequence within an order
- Reviews: one row per review

Joining these tables directly can multiply rows and inflate metrics such as revenue, freight cost, payment value, or review counts.

The warehouse design must preserve table grain and aggregate data appropriately before combining metrics across these sources.
