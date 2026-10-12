# Silver: очищенный факт строк заказов для Q3
## Фильтры: customer.mktsegment='BUILDING', orders.orderdate < 1995-03-15, lineitem.shipdate > 1995-03-15

CREATE SCHEMA IF NOT EXISTS silver.lakehouse;

CREATE TABLE silver.lakehouse.order_lines AS
SELECT
  l.orderkey       AS orderkey,
  o.orderdate      AS orderdate,
  o.shippriority   AS shippriority,
  l.extendedprice  AS extendedprice,
  l.discount       AS discount,
  l.shipdate       AS shipdate
FROM bronze.lakehouse.tpch_customer c
JOIN bronze.lakehouse.tpch_orders   o ON c.custkey = o.custkey
JOIN bronze.lakehouse.tpch_lineitem l ON l.orderkey = o.orderkey
WHERE c.mktsegment = 'BUILDING'
  AND o.orderdate < DATE '1995-03-15'
  AND l.shipdate > DATE '1995-03-15';

SELECT count(*) FROM silver.lakehouse.order_lines;
-- Ожидаемо: 30519