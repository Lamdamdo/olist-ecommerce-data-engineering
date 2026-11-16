# Data Validation

This document records the validation checks performed for the Olist data engineering project after publishing the Gold tables from Databricks to BigQuery.

## 1. BigQuery destination

- Google Cloud project: `databricksproject-510805`
- Analytics dataset: `db_olist_analytics`
- Databricks source schema: `workspace.gold`

## 2. Row-count validation

The BigQuery table row counts were compared with the expected row counts of the corresponding Databricks Gold tables.

| Table | Expected rows | Result |
|---|---:|---|
| `gold_sales` | 112,650 | PASS |
| `gold_customers` | 96,096 | PASS |
| `gold_product` | 32,789 | PASS |
| `gold_delivery` | 99,441 | PASS |
| `gold_order_payments` | 99,440 | PASS |
| `gold_payment_methods` | 101,686 | PASS |

**Result:** All six BigQuery table row counts matched the expected counts.

## 3. Key uniqueness validation

The expected unique key for each Gold table was checked for duplicate combinations.

| Table | Key checked | Result |
|---|---|---|
| `gold_sales` | (`order_id`, `order_item_id`) | PASS |
| `gold_customers` | `customer_unique_id` | PASS |
| `gold_product` | `product_id` | PASS |
| `gold_delivery` | `order_id` | PASS |
| `gold_order_payments` | `order_id` | PASS |
| `gold_payment_methods` | (`order_id`, `payment_type`) | PASS |

**Result:** All six uniqueness checks passed.

## 4. Aggregate comparison

A metric-level comparison was performed for total sales revenue:

- Databricks source: `workspace.gold.gold_sales`
- BigQuery destination: `databricksproject-510805.db_olist_analytics.gold_sales`
- Metric: `ROUND(SUM(price), 2)`

**Result:** The total sales revenue returned the same value in Databricks and BigQuery.

## 5. Overall result

| Validation | Outcome |
|---|---|
| BigQuery row counts | 6/6 PASS |
| Gold key uniqueness | 6/6 PASS |
| Total sales revenue comparison | MATCH |

The checks above provide evidence that the six Gold tables were published to BigQuery with matching row counts, expected key uniqueness, and a matching total sales revenue aggregate.

## 6. Scope and limitations

These are the validation checks actually performed for this project. They do not represent a complete column-by-column comparison or a full reconciliation of every metric and record. Additional checks can be added if the project is extended.
