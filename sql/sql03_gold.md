# Gold: готовая витрина Q3 «Shipping Priority»
## Топ-10 заказов по выручке

CREATE SCHEMA IF NOT EXISTS gold.lakehouse;

CREATE TABLE gold.lakehouse.q3_shipping_priority AS
SELECT
  orderkey,
  SUM(extendedprice * (1 - discount)) AS revenue,
  orderdate,
  shippriority
FROM silver.lakehouse.order_lines
GROUP BY orderkey, orderdate, shippriority
ORDER BY revenue DESC
LIMIT 10;

-- Проверка (с явным приведением типа — иначе DBeaver показывает мусор)
SELECT
  orderkey,
  CAST(revenue AS decimal(20,2)) AS revenue,
  orderdate,
  shippriority
FROM gold.lakehouse.q3_shipping_priority
ORDER BY revenue DESC;
-- Ожидаемо: 10 строк, топ-1 = 406 181,01 (orderkey 2456423)