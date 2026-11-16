# Gold Layer Data Dictionary

This document describes the six Gold tables created in the Olist e-commerce data engineering project.

**Storage:** Databricks `workspace.gold`  
**Published copy:** BigQuery `databricksproject-510805.db_olist_analytics`

> Grain means what one row represents. Column descriptions below follow the transformations in the project notebook. Where a table inherits source columns, the dictionary calls that out rather than guessing at every source field.

## 1. `gold_sales`

**Grain:** One row per order item (`order_id` + `order_item_id`).  
**Purpose:** Item-level sales analysis, including order, product and seller context.

| Column | Meaning |
|---|---|
| `order_id` | Order identifier |
| `customer_id` | Customer identifier used in the source order data |
| `customer_unique_id` | Stable customer identifier added from the Silver customer table |
| `order_status` | Current order status |
| `order_purchase_timestamp` | Timestamp when the order was placed |
| `order_cancellation_stage` | Derived stage at which an order was cancelled |
| `product_category_name` | Product category |
| `order_item_id` | Item sequence/identifier within an order |
| `product_id` | Product identifier |
| `seller_id` | Seller identifier |
| `price` | Item price |
| `freight_value` | Freight amount for the item |
| `seller_city` | Seller city |
| `seller_state` | Seller state |
| `item_total_value` | `price + freight_value`, rounded to 2 decimal places |
| `order_purchase_date` | Date extracted from purchase timestamp |
| `order_purchase_month` | Month number extracted from purchase date |

## 2. `gold_customers`

**Grain:** One row per `customer_unique_id`.  
**Purpose:** Customer-level profile and purchasing metrics.

| Column | Meaning |
|---|---|
| `customer_unique_id` | Unique customer identifier; table key |
| `customer_city` | Customer city |
| `customer_state` | Customer state |
| `customer_zip_code_prefix` | Customer ZIP-code prefix |
| `total_orders` | Distinct number of orders |
| `total_items` | Number of order-item records |
| `total_spend` | Sum of item prices |
| `total_freight` | Sum of freight values |
| `total_order_value` | Sum of item price plus freight |
| `canceled_orders` | Distinct orders counted as cancelled by the status condition in the notebook |
| `first_order_date` | Earliest purchase date |
| `last_order_date` | Latest purchase date |
| `Avg_order_value` | `total_order_value / total_orders`, rounded to 2 decimal places |

## 3. `gold_product`

**Grain:** One row per `product_id` and product category grouping.  
**Purpose:** Product sales and review-related summary metrics.

| Column | Meaning |
|---|---|
| `product_id` | Product identifier; expected unique in the resulting table |
| `product_category_name` | Product category |
| `total_units_sold` | Count of order-item records |
| `total_orders` | Distinct orders containing the product |
| `total_revenue` | Sum of item prices |
| `avg_product_price` | Average item price |
| `avg_freight_value` | Average freight value |
| `avg_review_score` | Average order review score joined to the product's order items |

**Interpretation note:** `avg_review_score` is based on the average review score per order joined to order items. It is not a direct product-only review score. The notebook uses inner joins for the review/product/order-item combination, so rows without a matching review can be excluded from this aggregation.

## 4. `gold_delivery`

**Grain:** One row per order (`order_id`).  
**Purpose:** Delivery-duration and delivery-performance analysis.

The table retains the columns from `workspace.silver.silver_orders`, adds the three derived duration fields below, and is later enriched with `customer_state` from `workspace.silver.silver_customer`.

| Column | Meaning |
|---|---|
| `delivery_days` | Difference in calendar days between purchase date and delivered-to-customer date; null if no delivered date |
| `estimated_delivery_days` | Difference in calendar days between purchase date and estimated delivery date |
| `delivery_delay_days` | Delivered date minus estimated delivery date, in days; positive = late, zero = on time, negative = early; null if either date is unavailable |
| `customer_state` | Customer state joined using `customer_id` |

**Inherited columns:** Order identifiers, customer identifier, order status, and order lifecycle timestamps are retained from the Silver orders table.

## 5. `gold_order_payments`

**Grain:** One row per `order_id`.  
**Purpose:** Order-level payment summary.

| Column | Meaning |
|---|---|
| `order_id` | Order identifier; table key |
| `total_payment_value` | Sum of payment values for the order, rounded to 2 decimal places |
| `total_payment_records` | Number of payment records for the order |
| `max_installments` | Maximum installment count recorded for the order |
| `payment_method_count` | Number of distinct payment types used for the order |

## 6. `gold_payment_methods`

**Grain:** One row per `order_id` + `payment_type`.  
**Purpose:** Payment summary broken down by payment method within each order.

| Column | Meaning |
|---|---|
| `order_id` | Order identifier |
| `payment_type` | Payment method/type |
| `total_payment_value` | Sum of payment values for this order and payment type, rounded to 2 decimal places |
| `payment_records` | Number of payment records for this order and payment type |
| `max_installments` | Maximum installment count for this order and payment type |

## Table grain and validation keys

| Gold table | Expected unique key |
|---|---|
| `gold_sales` | `order_id`, `order_item_id` |
| `gold_customers` | `customer_unique_id` |
| `gold_product` | `product_id` |
| `gold_delivery` | `order_id` |
| `gold_order_payments` | `order_id` |
| `gold_payment_methods` | `order_id`, `payment_type` |

## Notes

- Monetary columns are stored as numeric values and rounded to 2 decimal places where shown in the transformations.
- The project validated row counts, expected key uniqueness and total sales revenue between Databricks Gold and the BigQuery analytics copy.
- BigQuery publishing destination: `db_olist_analytics`.
