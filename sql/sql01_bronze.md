# Bronze: сырые данные TPC-H Q3
## Запрос Q3 использует три таблицы: customer, orders, lineitem

CREATE SCHEMA IF NOT EXISTS bronze.lakehouse;

-- customer: нужны только custkey и mktsegment (фильтр BUILDING)
CREATE TABLE bronze.lakehouse.tpch_customer AS
SELECT custkey, mktsegment
FROM tpch.sf1.customer;

-- orders: orderkey, custkey, orderdate (фильтр по дате), shippriority (вывод)
CREATE TABLE bronze.lakehouse.tpch_orders AS
SELECT orderkey, custkey, orderdate, shippriority
FROM tpch.sf1.orders;

-- lineitem: orderkey, extendedprice, discount (агрегат), shipdate (фильтр)
CREATE TABLE bronze.lakehouse.tpch_lineitem AS
SELECT orderkey, extendedprice, discount, shipdate
FROM tpch.sf1.lineitem;

-- Проверка количества строк
SELECT 'customer' AS tbl, count(*) FROM bronze.lakehouse.tpch_customer
UNION ALL
SELECT 'orders', count(*) FROM bronze.lakehouse.tpch_orders
UNION ALL
SELECT 'lineitem', count(*) FROM bronze.lakehouse.tpch_lineitem;
-- Ожидаемо: 150000 / 1500000 / 6001215