# 1.测试数据

| 表名       | 描述  | 数据量 |     |
| -------- | --- | --- | --- |
| lineitem |     |     |     |
| customer |     |     |     |
| orders   |     |     |     |
| supplier |     |     |     |
| nation   |     |     |     |
| region   |     |     |     |
# 2. 测试SQL以及特性比较点
```SQL
-- SQL1:硬计算能力比较
SELECT l_returnflag, l_linestatus, sum(l_quantity) as sum_qty,
sum(l_extendedprice) as sum_base_price,
sum(l_extendedprice * (1 - l_discount)) as sum_disc_price,
count(*) as count_order
FROM lineitem
WHERE l_shipdate <= DATE '1998-09-02'
GROUP BY l_returnflag, l_linestatus
ORDER BY l_returnflag, l_linestatus;
```
``` SQL
-- SQL2:多表连接能力，以及大小表join时处理能力比较
SELECT n_name, sum(l_extendedprice * (1 - l_discount)) as revenue
FROM customer, orders, lineitem, supplier, nation, region
WHERE c_custkey = o_custkey
AND l_orderkey = o_orderkey
AND l_suppkey = s_suppkey
AND c_nationkey = s_nationkey
AND s_nationkey = n_nationkey
AND n_regionkey = r_regionkey
AND r_name = 'ASIA' -- 可以改成其他地区
AND o_orderdate >= DATE '1994-01-01'
AND o_orderdate < DATE '1995-01-01'
GROUP BY n_name
ORDER BY revenue DESC;
```
``` SQL
-- SQL3:相比SQL1略简单，比较IO与数据查询优化能力
SELECT sum(l_extendedprice * l_discount) as revenue
FROM lineitem
WHERE l_shipdate >= DATE '1994-01-01'
AND l_shipdate < DATE '1995-01-01'
AND l_discount BETWEEN 0.05 AND 0.07
AND l_quantity < 24;
```
``` SQL
-- SQl4: 复杂查询能力比较
SELECT
nation,
o_year,
SUM(amount) AS sum_profit
FROM (
SELECT
n_name AS nation,
EXTRACT(YEAR FROM o_orderdate) AS o_year,
l_extendedprice * (1 - l_discount) - ps_supplycost * l_quantity AS amount
FROM part, supplier, lineitem, partsupp, orders, nation
WHERE
s_suppkey = l_suppkey
AND ps_suppkey = l_suppkey
AND ps_partkey = l_partkey
AND p_partkey = l_partkey
AND o_orderkey = l_orderkey
AND s_nationkey = n_nationkey
AND p_name LIKE '%green%'
) AS profit
GROUP BY nation, o_year
ORDER BY nation, o_year DESC;
```
``` SQL
-- 复杂查询，涉及到子查询的优化能力比较
SELECT
s_name,
COUNT(*) AS numwait
FROM supplier, lineitem l1, orders, nation
WHERE
s_suppkey = l1.l_suppkey
AND o_orderkey = l1.l_orderkey
AND o_orderstatus = 'F'
AND l1.l_receiptdate > l1.l_commitdate
AND EXISTS (
SELECT *
FROM lineitem l2
WHERE
l2.l_orderkey = l1.l_orderkey
AND l2.l_suppkey <> l1.l_suppkey
)
AND NOT EXISTS (
SELECT *
FROM lineitem l3
WHERE
l3.l_orderkey = l1.l_orderkey
AND l3.l_suppkey <> l1.l_suppkey
AND l3.l_receiptdate > l3.l_commitdate
)
AND s_nationkey = n_nationkey
AND n_name = 'SAUDI ARABIA'
GROUP BY s_name
ORDER BY numwait DESC, s_name
LIMIT 100;
```
# 3.测试资源
```yaml
# DUCKDB
resources:
  limits:
    cpu: '4'
    memory: 2Gi
  requests:
    cpu: 100m
    memory: 1Gi
```
```yaml
# Datafusion
resources:
  limits:
    cpu: '4'
    memory: 2Gi
  requests:
    cpu: 100m
    memory: 1Gi
```
```yaml
# SR4.0 FE
resources:
  limits:
    cpu: '1'
    memory: 2Gi
  requests:
    cpu: 100m
    memory: 1Gi
# SR4.0 BE
resources:
  limits:
    cpu: '3'
    memory: 2Gi
  requests:
    cpu: 100m
    memory: 1Gi
```
