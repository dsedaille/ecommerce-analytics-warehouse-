# Warehouse Design

## Purpose

This document defines the planned dimensional model for the Olist E-Commerce Analytics Warehouse.

The warehouse is designed to support business analysis related to:

- Revenue and sales performance
- Customer behavior and repeat purchasing
- Seller performance
- Product and category performance
- Delivery and fulfillment efficiency
- Payment behavior
- Customer satisfaction

The design follows dimensional-modeling principles by separating measurable business events into fact tables and descriptive business entities into dimension tables.

Each table is defined by a clear grain to reduce ambiguity and prevent double counting.

---

# Dimension Tables

## dim_date

**Grain:** One row per calendar date

**Source:** Generated calendar table based on the date range represented in the source data

**Primary Key:** `date_key`

**Business Key:** Calendar date

**Purpose:** Provides a reusable date dimension for time-based analysis across orders, deliveries, payments, and reviews.

### Planned Columns

- `date_key`
- `full_date`
- `day_of_month`
- `day_name`
- `day_of_week`
- `week_of_year`
- `month`
- `month_name`
- `quarter`
- `year`
- `year_month`
- `is_weekend`

### Business Questions Supported

- How does revenue change by day, month, quarter, or year?
- Are order volumes higher on certain days of the week?
- How does delivery performance change over time?
- Are review scores improving or declining over time?

### Modeling Notes

- `date_key` will serve as the warehouse surrogate key.
- The date dimension will be referenced multiple times by fact tables for different business dates, such as purchase date, delivery date, and review date.
- A single reusable date dimension avoids repeatedly extracting date attributes from timestamps in analytical queries.

---

## dim_customer

**Grain:** One row per unique underlying customer represented by `customer_unique_id`

**Source:** `olist_customers_dataset.csv`

**Primary Key:** `customer_key`

**Business Key:** `customer_unique_id`

**Purpose:** Stores reusable customer attributes for repeat-purchase, geographic, retention, and customer-value analysis.

### Planned Columns

- `customer_key`
- `customer_unique_id`
- `customer_zip_code_prefix`
- `customer_city`
- `customer_state`

### Business Questions Supported

- How many unique customers purchase from the marketplace?
- What percentage of customers make repeat purchases?
- Which geographic markets contain the most customers?
- Which customers generate the most marketplace activity?
- How does customer behavior vary by state or city?

### Modeling Notes

- `customer_unique_id` should be used instead of `customer_id` for customer-level behavioral analysis.
- A single underlying customer may be associated with multiple `customer_id` values across different orders.
- `customer_key` will serve as the warehouse surrogate key.
- Geographic attributes may eventually be standardized through a separate location dimension if needed.
- If conflicting geographic values exist for the same `customer_unique_id`, a documented rule will be required to determine which location is retained.

---

## dim_product

**Grain:** One row per unique product

**Source:**  
- `olist_products_dataset.csv`
- `product_category_name_translation.csv`

**Primary Key:** `product_key`

**Business Key:** `product_id`

**Purpose:** Stores reusable descriptive product attributes for sales, category, freight, fulfillment, and customer-satisfaction analysis.

### Planned Columns

- `product_key`
- `product_id`
- `product_category_name`
- `product_category_name_english`
- `product_name_length`
- `product_description_length`
- `product_photos_qty`
- `product_weight_g`
- `product_length_cm`
- `product_height_cm`
- `product_width_cm`

### Business Questions Supported

- Which products and categories generate the most revenue?
- Which categories appear most frequently in orders?
- Which categories have the highest freight costs?
- Are certain product characteristics associated with higher shipping costs?
- Which categories receive the highest or lowest customer review scores?
- Are certain product categories more likely to experience fulfillment delays?

### Modeling Notes

- `product_key` will serve as the warehouse surrogate key.
- `product_id` preserves the original source-system identifier.
- Portuguese category names will be joined to the category translation table for English reporting.
- Missing categories will be retained using a documented fallback value such as `Unknown`.
- The two valid product categories without English translations should be preserved rather than dropped.
- Source fields containing spelling errors such as `product_name_lenght` will be renamed to cleaner warehouse-standard names such as `product_name_length`.

---

## dim_seller

**Grain:** One row per unique seller

**Source:** `olist_sellers_dataset.csv`

**Primary Key:** `seller_key`

**Business Key:** `seller_id`

**Purpose:** Stores seller attributes for revenue, operational, geographic, and fulfillment performance analysis.

### Planned Columns

- `seller_key`
- `seller_id`
- `seller_zip_code_prefix`
- `seller_city`
- `seller_state`

### Business Questions Supported

- Which sellers generate the most sales?
- Which sellers fulfill the most orders?
- Which sellers have the highest freight costs?
- Which sellers experience the most delivery delays?
- Which sellers receive the strongest or weakest review scores?
- Which states or cities contain the most marketplace sellers?

### Modeling Notes

- `seller_key` will serve as the warehouse surrogate key.
- `seller_id` preserves the original source identifier.
- Seller attributes are descriptive and should not be repeated within transactional fact tables.
- Geographic attributes may eventually be standardized using a location dimension if the geolocation data is incorporated.

---

# Fact Tables

## fact_orders

**Grain:** One row per order

**Source:**  
- `olist_orders_dataset.csv`
- `olist_customers_dataset.csv` for customer mapping
- `dim_date` for date keys

**Primary Key:** `order_key`

**Business Key:** `order_id`

**Purpose:** Stores order-level operational events and fulfillment metrics.

### Planned Foreign Keys

- `customer_key`
- `purchase_date_key`
- `approved_date_key`
- `carrier_date_key`
- `delivered_date_key`
- `estimated_delivery_date_key`

### Planned Degenerate / Business Attributes

- `order_id`
- `order_status`

### Planned Measures and Derived Fields

- `item_count`
- `delivery_days`
- `delivery_delay_days`
- `late_delivery_flag`

### Business Questions Supported

- How many orders are placed over time?
- What percentage of orders are delivered late?
- How long does fulfillment take?
- How does actual delivery compare with estimated delivery?
- Which customer groups experience the most delivery delays?
- How does order status vary over time?

### Modeling Notes

- Monetary product revenue should not be stored here because product price originates at the order-item grain.
- Payment value should not be stored here because payment records have their own grain.
- Order-level counts and fulfillment measures belong here because they describe the order as a whole.
- `order_id` may be retained as a degenerate dimension because it is analytically useful but does not require its own dimension table.
- Derived delivery metrics should be calculated consistently during transformation rather than recreated in every dashboard query.

---

## fact_order_items

**Grain:** One row per item line within an order

**Source:** `olist_order_items_dataset.csv`

**Primary Key:** `order_item_key`

**Natural / Composite Business Key:** `order_id + order_item_id`

**Purpose:** Stores product-level marketplace sales and freight activity.

### Planned Foreign Keys

- `product_key`
- `seller_key`
- `shipping_limit_date_key`

### Planned Degenerate / Business Attributes

- `order_id`
- `order_item_id`

### Planned Measures

- `price`
- `freight_value`
- `item_total_value`

### Business Questions Supported

- Which products generate the most marketplace value?
- Which categories generate the most revenue?
- Which sellers generate the most sales?
- Which categories have the highest freight costs?
- What is the average product price by category?
- Which sellers or categories contribute the largest share of marketplace sales?

### Modeling Notes

- Product revenue should be calculated from `price` at this grain.
- Freight value also originates at the order-item grain.
- `item_total_value` may be derived as:

  `price + freight_value`

- Joining this table directly to payment or review facts could create row multiplication.
- Measures should be aggregated before being combined with other one-to-many fact tables.
- The verified combination of `order_id + order_item_id` is unique in the source and represents a valid natural composite key.

---

## fact_payments

**Grain:** One row per payment sequence associated with an order

**Source:** `olist_order_payments_dataset.csv`

**Primary Key:** `payment_key`

**Natural / Composite Business Key:** `order_id + payment_sequential`

**Purpose:** Stores payment activity and payment-method behavior.

### Planned Degenerate / Business Attributes

- `order_id`
- `payment_sequential`
- `payment_type`

### Planned Measures

- `payment_installments`
- `payment_value`

### Business Questions Supported

- Which payment methods are used most frequently?
- What is the total payment value processed through each payment method?
- How often do customers use installment payments?
- What is the average number of installments by payment type?
- How many orders use multiple payment methods or payment sequences?

### Modeling Notes

- Payments should remain separate from order items because both tables can contain multiple records per order.
- Directly joining payments to item-level facts could inflate payment values and product revenue.
- `order_id + payment_sequential` is unique in the source and represents a valid natural composite key.
- Payment-method analysis should be performed at the payment grain or after order-level aggregation.

---

## fact_reviews

**Grain:** One row per source review record

**Source:** `olist_order_reviews_dataset.csv`

**Primary Key:** `review_key`

**Business Identifier:** `review_id`

**Purpose:** Stores customer satisfaction and review activity.

### Planned Foreign Keys

- `review_creation_date_key`

### Planned Degenerate / Business Attributes

- `order_id`
- `review_id`

### Planned Measures and Attributes

- `review_score`
- `review_comment_title`
- `review_comment_message`
- `review_answer_timestamp`

### Business Questions Supported

- What is the average customer review score?
- Which products or categories receive the highest and lowest review scores?
- Which sellers are associated with stronger or weaker customer satisfaction?
- How does delivery performance relate to review score?
- Are late deliveries associated with lower customer satisfaction?

### Modeling Notes

- `review_id` is not unique in the source data and should not be used as a standalone warehouse primary key.
- A surrogate `review_key` will uniquely identify each warehouse row.
- Some orders contain multiple review records.
- Review metrics should be aggregated appropriately before being joined to other fact tables.
- Missing review comments do not indicate missing reviews because customers may submit a numeric rating without text.
- Review text may be retained for future qualitative or sentiment-analysis extensions, even if the initial Power BI dashboard focuses primarily on numeric review scores.

---

# Fact Table Grain Summary

| Fact Table | Grain | Primary Business Measure |
|---|---|---|
| `fact_orders` | One row per order | Order and fulfillment metrics |
| `fact_order_items` | One row per item within an order | Price and freight |
| `fact_payments` | One row per payment sequence | Payment value |
| `fact_reviews` | One row per review record | Review score |

Keeping these grains separate reduces the risk of row multiplication and double counting.

---

# Initial Relationship Model

The planned warehouse will follow a dimensional structure in which fact tables reference reusable dimensions.

```text
                         dim_date
                            |
                            |
dim_customer ----------- fact_orders
                            |
                            |
                      fact_order_items
                       /           \
              dim_product        dim_seller


fact_payments

fact_reviews
