# Бенчмарк сжатия GZIP vs ZSTD
## Меняем кодек в catalog/bronze.properties + docker restart trino,
## затем пересоздаём Bronze-таблицы (см. 01_bronze.sql) и замеряем.

-- 1. Узнать текущий кодек
SELECT * FROM bronze.lakehouse."tpch_lineitem$properties";

-- 2. Размер файлов
SELECT SUM(file_size_in_bytes) / 1024.0 / 1024.0 AS mb
FROM bronze.lakehouse."tpch_lineitem$files";

SELECT SUM(file_size_in_bytes) / 1024.0 / 1024.0 AS mb
FROM bronze.lakehouse."tpch_orders$files";

-- 3. Время Q3 по Bronze (замер вручную по секундомеру DBeaver)
SELECT
  l.orderkey,
  SUM(l.extendedprice * (1 - l.discount)) AS revenue,
  o.orderdate,
  o.shippriority
FROM bronze.lakehouse.tpch_customer AS c
JOIN bronze.lakehouse.tpch_orders   AS o ON c.custkey = o.custkey
JOIN bronze.lakehouse.tpch_lineitem AS l ON l.orderkey = o.orderkey
WHERE c.mktsegment = 'BUILDING'
  AND o.orderdate < DATE '1995-03-15'
  AND l.shipdate > DATE '1995-03-15'
GROUP BY l.orderkey, o.orderdate, o.shippriority
ORDER BY revenue DESC
LIMIT 10;

-- Результаты:
-- GZIP: lineitem 38.2 МБ, orders 7.3 МБ, Q3 = 5.227 сек
-- ZSTD: lineitem 38.6 МБ, orders 6.7 МБ, Q3 = 5.066 сек