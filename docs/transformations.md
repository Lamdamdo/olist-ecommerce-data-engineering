# Data Transformations

This document summarises the transformations implemented in the Olist Databricks notebook. It follows the project flow and records the logic used, including practical exploratory checks.

## 1. Source and Bronze layer

### Source
The raw Olist tables are accessed from the BigQuery source dataset through the Databricks foreign catalog:

- Catalog: `db_project_m_catalog`
- Schema: `db_olist_dataset`

### Bronze
The notebook creates the `workspace.bronze`, `workspace.silver` and `workspace.gold` schemas, then persists the source tables into Bronze.

Bronze tables:
- `orders`
- `customers`
- `products`
- `geolocation`
- `order_items`
- `order_payments`
- `order_reviews`
- `sellers`
- `product_category_name_translation`

The product-category translation table is loaded with SQL. The source header row is excluded, and the two source fields are named `product_category_name` and `product_category_name_english`.

**Bronze approach:** preserve the source data in Databricks tables as the starting point for downstream cleaning and transformation.

## 2. Silver layer

Silver transformations clean and standardise the source data before it is used in Gold aggregations.

### Customers — `silver_customer`
- Standardise `customer_state` and `customer_city` using title case (`initcap`).
- Remove duplicate records using `customer_id` as the deduplication key.
- Persist as `workspace.silver.silver_customer`.

### Geolocation — `silver_geo`
- Convert `geolocation_city` to lowercase for standardisation and inspection.
- Apply a city mapping dictionary to consolidate spelling, spacing, accent and punctuation variants for major cities.
- Persist the resulting table as `workspace.silver.silver_geo`.

### Order items — `silver_order_items`
- Inspect the schema and check for records with negative `price`.
- Persist the order-item data as `workspace.silver.silver_order_items`.

**Note:** the notebook includes an inspection of negative-price rows; it does not apply a filter to remove them in the persisted Silver table.

### Order payments — `silver_order_payments`
- Standardise `payment_type` labels using a mapping:
  - `credit_card` → `Credit Card`
  - `boleto` → `Bank Payment`
  - `voucher` → `Voucher`
  - `debit_card` → `Debit Card`
  - `not_defined` → `Unknown`
- Persist as `workspace.silver.silver_order_payments`.

### Order reviews — `silver_order_reviews`
- Inspect the review data and check for null `order_id` values.
- Persist the review data as `workspace.silver.silver_order_reviews`.

**Note:** the notebook checks for null order IDs but does not remove review rows as part of this step. Review comments are not translated; Portuguese comments are retained.

### Orders — `silver_orders`
- Standardise `order_status` using title case.
- Derive `order_cancellation_stage`:
  - `Not Canceled` when the order status is not `Canceled`.
  - `Before Approval` when the approval timestamp is null.
  - `After Approval` when the carrier hand-off timestamp is null.
  - `During Delivery` when the customer-delivery timestamp is null.
  - `Delivered` as the final fallback for a cancelled order with the preceding timestamps populated.
- Persist as `workspace.silver.silver_orders`.

### Products — `silver_product`
- Join products to the category translation table using a **left join** on `product_category_name`.
- Replace the original Portuguese category field with the English category field.
- Cast product name length, description length, photo count, weight and dimensions to `double`.
- Fill null values with `0` for:
  - `product_description_lenght`
  - `product_name_lenght`
  - `product_photos_qty`
- Fill missing category values with `No Category`.
- Standardise category labels: map `Baby` to `Baby Care`, replace underscores with spaces, and apply title case.
- Persist as `workspace.silver.silver_product`.

Physical weight and dimension fields are cast to numeric types; the notebook does not fill their nulls with zero.

### Sellers — `silver_sellers`
- Inspect seller IDs and distinct seller cities.
- Apply a city mapping dictionary to consolidate common spelling and formatting variants.
- Standardise `seller_city` using title case.
- Replace seller-city values containing `/` or `\` with `Multiple Cities`.
- Persist as `workspace.silver.silver_sellers`.

## 3. Gold layer

Gold tables combine Silver data and calculate business-level metrics for analysis.

### `gold_sales`
**Grain:** one row per order item.
- Start with Silver order items.
- Left join orders on `order_id`.
- Left join products on `product_id`.
- Left join sellers on `seller_id`.
- Select order, customer, product, seller, status, timestamp, category, price and freight fields.
- Derive `item_total_value` as `price + freight_value`, rounded to 2 decimal places.
- Derive `order_purchase_date` from `order_purchase_timestamp`.
- Derive `order_purchase_month` from the purchase date.
- Persist to `workspace.gold.gold_sales`.
- Enrich the table with `customer_unique_id` by left joining the Silver customer table on `customer_id`, then overwrite the Gold table with the updated schema.

### `gold_customers`
**Grain:** one row per `customer_unique_id`.
- Group the sales data by `customer_unique_id`.
- Calculate distinct order count, item count, total item spend, total freight, total order value, cancelled-order count, first purchase date and last purchase date.
- Calculate average order value as `total_order_value / total_orders`, rounded to 2 decimal places.
- Aggregate customer city, state and ZIP prefix from the Silver customer data.
- Left join customer information to the customer metrics.
- Persist to `workspace.gold.gold_customers`.

### `gold_product`
**Grain:** one row per `product_id`.
- Calculate average review score per order from Silver reviews.
- Join the order-level review average to Silver order items by `order_id`.
- Join the result to Silver products by `product_id`.
- Group by product ID and category.
- Calculate units sold, distinct orders, total revenue, average product price, average freight value and average review score.
- Persist to `workspace.gold.gold_product`.

**Interpretation:** `avg_review_score` is an order-level review score associated with products appearing in those orders; it is not a direct product-specific review score. The joins in this aggregation are inner joins, so rows without matching review/order-item/product records are not included in this product metric calculation.

### `gold_delivery`
**Grain:** one row per order.
- Start from Silver orders.
- Derive `delivery_days` as delivered-to-customer date minus purchase date, in calendar days.
- Derive `estimated_delivery_days` as estimated delivery date minus purchase date, in calendar days.
- Derive `delivery_delay_days` as delivered-to-customer date minus estimated delivery date, in calendar days.
- The delay calculation is populated only when both delivered and estimated delivery timestamps are present.
- Enrich the table with `customer_state` by left joining a distinct customer ID/state lookup from Silver customers.
- Persist the updated table to `workspace.gold.gold_delivery`.

**Interpretation:** positive `delivery_delay_days` means late; zero means on time; negative means delivered early.

### `gold_payment_methods`
**Grain:** one row per `order_id` + `payment_type`.
- Group payment records by order ID and payment type.
- Calculate total payment value, payment-record count and maximum installments.
- Persist to `workspace.gold.gold_payment_methods`.

### `gold_order_payments`
**Grain:** one row per `order_id`.
- Group payment records by order ID.
- Calculate total payment value, total payment-record count, maximum installments and distinct payment-method count.
- Persist to `workspace.gold.gold_order_payments`.

## 4. BigQuery publishing

The six Databricks Gold tables are published to the separate BigQuery analytics dataset:

`databricksproject-510805.db_olist_analytics`

The project uses the BigQuery Python client:
1. Retrieve the service-account JSON from Databricks Secrets.
2. Create Google service-account credentials and a BigQuery client.
3. Read each Gold table as a Spark DataFrame.
4. Convert the DataFrame to Pandas with `toPandas()`.
5. Load the Pandas DataFrame to the matching BigQuery table using `load_table_from_dataframe`.
6. Use `WRITE_TRUNCATE` to replace the destination table on each publishing run.

The Spark BigQuery connector was considered as an alternative but was not used in this project.

## 5. Validation performed

The project performed these checks after Gold publishing:

- BigQuery row counts compared with expected Gold row counts for all six tables.
- Uniqueness checks on the expected table keys:
  - `gold_sales`: (`order_id`, `order_item_id`)
  - `gold_customers`: `customer_unique_id`
  - `gold_product`: `product_id`
  - `gold_delivery`: `order_id`
  - `gold_order_payments`: `order_id`
  - `gold_payment_methods`: (`order_id`, `payment_type`)
- Compared total sales revenue between Databricks Gold and BigQuery `gold_sales`; the values matched.

## 6. Implementation notes

- The notebook uses exploratory `show()`, `display()`, schema and duplicate checks during development. These are part of the practical workflow.
- Several writes use overwrite mode because the project builds and refreshes its curated tables in a controlled notebook workflow.
- The customer cancellation metric should be reviewed if the notebook is rerun: the Silver order status is converted to title case (`Canceled`), while the customer aggregation condition in the original notebook compares against lowercase `canceled`. Spark string comparison is case-sensitive by default, so that condition may not count cancelled orders as intended. This note records a potential logic issue rather than silently changing the original transformation.
