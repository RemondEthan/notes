Iceberg 支持通过 **Sort Order（排序顺序）** 定义 “排序键”，用于指定表中数据的物理排序方式。这种排序机制不同于分区（Partitioning），它是对**同一分区内的数据**进行有序组织，核心目的是优化查询性能（如加速范围查询、减少数据扫描量）和数据合并（Compaction）效率。

### 一、Iceberg 排序键（Sort Order）的核心特性

1. **定义方式**
    
    排序键通过 `SortOrder` 定义，可指定多个排序字段及排序方向（升序 `ASC` 或降序 `DESC`）。例如：
    
    - 按 `event_time DESC, user_id ASC` 排序，意味着数据先按 `event_time` 降序排列，相同 `event_time` 的记录再按 `user_id` 升序排列。
2. **与分区的区别**
    
    - **分区（Partitioning）**：将数据按分区字段拆分为不同的物理目录（如 `dt=20240101`），实现 “数据隔离”。
    - **排序（Sort Order）**：在**同一分区内部**对数据记录进行排序，属于 “分区内的数据组织方式”。
    
    例如，一个按 `dt` 分区的表，可在每个 `dt` 分区内按 `user_id` 排序，既实现了按日期的分区隔离，又让每个分区内的用户数据有序。
    

### 二、如何定义排序键

#### 1. 通过 SQL 创建表时指定（以 Spark 为例）

sql

```sql
CREATE TABLE iceberg_db.event_table (
  id INT,
  event_time TIMESTAMP,
  user_id STRING,
  content STRING
)
USING iceberg
PARTITIONED BY (date(event_time))  -- 按日期分区
TBLPROPERTIES (
  'write.sort-order' = 'event_time DESC, user_id ASC'  -- 定义排序键
);
```

#### 2. 通过 Iceberg API 定义

java

运行

```java
// 定义排序字段：event_time 降序，user_id 升序
SortOrder sortOrder = SortOrder.builder()
    .asc("user_id")          // 升序
    .desc("event_time")      // 降序
    .build();

// 创建表时关联排序键
Table table = CatalogUtil.createTable(
    catalog,
    TableIdentifier.of("iceberg_db", "event_table"),
    schema,
    partitionSpec,
    sortOrder,
    Map.of()
);
```

### 三、排序键的作用与优势

1. **加速查询过滤**
    
    排序后的数据文件会记录每个文件内排序字段的 `min/max` 统计信息（如 `event_time` 的最小值和最大值）。查询时，Iceberg 可利用这些统计信息进行**文件级剪枝**，跳过不满足条件的文件。
    
    例如，查询 `event_time > '2024-01-01 12:00:00'` 时，若某文件的 `max(event_time)` 小于该值，可直接跳过该文件，减少扫描量。
    
2. **优化数据合并（Compaction）**
    
    Iceberg 会定期合并小文件（Compaction），排序键可确保合并后的大文件按指定顺序组织，避免后续查询时的无序扫描。尤其对于频繁更新的表（如通过 `MERGE INTO` 操作），排序后的文件能减少更新时的随机 IO。
    
3. **提升等值查询和范围查询性能**
    
    对于按排序键的查询（如 `WHERE user_id = 'u123'` 或 `event_time BETWEEN ...`），排序后的数据可让查询引擎更快定位到目标记录（类似索引的效果，但属于物理数据布局优化）。
    

### 四、注意事项

1. **排序键的 overhead**
    
    写入数据时按排序键排序会增加一定的计算开销（需在写入前排序），适合**读多写少**的场景。对于高频写入的实时表，需权衡写入性能与查询性能。
    
2. **排序键的修改**
    
    Iceberg 支持通过 `ALTER TABLE` 修改排序键（属于表元数据演化），但修改后仅对新写入的数据生效，历史数据仍保持原有排序方式。若需全表重排序，需执行 `rewrite_data_files` 操作（如通过 Spark 调用 Iceberg 的重写 API）。
    
3. **与分区键的配合**
    
    通常建议排序键包含查询中频繁过滤的非分区字段。例如，按 `dt` 分区的表，可在分区内按 `user_id` 排序，兼顾分区剪枝和分区内的高效查询。
    

### 总结

Iceberg 支持通过 `Sort Order` 定义排序键，用于指定分区内数据的物理排序方式。它通过优化数据布局和统计信息利用，显著提升查询性能（尤其是过滤和范围查询），是 Iceberg 表在大规模数据场景下高效查询的重要优化手段。