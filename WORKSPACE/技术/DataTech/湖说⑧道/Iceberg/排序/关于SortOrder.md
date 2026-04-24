在使用纯 Iceberg API 写入时，SortOrder 的生效机制与 Spark 环境有本质区别：**Iceberg API 本身不执行物理排序，而是仅记录排序元数据** 。

### DataFile 创建时的 SortOrder 记录

当使用 Iceberg API 创建 DataFile 时，排序信息通过 `DataFiles.Builder` 的 `withSortOrder()` 方法记录到文件元数据中 DataFiles.java:314-319 。

在 `DataFiles.Builder.build()` 方法中，`sortOrderId` 被传递给 `GenericDataFile` 构造函数 DataFiles.java:335-352 。

### 元数据写入过程

#### 1. DataFile 元数据结构

`GenericDataFile` 继承自 `BaseFile`，在构造时接收 `sortOrderId` 参数 GenericDataFile.java:40-73 。这个 ID 存储在 `BaseFile` 的 `sortOrderId` 字段中 BaseFile.java:82 。

#### 2. ContentFile 序列化

当 DataFile 被序列化到 manifest 文件时，`ContentFileParser.toJson()` 方法会将 `sortOrderId` 写入 JSON ContentFileParser.java:112-114 。

#### 3. 表元数据更新

表的 SortOrder 信息通过 `TableMetadataParser.toJson()` 写入表元数据文件 TableMetadataParser.java:207-213 。

### 纯 API 写入的限制

**关键点：Iceberg API 本身不提供数据排序功能** 。当您使用纯 Iceberg API 写入时：

1. **无物理排序**：API 不会对数据进行实际排序，只是记录排序规范
2. **用户责任**：需要用户在写入前自行确保数据按照 SortOrder 规范排序
3. **元数据一致性**：即使数据未排序，`sortOrderId` 仍会被记录到文件元数据中

### 实际使用场景

在测试代码中可以看到，创建表时可以指定 SortOrder TestTables.java:136-144 ，但这只是设置了表级别的排序规范，不会影响实际数据写入的排序。

### SortOrder 元数据更新

当需要更新表的 SortOrder 时，会创建 `MetadataUpdate.AddSortOrder` 或 `MetadataUpdate.SetDefaultSortOrder` 更新 TestMetadataUpdateParser.java:296-332 ，这些更新会被序列化并应用到表元数据中。

## Notes

与 Spark 环境不同，纯 Iceberg API 是声明式的排序系统。它提供了排序规范的定义和元数据管理，但不执行实际的数据排序。如果需要物理排序，必须在写入 Iceberg 之前使用其他工具（如 Spark、Flink 等计算引擎）对数据进行预排序，然后使用 Iceberg API 写入已排序的数据并正确设置 `sortOrderId`。