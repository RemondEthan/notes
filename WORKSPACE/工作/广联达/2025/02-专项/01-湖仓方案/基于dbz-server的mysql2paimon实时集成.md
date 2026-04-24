

# 基于dbz-server的mysql2paimon实时集成

本项目是基于Debezium实时数据库变更捕获引擎和 Apache Paimon 流批一体湖仓系统的数据同步服务。其核心目标是：

- 实时捕获源端数据库（MySQL）的数据变更；
- 将 CDC 数据写入到 Paimon 表中；
- 支持表结构变更（DDL）同步；
- 支持分区表处理；
- 提供 Kafka 消息通知机制。

---

## 二、系统架构图



## 三、模块功能介绍

### 1. EngineServer - 引擎启动器（Spring Bean）

#### 功能：
- 使用 @PostConstruct 注解在 Spring 容器初始化后自动启动 Debezium 引擎。
- 加载配置文件，构建 Debezium 引擎。
- 启动线程运行 Debezium。

#### 核心方法：
```java
consumer = new PaimonChangeConsumer(config);
@PostConstruct
public void setup() {
    ...
    this.engine = DebeziumEngine.create(...)
            .using(props)
            .notifying(consumer)
            .build();
    executor.execute(() -> {
        this.engine.run();
    });
}
```

#### 流程：

引擎根据我们所提供的配置文件获取事件，并将事件推送给消费者，handlebatch用来接收事件，通过解析事件来判断是什么操作，进行后续步骤。


---

### 2.参数初始化

#### 功能：

- 初始化公共参数
- 初始化Source参数
- 初始化Sink参数

#### 核心方法：

**CommonConfiguration：**

```
public class CommonConfiguration {
    final Configuration config;
    final String internalDbHost;
    final String internalDbPort;
    final String internalDbName;
    final String internalDbUser;
    final String internalDbPassword;
    final String internalDbType;
    public CommonConfiguration(Configuration config) {
        this.config = config;
        this.internalDbHost = config.getString(INTERNAL_DB_HOST);
        this.internalDbPort = config.getString(INTERNAL_DB_PORT);
        this.internalDbName = config.getString(INTERNAL_DB_NAME);
        this.internalDbUser = config.getString(INTERNAL_DB_USER);
        this.internalDbPassword = config.getString(INTERNAL_DB_PASSWORD);
        this.internalDbType = config.getString(INTERNAL_DB_TYPE);
    }
 }
```

**SourceConfig：**

```
public class SourceConfig {
    final Configuration config;
    public static final String SourceConfigPrefix = "debezium.source.";
    public static final String KAKFA_PREFIX = "debezium.source.signal.kafka.";
    //redis相关
    public static final String REDIS_ADDRESS = "schema.history.internal.redis.address";
    public static final String REDIS_PASSWORD = "schema.history.internal.redis.password";
    //kafka相关
    public static final String CONNECTOR_CLIENT_CONFIG_OVERRIDE_POLICY = "connector.client.config.override.policy";
    public static final String TOPIC_NAME = "topic";
    public static final String BOOTSTRAP_SERVERS = "bootstrap.servers";
    //mysql相关
    public static final String MYSQL_PORT = "database.port";
    public static final String MYSQL_USER = "database.user";
    public static final String MYSQL_PASSWORD = "database.password";
    //是否按照分区排序
    public static final String SORT_PARTITIONS = "order.by.partition";
    //消息变更topic
    public static final String MESSAGE_CHANGE_TOPIC = "message.change.topic";
}
```

**SinkConfig：**

```
public class SinkConfig {
    final Configuration config;
    Map<String, DebeziumMappingConfig> tableMappings;
    public SinkConfig(Configuration config) {
        this.config = config;
        this.tableMappings = getTableMappings(config);
    }
    public Map<String, DebeziumMappingConfig> getTableMappings(Configuration config) {
        return mappings;
    }
}
```

#### 主要流程：

在CommonConfiguration中主要写一些公共参数，如我们使用swan的库的信息，在SourceConfig主要写一些与source相关的参数，比如源库的信息，在SinkConfig中主要写一些目标表相关的信息，其中主要读取mapping的信息。

### 3. `PaimonChangeConsumer` - CDC 事件消费者

#### 功能：
- 消费 Debezium 的 `ChangeEvent`。
- 解析事件类型：DATA、CREATE、DROP、ALTER、RENAME、DDL_OTHER。
- 调用 `PaimonSinkContext` 写入数据。
- 调用 `PaimonSinkContext`处理 DDL 事件并更新 Paimon 表结构。
- 支持 Kafka 分区消息通知。

#### 主要流程：
```java
@Override
public void handleBatch(List<ChangeEvent<Object, Object>> changeEventList, RecordCommitter<ChangeEvent<Object, Object>> committer) {
    for (ChangeEvent event : changeEventList) {
        // 解析事件类型
        ChangeEventRecord changeEventRecord = ChangeEventRecord.from(changeEvent, tableMappings);
        ChangeEventRecord record = parseRecord(changeEventRecord);
        switch (record.getEventType().getEventType()) {
            case "DATA":
                paimonSinkContext.writeRowData(...);
                break;
            case "CREATE":
                execCreate(...);
                break;
            case "ALTER":
                execAlter(...);
                break;
        }
    }

    flushAndCommit(); // 刷盘提交
}
```


---

#### 流程：

dbz server接收到时间消息后进行解析具体解析，具体解析代码通过进行解析json，然后转为record，在record中通过寻找ddl然后得知是什么操作，根据具体的操作类型，进行后续步骤。

### 4. PaimonSinkContext - Paimon 写入上下文管理器

#### 功能：
- 初始化 Paimon Catalog。
- 创建/删除/修改 Paimon 表。
- 写入数据行。
- 支持 Schema 变更（SchemaChange）。
- 处理时间字段精度问题（TIMESTAMP 默认精度不足时修正为 4，time类型转string）。

#### 示例：创建表逻辑
```java
@Override
public boolean createTable(ChangeEventRecord changeEventRecord) {
    Identifier identifier = getIdentifier(changeEventRecord);
    catalog.createDatabaseIfNotExists(identifier.getDatabaseName());
    
    Schema schema = SchemaUtils.buildSchema(...); // 构建 Schema
    catalog.createTable(identifier, schema, true); // 创建表
}
```

##### 多分区以及分桶机制：

多分区：

```
partitionList = Arrays.asList(debeziumMappingConfig.getSinkPartitionField().split(","));
Schema schema = SchemaUtils.buildSchema(fieldTypes, fieldComments, primaryList, table.comment(), partitionList, options);
```

多分桶：

```
if(debeziumMappingConfig.getSinkBuckets() != null && debeziumMappingConfig.getSinkDistributedField() != null ){
                    long bucketFiledValue = Long.parseLong(record.fields().get(debeziumMappingConfig.getSinkDistributedField()));
                    int bucketCount = Integer.parseInt(debeziumMappingConfig.getSinkBuckets()); //分桶数量从配置文件中获取
                    int bucket =  (int) (bucketFiledValue % bucketCount);
                    SinkRecord sinkRecord = write.writeAndReturn(genericRow,bucket);
                    if (sinkRecord != null) {
                        touchBucket(writtenBuckets, sinkRecord.partition(), sinkRecord.bucket());
                    }
   }
```

这里主要使用的分桶机制是根据官网分桶机制设置：

![image-20250720150155138](C:\Users\licc-k\AppData\Roaming\Typora\typora-user-images\image-20250720150155138.png)

#### 示例：Schema 变更逻辑

```java
@Override
public boolean alterTable(ChangeEventRecord changeEventRecord) {
    Identifier identifier = getIdentifier(changeEventRecord);
    List<SchemaChange> schemaChanges = extractSchemaChangesFromDDL(...);
    schemaManager.commitChanges(schemaChanges); // 应用 Schema 变更
}
```

通过changeEventRecord.getPayload().getString("ddl")，changeEventRecord中有ddl这样的key，通过正则匹配达成如何进行操作的效果

#### 示例：类型转换逻辑

```java
for (Map.Entry<String, DataType> entry : fieldTypes.entrySet()) {
    if (entry.getValue().is(DataTypeRoot.TIME_WITHOUT_TIME_ZONE)) {
        entry.setValue(new VarCharType(255));
    }
    if(entry.getValue().is(DataTypeRoot.TIMESTAMP_WITHOUT_TIME_ZONE)){
        int precision = ((TimestampType) entry.getValue()).getPrecision();
        if(precision < 4){
            entry.setValue(new TimestampType(4));
        }
    }
}
```




---

### 5. `PaimonChangeConsumer.flushAndCommit()` - 数据刷盘逻辑

#### 功能：
- 控制数据刷新频率。
- 批量刷盘。
- 发送 Kafka 分区更新消息。

```java
private void flushAndCommit() throws InterruptedException {
    if (System.currentTimeMillis() - lastFlushTime < flushInterval * 1000) return;

    paimonSinkContext.flushAllData(); // 刷盘所有数据

    pendingPartitionDataUpdateLists.forEach(this::sendDataUpdateMessage); // 发送 Kafka 消息
    globalCommitter.markBatchFinished(); // 提交 offset
}
```


---

### 6. `sendDataUpdateMessage()` - Kafka 消息发送

#### 功能：
- 将分区更新信息按表名和分区键聚合。
- 构造 JSON 消息发送至 Kafka Topic。

```java
private void sendDataUpdateMessage(List<PartitionDataUpdateAlarm> alarmList) {
    Map<String, Map<String, List<PartitionDataUpdateAlarm>>> grouped = new HashMap<>();
    for (var alarm : alarmList) {
        String key = alarm.getTableFullName();
        String partitionKey = alarm.getPartitionString();
        grouped.computeIfAbsent(key, k -> new HashMap<>())
               .computeIfAbsent(partitionKey, k -> new ArrayList<>())
               .add(alarm);
    }

    grouped.forEach((table, partitions) -> {
        partitions.forEach((partitionKey, alarms) -> {
            UpdateMessage msg = UpdateMessage.builder()
                    .tableName(table)
                    .partition(alarms.get(0).getPartitionValue())
                    .updateCount(alarms.size())
                    .build();

            ProducerRecord<String, String> record = new ProducerRecord<>("topic", json);
            producer.send(record);
        });
    });
}
```

###  7.后期优化

#### 发送快照信息时增加过滤条件

##### 功能：

现在我们遇到的场景是gde那边需要使用租户作为分区，但是现在分区过多，每一次来一批数据我们都需要往obs中io好多次，这样就会导致全量写入的时候效率过慢，目前解决方式使用修改增量快照的方式，我们首先查出该表中所有的分区数，然后使用了additional-conditions的方式进行过滤，对于每个分区的数据进行查询，最后再写入，这样就不会出现多个分区重复写入的情况。

##### 具体代码：

###### 查出所有分区：

```
    private List<String> getPartitionFieldValues(String tableName) {
        List<String> partitionValuesList = new ArrayList<>();

        DebeziumMappingConfig mappingConfig = tableMappings.get(tableName);
        if (mappingConfig == null || mappingConfig.getSinkPartitionField() == null) {
            log.warn("No partition field configured for table: {}", tableName);
            return partitionValuesList;
        }

        String partitionField = mappingConfig.getSinkPartitionField();
        String dbName = mappingConfig.getSourceDatabase();
        String dbHost = sourceConfig.getMysqlHost();
        String dbPort = sourceConfig.getMysqlPort();
        String dbUser = sourceConfig.getMysqlUser();
        String dbPassword = sourceConfig.getMysqlPassword();
        String url = String.format("jdbc:mysql://%s:%s/%s?useSSL=false&serverTimezone=UTC", dbHost, dbPort, dbName);
        // 处理多个字段
        String[] partitionFields = partitionField.split(",");
        String concatenatedField = Arrays.stream(partitionFields)
                .map(f -> "CAST(" + f.trim() + " AS CHAR)")
                .collect(Collectors.joining(" + '_' + "));

        String sql = "SELECT DISTINCT " + concatenatedField + " FROM " + mappingConfig.getSourceTable();

        try (
                Connection conn = DriverManager.getConnection(url, dbUser, dbPassword);
                PreparedStatement ps = conn.prepareStatement(sql);
                ResultSet rs = ps.executeQuery()
        ) {
            while (rs.next()) {
                String value = rs.getString(1);  // 获取拼接后的唯一字段
                partitionValuesList.add(value);
            }
        } catch (Exception e) {
            log.error("Error querying partition values for table {}: {}", tableName, e.getMessage());
        }

        return partitionValuesList;
    }
```

###### 根据分区拼接快照：

```
for (String partitionKey : partitionFieldValues) {
    String[] values = partitionKey.split("_");  // 根据拼接符拆分
    if (values.length != partitionFields.length) {
        log.warn("Partition key and field count mismatch: {}", partitionKey);
        continue;
    }

    Map<String, Object> signalMap = new HashMap<>();
    signalMap.put("type", "execute-snapshot");

    Map<String, Object> signalMapData = new HashMap<>();
    signalMapData.put("data-collections", Collections.singleton(tableName));
    signalMapData.put("type", "INCREMENTAL");

    List<Map<String, Object>> additionalConditions = new ArrayList<>();

    for (int i = 0; i < partitionFields.length; i++) {
        Map<String, Object> condition = new HashMap<>();
        condition.put("data-collection", tableName);
        condition.put("filter", partitionFields[i].trim() + "='" + values[i] + "'");
        additionalConditions.add(condition);
    }

    if (!additionalConditions.isEmpty()) {
        signalMapData.put("additional-conditions", additionalConditions);
    }

    signalMap.put("data", signalMapData);

    ProducerRecord<Object, Object> producerRecord = new ProducerRecord<>(topic, kafkaTopicPrefix, JSONObject.toJSONString(signalMap));
    Future<RecordMetadata> metadataFuture = producer.send(producerRecord);
    RecordMetadata recordMetadata = metadataFuture.get();
    log.info("produced incremental signal success for table:{} offset:{}", tableName, recordMetadata.offset());
}
```



###### dbz官网地址：https://debezium.io/documentation/reference/2.7/connectors/mysql.html

| Field                   | Default       | Value                                                        |
| :---------------------- | :------------ | :----------------------------------------------------------- |
| `type`                  | `incremental` | Specifies the type of snapshot that you want to run. Currently, you can request `incremental` or `blocking` snapshots. |
| `data-collections`      | *N/A*         | An array that contains regular expressions matching the fully-qualified names of the tables to include in the snapshot. For the MySQL connector, use the following format to specify the fully qualified name of a table: `database.table`. |
| `additional-conditions` | *N/A*         | An optional array that specifies a set of additional conditions that the connector evaluates to determine the subset of records to include in a snapshot. Each additional condition is an object that specifies the criteria for filtering the data that an ad hoc snapshot captures. You can set the following parameters for each additional condition:`data-collection`The fully-qualified name of the table that the filter applies to. You can apply different filters to each table.`filter`Specifies column values that must be present in a database record for the snapshot to include it, for example, `"color='blue'"`.  The values that you assign to the `filter` parameter are the same types of values that you might specify in the `WHERE` clause of `SELECT` statements when you set the `snapshot.select.statement.overrides` property for a blocking snapshot. |
| `surrogate-key`         | *N/A*         | An optional string that specifies the column name that the connector uses as the primary key of a table during the snapshot process. |

举例：

```
**INSERT** **INTO** db1.debezium_signal (**id**, **type**, **data**)  **values** ('ad-hoc-1',       'execute-snapshot',      '{"data-collections": ["db1.table1", "db1.table2"],     "type":"incremental",     "additional-conditions":[{"data-collection": "db1.table1" ,"filter":"color=\'blue\'"}]}'); 
```

