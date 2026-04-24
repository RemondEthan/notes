#### 01.核心问题
- Iceberg数据在生产应用中，会产生大量小的增量文件与 delete 文件，compaction 的目标是 **合并小文件、降低文件/manifest 数量、把 delete 应用并减少读开销，同时控制 snapshot/元数据膨胀与并发安全**，从而提高查询/扫描效率与稳定性。

#### 02.问题关键点
- **小文件问题**：大量小 Parquet/ORC 文件会造成大量 I/O、打开文件开销、manifest/scan-range 增多    
- **数据重写（Rewrite Data Files）**：把多个小文件合成目标较大的文件（target file size），可按 partition 进行。
- **Manifest compaction（Rewrite Manifests）**：把许多 manifest 合并减少 manifest-list 的数量。
- **Delete 文件**：
    - **Position deletes**：指明要在某 data file 的具体位置删除行，通常在写入时与数据文件相配合，BE 执行时按位置过滤，读取性能较好。
    - **Equality deletes**：按列条件去删除，会导致 planner 生成 anti-joins 等更复杂计划。
- **Snapshot 管理**：合并/重写会生成新快照，需要做过期删除（expireSnapshots + removeOrphanFiles）来回收空间。

#### 03.优化策略
- **写入端优化（源头）**
    - CDC 写入尽量聚合 micro-batches：把短期的小批次合并为更“大”的写入（例如在写入端做缓冲 1s/1000 rows），减少写文件频次。
    - 控制小文件阈值（producer侧 target file size）尽可能和 compaction 的目标一致。
- **定期 Minor Compaction（频繁、轻量）**
    - 频率：短周期（分钟级到小时级），目标是把最近产生的小文件合并成中等文件（例如 64–256 MB），减少查询开销且不会触发大量 snapshot 清理成本。
    - 作用：降低 manifest/file 数量、减少小文件但保持写入延迟短。
- **周期 Major Compaction（稀疏、重写）**
    - 频率：日级或夜间（如每天或每周），目标是把分区中的文件合并到最终目标大小（例如 512 MB）。
    - 同时处理 delete 文件（应用 position deletes into data files，或者 rewrite equality deletes 的影响）。
    - 伴随 snapshot 回收（expireSnapshots、removeOrphanFiles）。
- **按 Partition / Bucket / SortKey 策略分层合并**
    - 按 partition 做局部合并（避免跨大量 partition 合并导致 shuffle 过大）。
    - 若表有 sortKey / clustering（例如 Iceberg 的 sort order 或者自定义 bucketing），在合并时保持同样的排序/桶策略，提升后续扫描效率（减少读时 merge cost）。
- **Manifest / Metadata 合并**
    - 在数据文件合并后，做 manifest 合并（rewriteManifests），减少 planning 时需要读取的 manifest 数量。
- **删除与过期**
    - 合并完成并验证后，调用 `expireSnapshots` + `removeOrphanFiles` 来回收空间。
    - 注意：先确保读者不需要旧 snapshot（有读一致性要求时谨慎）。
> 以下示例以 Spark + Iceberg Actions API（Java/Scala）与 SQL `CALL` 为主。不同 Iceberg 版本接口略有差异，请以你版本的 API 为准。
#### 03.实现方案
```sql
-- 对于 Spark Catalog（示例）：
CALL my_catalog.system.rewrite_data_files(
  table => 'db.my_table',
  options => map('target-file-size-bytes','536870912', 'split-open-delete-files','true'),
  purge => false
);

```
```java
import org.apache.iceberg.Table;
import org.apache.iceberg.actions.Actions;

Table table = ...; // load via Catalog
long targetSize = 512L * 1024 * 1024; // 512MB

Actions.forTable(table)
    .rewriteDataFiles()
    .option("target-file-size-bytes", Long.toString(targetSize))
    // 可选：按 partition 字段分组，或指定 max-input-files
    .option("split-open-delete-files", "true")
    .execute();

```
```java
// 只合并manifest
Actions.forTable(table)
    .rewriteManifests()
    .execute();

```
#### 04.最佳实践
- **目标文件大小（target-file-size-bytes）**
    
    - OLAP 扫描为主：512 MB 到 1 GB（但受 cloud object store round-trip 与 split 大小影响，512MB 常规）。
        
    - 小表/快速迭代：128–256 MB。
        
- **小文件阈值（触发规则）**
    
    - 如果 partition 中文件数 > 100 或小文件占比 > 50% 且小文件平均 < 64MB，则触发 minor compaction。
        
- **并行度 / shuffle**
    
    - 控制 rewrite job 的并行度，避免一次合并触发集群资源耗尽。分区级并行较好。
        
- **delete 文件策略**
    
    - 如果有大量 pos deletes：在 major compaction 时合并进 data files以减少 runtime delete apply。
        
    - eq deletes：评估是否在写入端尽可能生成 pos deletes 或在 compaction 时做 predicate-based rewrite。

---
# 源码剖析
`rewrite_data_files` 在 Spark 上映射到 Iceberg 的 `RewriteDataFiles` action（SparkActions 的实现），总体流程是：
>扫描表元数据 → 选出要重写的数据文件并分组 → 启动并行 Spark 任务（每个任务读一组文件并把 records 写入新的目标文件）→ 为每个“写出组”产生新数据文件和对应 manifest → 尝试把这些改动以（一个或多个）原子 commit 写回 Iceberg 元数据（快照）→ （可选）并行/分批提交以降低冲突风险 → 完成后垃圾回收或保留旧 snapshot，等待你手动 expire。
## 阶段 0 — 入口与参数解析（Procedure → Action）

**做什么**：Spark SQL layer 解析 `CALL <catalog>.system.rewrite_data_files(table => 'db.tbl', where => '...', options => map(...))`，并把参数转换为对 `SparkActions.get(spark).rewriteDataFiles(table)` 的调用与 option 设置。  
**源码点**：Spark procedures 页面和 `SparkActions` / `RewriteDataFilesSparkAction` 的入口。Spark 的 `CALL` 到 `system` 的映射在 Iceberg 的 spark-procedures helper 中实现。[iceberg.apache.org+1](https://iceberg.apache.org/docs/latest/spark-procedures/?utm_source=chatgpt.com)  
**注意**：`where` 可以把筛选条件下推到文件选择阶段（避免扫描全表）。
```java
/**  
 * A procedure that rewrites datafiles in a table. * * @see org.apache.iceberg.spark.actions.SparkActions#rewriteDataFiles(Table)  
 */
// RewriteDataFilesProcedure

public Iterator<Scan> call(InternalRow args) {  
  ProcedureInput input = new ProcedureInput(spark(), tableCatalog(), PARAMETERS, args);  
  Identifier tableIdent = input.ident(TABLE_PARAM);  
  String strategy = input.asString(STRATEGY_PARAM, null);  
  String sortOrderString = input.asString(SORT_ORDER_PARAM, null);  
  Map<String, String> options = input.asStringMap(OPTIONS_PARAM, ImmutableMap.of());  
  String where = input.asString(WHERE_PARAM, null);  
  
  return modifyIcebergTable(  
      tableIdent,  
      table -> {  
        RewriteDataFiles action = actions().rewriteDataFiles(table).options(options);  
  
        if (strategy != null || sortOrderString != null) {  
          action = checkAndApplyStrategy(action, strategy, sortOrderString, table.schema());  
        }  
		// 如果设置了SortOrder或者ZOred，这里会设置生效
        action = checkAndApplyFilter(action, where, tableIdent);  
		// 构建完SparkAction后开始执行 RewriteDataFilesSparkAction
        RewriteDataFiles.Result result = action.execute();  
  
        return asScanIterator(OUTPUT_TYPE, toOutputRows(result));  
      });  
}
// BaseProcedure modifyIcebergTable-> execute
private <T> T execute(  
    Identifier ident, boolean refreshSparkCache, Function<org.apache.iceberg.Table, T> func) {  
  SparkTable sparkTable = loadSparkTable(ident);  
  org.apache.iceberg.Table icebergTable = sparkTable.table();  
  
  T result = func.apply(icebergTable);  
  
  if (refreshSparkCache) {  
    refreshSparkCache(ident, sparkTable);  
  }  
  
  return result;
```
## 阶段 1 — 快照与 manifest 扫描（计划阶段）

**做什么**：从表的当前 snapshot（或指定 snapshot）读取 manifest lists → 展开成所有 `DataFile`（和 delete-file 信息），把候选文件集合化。此阶段决定“哪些 file 是候选被重写的”。`where` 与 options（比如 `older-than`, `min-input-files`）在此阶段应用过滤。  
**源码点**：`BaseRewriteDataFilesAction` 会收集 `FileScanTask` 列表；Spark 实现调用 Table API 的 `currentSnapshot()`、manifest list parser。  
**元数据**：读的是表的 manifest JSON / metadata，**不会修改表元数据**，只是构建计划。  
**诊断点**：如果表 metadata 泄露或缺失（manifest file missing），你会在这里看到 `ValidationException` 或 `FileNotFoundException`。日志级别开 DEBUG 可查看解析出的 file 列表。[iceberg.apache.org](https://iceberg.apache.org/javadoc/0.11.0/index.html?org%2Fapache%2Ficeberg%2Factions%2FBaseRewriteDataFilesAction.html=&utm_source=chatgpt.com)

```java
// RewriteDataFilesSparkAction.java
public RewriteDataFiles.Result execute() {  
  if (table.currentSnapshot() == null) {  
    return EMPTY_RESULT;  
  }  
  
  long startingSnapshotId = table.currentSnapshot().snapshotId();  
  // 这里主要初始化SparkShufflingDataRewritePlanner
  init(startingSnapshotId);  
  // 这里准备去扫描iceberg table了，还没有实际的动作，后边展开plan逻辑
  FileRewritePlan<FileGroupInfo, FileScanTask, DataFile, RewriteFileGroup> plan = planner.plan(); 
  
  if (plan.totalGroupCount() == 0) {  
    LOG.info("Nothing found to rewrite in {}", table.name());  
    return EMPTY_RESULT;  
  }  
  
  Builder resultBuilder =  
      partialProgressEnabled  
          ? doExecuteWithPartialProgress(plan, commitManager(startingSnapshotId))  
          : doExecute(plan, commitManager(startingSnapshotId));  
  ImmutableRewriteDataFiles.Result result = resultBuilder.build();  
  
  if (removeDanglingDeletes) {  
    RemoveDanglingDeletesSparkAction action =  
        new RemoveDanglingDeletesSparkAction(spark(), table);  
    int removedDeleteFiles = Iterables.size(action.execute().removedDeleteFiles());  
    return result.withRemovedDeleteFilesCount(  
        result.removedDeleteFilesCount() + removedDeleteFiles);  
  }  
  
  return result;  
}
```
```java
// 现在重点分析planner.plan()的执行
// BinPackRewriteFilePlanner.java 所有的Sort＊*Planner都继承BinPack
public FileRewritePlan<FileGroupInfo, FileScanTask, DataFile, RewriteFileGroup> plan() {  
    /**  
     * TODO 这里的两层List是代表什么结构  
     */  
  StructLikeMap<List<List<FileScanTask>>> plan = planFileGroups();  
  RewriteExecutionContext ctx = new RewriteExecutionContext();  
  List<RewriteFileGroup> selectedFileGroups = Lists.newArrayList();  
  AtomicInteger fileCountRunner = new AtomicInteger();  
  
  plan.entrySet().stream()  
      .filter(e -> !e.getValue().isEmpty())  
      .forEach(  
          entry -> {  
            StructLike partition = entry.getKey();  
            entry  
                .getValue()  
                .forEach(  
                    fileScanTasks -> {  
                      long inputSize = inputSize(fileScanTasks);  
                      if (maxFilesToRewrite == null) {  
                        selectedFileGroups.add(  
                            newRewriteGroup(  
                                ctx,  
                                partition,  
                                fileScanTasks,  
                                inputSplitSize(inputSize),  
                                expectedOutputFiles(inputSize)));  
                      } else if (fileCountRunner.get() < maxFilesToRewrite) {  
                        int remainingSize = maxFilesToRewrite - fileCountRunner.get();  
                        int scanTasksToRewrite = Math.min(fileScanTasks.size(), remainingSize);  
                        selectedFileGroups.add(  
                            newRewriteGroup(  
                                ctx,  
                                partition,  
                                fileScanTasks.subList(0, scanTasksToRewrite),  
                                inputSplitSize(inputSize),  
                                expectedOutputFiles(inputSize)));  
                        fileCountRunner.getAndAdd(scanTasksToRewrite);  
                      }  
                    });  
          });  
  Map<StructLike, Integer> groupsInPartition = plan.transformValues(List::size);  
  int totalGroupCount = groupsInPartition.values().stream().reduce(Integer::sum).orElse(0);  
  return new FileRewritePlan<>(  
      CloseableIterable.of(  
          selectedFileGroups.stream()  
              .sorted(RewriteFileGroup.comparator(rewriteJobOrder))  
              .collect(Collectors.toList())),  
      totalGroupCount,  
      groupsInPartition);  
}


private StructLikeMap<List<List<FileScanTask>>> planFileGroups() {  
  TableScan scan =  
      table().newScan().filter(filter).caseSensitive(caseSensitive).ignoreResiduals();  
  
  if (snapshotId != null) {  
    scan = scan.useSnapshot(snapshotId);  
  }  
  // 这里调用走了------------------>
  CloseableIterable<FileScanTask> fileScanTasks = scan.planFiles();  
  
  try {  
    Types.StructType partitionType = table().spec().partitionType();  
    StructLikeMap<List<FileScanTask>> filesByPartition =  
        groupByPartition(table(), partitionType, fileScanTasks);  
    return filesByPartition.transformValues(tasks -> ImmutableList.copyOf(planFileGroups(tasks)));  
  } finally {  
    try {  
      fileScanTasks.close();  
    } catch (IOException io) {  
      LOG.error("Cannot properly close file iterable while planning for rewrite", io);  
    }  
  }  
}
```
```java
// SnapshotScan.java
public CloseableIterable<T> planFiles() {  
  Snapshot snapshot = snapshot();  
  
  if (snapshot == null) {  
    LOG.info("Scanning empty table {}", table());  
    return CloseableIterable.empty();  
  }  
  
  LOG.info(  
      "Scanning table {} snapshot {} created at {} with filter {}",  
      table(),  
      snapshot.snapshotId(),  
      DateTimeUtil.formatTimestampMillis(snapshot.timestampMillis()),  
      ExpressionUtil.toSanitizedString(filter()));  
  
  Listeners.notifyAll(new ScanEvent(table().name(), snapshot.snapshotId(), filter(), schema())); 
  // 拿到字段的id和name
  List<Integer> projectedFieldIds = Lists.newArrayList(TypeUtil.getProjectedIds(schema()));  
  List<String> projectedFieldNames =  
      projectedFieldIds.stream().map(schema()::findColumnName).collect(Collectors.toList());  
  
  Timer.Timed planningDuration = scanMetrics().totalPlanningDuration().start();  
  // 这里核心是执行doPlanFIles()，完成后执行lamba
  return CloseableIterable.whenComplete(  
      doPlanFiles(),
      () -> {  
        planningDuration.stop();  
        Map<String, String> metadata = Maps.newHashMap(context().options());  
        metadata.putAll(EnvironmentContext.get());  
        ScanReport scanReport =  
            ImmutableScanReport.builder()  
                .schemaId(schema().schemaId())  
                .projectedFieldIds(projectedFieldIds)  
                .projectedFieldNames(projectedFieldNames)  
                .tableName(table().name())  
                .snapshotId(snapshot.snapshotId())  
                .filter(  
                    ExpressionUtil.sanitize(  
                        schema().asStruct(), filter(), context().caseSensitive()))  
                .scanMetrics(ScanMetricsResult.fromScanMetrics(scanMetrics()))  
                .metadata(metadata)  
                .build();  
        context().metricsReporter().report(scanReport);  
      });  
}
```
```java
// DataTableScan.java
public CloseableIterable<FileScanTask> doPlanFiles() {  
  Snapshot snapshot = snapshot();  
  
  FileIO io = table().io();  
  List<ManifestFile> dataManifests = snapshot.dataManifests(io);  
  List<ManifestFile> deleteManifests = snapshot.deleteManifests(io);  
  scanMetrics().totalDataManifests().increment((long) dataManifests.size());  
  scanMetrics().totalDeleteManifests().increment((long) deleteManifests.size());  
  ManifestGroup manifestGroup =   
      new ManifestGroup(io, dataManifests, deleteManifests)  
          .caseSensitive(isCaseSensitive())  
          .select(scanColumns())  
          .filterData(filter())  
          .specsById(table().specs())  
          .scanMetrics(scanMetrics())  
          .ignoreDeleted()  
          .columnsToKeepStats(columnsToKeepStats());  
  
  if (shouldIgnoreResiduals()) {  
    manifestGroup = manifestGroup.ignoreResiduals();  
  }  
  
  if (shouldPlanWithExecutor() && (dataManifests.size() > 1 || deleteManifests.size() > 1)) {  
    manifestGroup = manifestGroup.planWith(planExecutor());  
  }  
  
  return manifestGroup.planFiles();  
}
```
```java
// ManifestGroup.java
 /**
   * Returns an iterable of scan tasks. It is safe to add entries of this iterable to a collection
   * as {@link DataFile} in each {@link FileScanTask} is defensively copied.
   *
   * @return a {@link CloseableIterable} of {@link FileScanTask}
   */
  public CloseableIterable<FileScanTask> planFiles() {
    return plan(ManifestGroup::createFileScanTasks);
  }

  public <T extends ScanTask> CloseableIterable<T> plan(CreateTasksFunction<T> createTasksFunc) {
    LoadingCache<Integer, ResidualEvaluator> residualCache =
        Caffeine.newBuilder()
            .build(
                specId -> {
                  PartitionSpec spec = specsById.get(specId);
                  Expression filter = ignoreResiduals ? Expressions.alwaysTrue() : dataFilter;
                  return ResidualEvaluator.of(spec, filter, caseSensitive);
                });

    DeleteFileIndex deleteFiles = deleteIndexBuilder.scanMetrics(scanMetrics).build();

    boolean dropStats = ManifestReader.dropStats(columns);
    if (deleteFiles.hasEqualityDeletes()) {
      select(ManifestReader.withStatsColumns(columns));
    }

    LoadingCache<Integer, TaskContext> taskContextCache =
        Caffeine.newBuilder()
            .build(
                specId -> {
                  PartitionSpec spec = specsById.get(specId);
                  ResidualEvaluator residuals = residualCache.get(specId);
                  return new TaskContext(
                      spec, deleteFiles, residuals, dropStats, columnsToKeepStats, scanMetrics);
                });

    Iterable<CloseableIterable<T>> tasks =
        entries(
            (manifest, entries) -> {
              int specId = manifest.partitionSpecId();
              TaskContext taskContext = taskContextCache.get(specId);
              return createTasksFunc.apply(entries, taskContext);
            });

    if (executorService != null) {
      return new ParallelIterable<>(tasks, executorService);
    } else {
      return CloseableIterable.concat(tasks);
    }
  }
```
## 阶段 2 — 分组（file grouping / packing into tasks）

**做什么**：将上一步的 candidate files 根据分区/策略打包成若干 “file groups”（每个 group 将由一个 Spark task 读取并合并成若干个输出文件）。常见策略：按 partition 分组、按目标大小 `target-file-size-bytes` 做 bin-packing、按 `z-order`/`sort` 策略做文件重排（如果指定了 z-ordering）。  
**源码点**：`BaseRewriteDataFilesAction` 中的 bin-packing 算法和 `RewriteDataFilesSparkAction` 的组织逻辑。你会看到 `numBins` / `pack` / `maxFileSize` 等选项影响分组大小。[GitHub+1](https://github.com/apache/iceberg/issues/4302?utm_source=chatgpt.com)  
**内存/IO侧影响**：分组决定了每个 Spark task 的输入规模（直接影响 executor 内存、shuffle 大小）。合理选择 `min-input-files`、`target-file-size` 能使任务均衡。  
**常见失败**：分组策略 bug（老版本有 V2 表在 group-by-partition 的缺陷）会导致分组不合理或重复重写（见 issue）。请在代码中查看 `grouping` 部分并加日志
```java
public RewriteDataFilesActionResult execute() {  
  CloseableIterable<FileScanTask> fileScanTasks = null;  
  if (table.currentSnapshot() == null) {  
    return RewriteDataFilesActionResult.empty();  
  }  
    /**  
     * 这里会有元数据的刷新  
     */  
  long startingSnapshotId = table.currentSnapshot().snapshotId();  
  try {  
      // TODO read  
    fileScanTasks =  
        table  
            .newScan()  
            .useSnapshot(startingSnapshotId)  
            .caseSensitive(caseSensitive)  
            .ignoreResiduals()  
            .filter(filter)  
            .planFiles();  
  } finally {  
    try {  
      if (fileScanTasks != null) {  
        fileScanTasks.close();  
      }  
    } catch (IOException ioe) {  
      LOG.warn("Failed to close task iterable", ioe);  
    }  
  }  
  
  Map<StructLikeWrapper, Collection<FileScanTask>> groupedTasks =  
      groupTasksByPartition(fileScanTasks.iterator());  
  Map<StructLikeWrapper, Collection<FileScanTask>> filteredGroupedTasks =  
      groupedTasks.entrySet().stream()  
          .filter(kv -> kv.getValue().size() > 1)  
          .collect(Collectors.toMap(Map.Entry::getKey, Map.Entry::getValue));  
  
  // Nothing to rewrite if there's only one DataFile in each partition.  
  if (filteredGroupedTasks.isEmpty()) {  
    return RewriteDataFilesActionResult.empty();  
  }  
  // Split and combine tasks under each partition  
  List<CombinedScanTask> combinedScanTasks =  
      filteredGroupedTasks.values().stream()  
          .map(  
              scanTasks -> {  
                CloseableIterable<FileScanTask> splitTasks =  
                    TableScanUtil.splitFiles(  
                        CloseableIterable.withNoopClose(scanTasks), targetSizeInBytes);  
                return TableScanUtil.planTasks(  
                    splitTasks, targetSizeInBytes, splitLookback, splitOpenFileCost);  
              })  
          .flatMap(Streams::stream)  
          .filter(task -> task.files().size() > 1 || isPartialFileScan(task))  
          .collect(Collectors.toList());  
  
  if (combinedScanTasks.isEmpty()) {  
    return RewriteDataFilesActionResult.empty();  
  }  
  
  List<DataFile> addedDataFiles = rewriteDataForTasks(combinedScanTasks);  
  List<DataFile> currentDataFiles =  
      combinedScanTasks.stream()  
          .flatMap(tasks -> tasks.files().stream().map(FileScanTask::file))  
          .collect(Collectors.toList());  
  replaceDataFiles(currentDataFiles, addedDataFiles, startingSnapshotId);  
  
  return new RewriteDataFilesActionResult(currentDataFiles, addedDataFiles);  
}
```