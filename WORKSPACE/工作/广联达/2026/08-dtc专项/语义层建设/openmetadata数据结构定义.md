# OpenMetadata 语义资源（Semantic Resources）设计视图  
  
> 基线：tag **`2.0.5-release`**（commit `34b871b133`）。  
> 覆盖：概念界定与示例、数据存储格式（DDL / POJO / 存储去向 / 资源间联系）、Java Class 定义、MCP 自然语言检索链路。  
> 本版本语义资源共 **5 种**：`glossary`、`glossaryTerm`、`classification`、`tag`、`metric`。  
  
---  
  
## 0. 概念界定  
  
"语义资源" = 承载业务语义能力的一组实体。每个实体由 JSON Schema 唯一定义（`$comment: "@om-entity-type"`，entityType = 文件名），生成 Java POJO。  
  
| 语义资源 | 描述 | JSON Schema | entityType | REST 路径 | 存储表 |  
|---|---|---|---|---|---|  
| **Glossary** 术语表 | 术语的顶层容器（一个企业通常一个或按域多个），提供审批人、互斥规则 | `entity/data/glossary.json` | `glossary` | `/v1/glossaries` | `glossary_entity` |  
| **GlossaryTerm** 术语 | 业务概念的最小治理单元：定义、同义词、层级、类型化关联，可挂载到表/列 | `entity/data/glossaryTerm.json` | `glossaryTerm` | `/v1/glossaryTerms` | `glossary_term_entity` |  
| **Classification** 标签分类 | 标签的命名空间/分类桶（原 TagCategory），决定子标签是否互斥 | `entity/classification/classification.json` | `classification` | `/v1/classifications` | `classification`（原 `tag_category`） |  
| **Tag** 标签 | 分类下的具体标记，可嵌套子标签，用于机密分级、生命周期等打标 | `entity/classification/tag.json` | `tag` | `/v1/tags` | `tag` |  
| **Metric** 指标 | 语义层指标定义：计算表达式、类型/单位/粒度、维度/度量/过滤器，可挂载到资产 | `entity/data/metric.json` | `metric` | `/v1/metrics` | `metric_entity` |  
  
支撑类型（非实体）：  
  
- `type/tagLabel.json` — `TagLabel`（挂载标注：`tagFQN`、`source: Classification|Glossary`、`labelType`、`state`）  
- `type/termRelation.json` — `TermRelation`（`{relationType, term}`，`relationType` 默认 `relatedTo` 的自由字符串）  
- `configuration/glossaryTermRelationSettings.json` — 术语关系类型注册表（settings JSON，非实体）：种子 10 种关系 `relatedTo / synonym / antonym / broader / narrower / partOf / hasPart / calculatedFrom / usedToCalculate / seeAlso`（`bootstrap/sql/migrations/native/1.13.0/mysql/postDataMigrationSQLScript.sql:71`）  
  
实体类型常量（`openmetadata-service/.../Entity.java`）：  
  
```java  
public static final String METRIC = "metric";                    // L184public static final String GLOSSARY = "glossary";                // L212public static final String GLOSSARY_TERM = "glossaryTerm";       // L213public static final String TAG = "tag";                          // L214public static final String CLASSIFICATION = "classification";    // L215```  
  
### 0.1 各语义资源描述与示例  
  
#### Glossary —— 术语表  
  
企业级业务术语容器。不直接挂载资产，挂载发生在它的术语上。`mutuallyExclusive=true` 表示其直接子术语互斥（一个列只能贴一个）。  
  
```json  
{  
  "id": "a1b2c3d4-0001-4000-8000-000000000001",  
  "name": "BusinessGlossary",  
  "fullyQualifiedName": "BusinessGlossary",  
  "description": "Enterprise-wide business terminology.",  
  "owners": [{ "type": "user", "name": "anna.admin" }],  
  "reviewers": [{ "type": "team", "name": "DataGovernance" }],  
  "mutuallyExclusive": false,  
  "termCount": 12,  
  "usageCount": 47,  
  "entityStatus": "Approved",  
  "version": 0.3, 
  "deleted": false, 
  "provider": "user"  
}  
```  
  
#### GlossaryTerm —— 术语  
  
业务概念的治理单元。FQN 形如 `glossaryName.parentTerm.childTerm`；`tags` 表示"术语自身的标签"（如 `User.PhoneNumber` 带 `PII.Sensitive`），贴到列上时会**派生**出该标签；`relatedTerms` 是类型化语义关联。  
  
```json  
{  
  "id": "a1b2c3d4-0002-4000-8000-000000000002",  
  "name": "PhoneNumber",  
  "displayName": "Phone Number",  
  "description": "A telephone number at which a person may be contacted.",  
  "fullyQualifiedName": "BusinessGlossary.User.PhoneNumber",  
  "synonyms": ["Tel", "ContactNumber"],  
  "glossary": { "type": "glossary", "id": "a1b2c3d4-0001-...", "name": "BusinessGlossary" },  
  "parent": { "type": "glossaryTerm", "id": "a1b2c3d4-0003-...", "name": "User" },  
  "relatedTerms": [  
    { "relationType": "relatedTo", "term": { "type": "glossaryTerm", "id": "a1b2c3d4-0004-...", "name": "EmailAddress" } }  
  ],  "references": [{ "name": "GDPR Article 4(1)", "endpoint": "https://gdpr-info.eu/art-4-gdpr/" }],  
  "tags": [  
    { "tagFQN": "PII.Sensitive", "source": "Classification", "labelType": "Manual", "state": "Confirmed" }  
  ],  "usageCount": 31, "entityStatus": "Approved",  
  "version": 1.2, "deleted": false, "provider": "user"  
}  
```  
  
#### Classification —— 标签分类  
  
标签的命名空间。`mutuallyExclusive=true` 时其下标签互斥。历史上叫 `tag_category`，v008 迁移改名。  
  
```json  
{  
  "id": "a1b2c3d4-0010-4000-8000-000000000010",  
  "name": "PII",  
  "fullyQualifiedName": "PII",  
  "displayName": "PII Classification",  
  "description": "Personally Identifiable Information classification tags.",  
  "mutuallyExclusive": true,  
  "termCount": 3, "usageCount": 128,  
  "entityStatus": "Approved",  
  "version": 0.1, "deleted": false, "provider": "system"  
}  
```  
  
#### Tag —— 标签  
  
分类下的具体标记，可嵌套（`PII.Sensitive.Email`）。贴到表/列上的是 `tag_usage` 行，实体 JSON 里**不存 children**。  
  
```json  
{  
  "id": "a1b2c3d4-0011-4000-8000-000000000011",  
  "name": "Sensitive",  
  "displayName": "Sensitive",  
  "fullyQualifiedName": "PII.Sensitive",  
  "description": "Data that must be protected under privacy regulations.",  
  "classification": { "type": "classification", "id": "a1b2c3d4-0010-...", "name": "PII" },  
  "parent": null,  
  "usageCount": 96,  
  "entityStatus": "Approved",  
  "version": 0.4, "deleted": false, "provider": "user"  
}  
```  
  
#### Metric —— 指标  
  
语义层指标：`metricExpression` 是权威计算定义（SQL 等），`metricType/unitOfMeasurement/granularity` 是检索刻面，`dimensions/measures/filters` 来自语义层建模；通过 `assets`（`APPLIED_TO` 边）挂到表/列/看板。  
  
```json  
{  
  "id": "a1b2c3d4-0020-4000-8000-000000000020",  
  "name": "monthly_recurring_revenue",  
  "displayName": "Monthly Recurring Revenue",  
  "description": "Normalized recurring revenue billed in a month.",  
  "metricExpression": { "language": "SQL", "code": "SELECT SUM(amount) FROM billing.subscriptions WHERE status = 'active'" },  
  "metricType": "SUM",  
  "unitOfMeasurement": "DOLLARS",  
  "granularity": "MONTH",  
  "dimensions": [  
    { "name": "region", "type": "CATEGORICAL", "expression": "customer_region" },  
    { "name": "billing_month", "type": "TIME", "expression": "date_trunc('month', billed_at)" }  
  ],  "measures": [{ "name": "mrr_amount", "aggregation": "SUM(amount)" }],  
  "filters": [{ "where": "status = 'active' AND deleted = false" }],  
  "relatedMetrics": [{ "type": "metric", "id": "a1b2c3d4-0021-...", "name": "arr" }],  
  "assets": [{ "type": "table", "id": "tbl-0001", "name": "fct_subscriptions" }],  
  "tags": [{ "tagFQN": "BusinessGlossary.Revenue", "source": "Glossary", "labelType": "Manual", "state": "Confirmed" }],  
  "entityStatus": "Approved",  
  "version": 2.1, "deleted": false, "provider": "user"  
}  
```  
  
---  
  
## 1. 数据存储格式  
  
### 1.1 存储去向总览（RDB / 索引 / 向量）  
  
每个语义资源有 **1 个权威存储（RDB）** 和若干**派生投影**（检索索引、向量嵌入）。投影坏了可从 RDB 全量重建（reindex）。  
  
| 语义资源 | RDB 权威表 | ES/OS 检索索引 | 向量嵌入（dataAssetEmbeddings） | entity_relationship 中的角色 | tag_usage 中的角色 |  
|---|---|---|---|---|---|  
| glossary | `glossary_entity` | `glossary_search_index`（别名 `glossary`，child: `glossaryTerm`） | ✅ 嵌入（`AvailableEntityTypes` 含 `glossary`） | from：`CONTAINS`→根术语、`HAS`→非根术语 | 作为 source=1 的标签目标（术语表自身挂标少见） |  
| glossaryTerm | `glossary_term_entity` | `glossary_term_search_index`（别名 `glossaryTerm`，parents: `all`/`glossary`/`dataAsset`） | ✅ 嵌入（含 `glossaryTerm`） | 层级 `CONTAINS`/`HAS` 的中间节点；术语间 `RELATED_TO(15)` | ① 自身 `tags` 挂在**术语 FQN** 上（source=0 的分类标签）② 作为挂载源（source=1）贴到表/列 |  
| classification | `classification` | `classification_search_index`（别名 `classification`，child: `tag`） | ❌ 不嵌入 | from：`CONTAINS`→tag | 无 |  
| tag | `tag` | `tag_search_index`（别名 `tag`，parents: `classification`/`all`/`dataAsset`） | ❌ 不嵌入 | `CONTAINS` 层级节点 | 作为挂载源（source=0）贴到表/列 |  
| metric | `metric_entity` | `metric_search_index`（别名 `metric`，parents: `all`/`dataAsset`/`dataAssetEmbeddings`） | ✅ 嵌入（含 `metric`，chunk 携带 `metricExpression/metricType/granularity/unitOfMeasurement`） | from：`RELATED_TO`→metric、`APPLIED_TO`→资产 | 作为挂载目标（source=1，metric 自身的 `tags`） |  
  
其它公共投影：  
  
| 存储 | 内容 | 说明 |  
|---|---|---|  
| `entity_extension` | 每实体每版本一行（`extension = '<entityType>.version.<v>'`） | 版本历史 |  
| `change_event` | 变更事件流 | webhook / 索引增量消费 |  
| ES 别名 `all` / `dataAsset` | 聚合检索别名 | 全局搜索与 MCP `search_metadata` 的路由目标 |  
  
依据：`openmetadata-spec/.../elasticsearch/indexMapping.json`（metric/glossary/glossaryTerm/tag/classification 各段）、`search/vector/utils/AvailableEntityTypes.java:19-49`（可嵌入类型白名单，含 `glossary`/`glossaryTerm`/`metric`，不含 tag/classification）。  
  
### 1.2 Database DDL  
  
存储模型：**每实体一行 JSON 文档**（Document-per-row）。除 `json` 与哈希列（1.1.0+ 的 `nameHash`/`fqnHash`）外，其余列都是从 JSON 抽出的**生成列**（`GENERATED ALWAYS AS`），只用于索引/唯一约束；DAO 层只写 `json` + 哈希列。  
  
#### `entity_relationship` —— 层级 + 术语关系的权威存储  
  
```sql  
-- bootstrap/sql/migrations/flyway/com.mysql.cj.jdbc.Driver/v001__create_db_connection_info.sql:4  
-- 1.13+ 增加 relationType varchar(64)，PK 改为 (fromId, toId, relation, relationType)（术语类型化关系）  
CREATE TABLE IF NOT EXISTS entity_relationship (  
    fromId VARCHAR(36) NOT NULL,                -- 起点实体 id    toId VARCHAR(36) NOT NULL,                  -- 终点实体 id    fromEntity VARCHAR(256) NOT NULL,           -- 起点实体类型（如 glossaryTerm）  
    toEntity VARCHAR(256) NOT NULL,             -- 终点实体类型  
    relation TINYINT NOT NULL,                  -- Relationship 枚举序数（持久化契约，只能追加）  
    jsonSchema VARCHAR(256),                    -- json 列的 schema    json JSON,                                  -- 边上元数据（如 {"relationType":"broader"}）  
    deleted BOOLEAN NOT NULL DEFAULT FALSE,    INDEX edge_index (fromId, toId, relation),    INDEX from_index (fromId, relation),    INDEX to_index (toId, relation),    PRIMARY KEY (fromId, toId, relation));  
```  
  
#### `tag_usage` —— 术语/标签挂载到资产的专用表（不走 entity_relationship）  
  
```sql  
-- v001:370；1.1.x 后 targetFQN/tagFQN 换为 targetFQNHash/tagFQNHash；1.11.5+ 增加 appliedAt/appliedByCREATE TABLE IF NOT EXISTS tag_usage (  
    source TINYINT NOT NULL,            -- TagSource 序数：0=Classification, 1=Glossary  
    tagFQN VARCHAR(256) NOT NULL,       -- 'PII.Sensitive' 或 'BusinessGlossary.User.PhoneNumber'    targetFQN VARCHAR(256) NOT NULL,    -- 被挂载对象 FQN：表 FQN 或【列 FQN】（列没有独立实体行）  
    labelType TINYINT NOT NULL,         -- 0 Manual,1 Propagated,2 Automated,3 Derived,4 Generated    state TINYINT NOT NULL,             -- 0 Suggested, 1 Confirmed    UNIQUE (source, tagFQN, targetFQN));  
```  
  
#### 实体表  
  
```sql  
-- glossary_entity（v001:400；1.1.0 加 nameHash 唯一列）  
CREATE TABLE IF NOT EXISTS glossary_entity (  
    id VARCHAR(36) GENERATED ALWAYS AS (json ->> '$.id') STORED NOT NULL,    name VARCHAR(256) GENERATED ALWAYS AS (json ->> '$.name') NOT NULL,    json JSON NOT NULL,                      -- Glossary 完整 JSON 文档  
    updatedAt BIGINT UNSIGNED GENERATED ALWAYS AS (json ->> '$.updatedAt') NOT NULL,    updatedBy VARCHAR(256) GENERATED ALWAYS AS (json ->> '$.updatedBy') NOT NULL,    deleted BOOLEAN GENERATED ALWAYS AS (json -> '$.deleted'),    PRIMARY KEY (id),    UNIQUE (name));  
  
-- glossary_term_entity（v001:411）  
CREATE TABLE IF NOT EXISTS glossary_term_entity (  
    id VARCHAR(36) GENERATED ALWAYS AS (json ->> '$.id') STORED NOT NULL,    fullyQualifiedName VARCHAR(256) GENERATED ALWAYS AS (json ->> '$.fullyQualifiedName') NOT NULL,    json JSON NOT NULL,                      -- GlossaryTerm 完整 JSON（synonyms/references 内嵌）  
    updatedAt BIGINT UNSIGNED GENERATED ALWAYS AS (json ->> '$.updatedAt') NOT NULL,    updatedBy VARCHAR(256) GENERATED ALWAYS AS (json ->> '$.updatedBy') NOT NULL,    deleted BOOLEAN GENERATED ALWAYS AS (json -> '$.deleted'),    PRIMARY KEY (id),    UNIQUE (fullyQualifiedName));  
  
-- tag_category（v001:350）→ v008:1 改名：RENAME TABLE tag_category TO classification;  
CREATE TABLE IF NOT EXISTS classification (  
    id VARCHAR(36) GENERATED ALWAYS AS (json ->> '$.id') STORED NOT NULL,    name VARCHAR(256) GENERATED ALWAYS AS (json ->> '$.name') NOT NULL,    json JSON NOT NULL, -- JSON stores category information and does not store children    updatedAt BIGINT UNSIGNED GENERATED ALWAYS AS (json ->> '$.updatedAt') NOT NULL,    updatedBy VARCHAR(256) GENERATED ALWAYS AS (json ->> '$.updatedBy') NOT NULL,    deleted BOOLEAN GENERATED ALWAYS AS (json -> '$.deleted'),    UNIQUE (name));  
  
-- tag（v001:360）  
CREATE TABLE IF NOT EXISTS tag (  
    id VARCHAR(36) GENERATED ALWAYS AS (json ->> '$.id') STORED NOT NULL,    fullyQualifiedName VARCHAR(256) GENERATED ALWAYS AS (json ->> '$.fullyQualifiedName') NOT NULL,    json JSON NOT NULL, -- JSON stores all tag attributes and does not store children    updatedAt BIGINT UNSIGNED GENERATED ALWAYS AS (json ->> '$.updatedAt') NOT NULL,    updatedBy VARCHAR(256) GENERATED ALWAYS AS (json ->> '$.updatedBy') NOT NULL,    deleted BOOLEAN GENERATED ALWAYS AS (json -> '$.deleted'),    UNIQUE (fullyQualifiedName));  
  
-- metric_entity（v001:147）  
CREATE TABLE IF NOT EXISTS metric_entity (  
    id VARCHAR(36) GENERATED ALWAYS AS (json ->> '$.id') STORED NOT NULL,    fullyQualifiedName VARCHAR(256) GENERATED ALWAYS AS (json ->> '$.fullyQualifiedName') NOT NULL,    json JSON NOT NULL,    updatedAt BIGINT UNSIGNED GENERATED ALWAYS AS (json ->> '$.updatedAt') NOT NULL,    updatedBy VARCHAR(256) GENERATED ALWAYS AS (json ->> '$.updatedBy') NOT NULL,    deleted BOOLEAN GENERATED ALWAYS AS (json -> '$.deleted'),    PRIMARY KEY (id),    UNIQUE (fullyQualifiedName));  
```  
  
### 1.3 Java POJO ↔ DDL 映射说明  
  
POJO 由 JSON Schema 经 jsonschema2pojo 生成（`org.openmetadata.schema.entity.data.*` / `...classification.*`），**`json` 列就是整个 POJO 的 Jackson 序列化**。映射关系：  
  
```  
POJO 字段                          │  落到哪                          │  说明  
──────────────────────────────────┼─────────────────────────────────┼──────────────────────────────  
id / name / fullyQualifiedName     │  json 列 + 对应生成列            │  生成列由 json 抽出，做 PK/唯一键  
                                   │  +（1.1.0+）nameHash/fqnHash    │  哈希列手写，DAO insert/update 时写入  
description/version/updatedAt/…   │  仅 json 列                      │  读时整段反序列化回 POJOowners/reviewers/domains/tags      │  仅 json 列（EntityReference/    │  引用存在 JSON 里；权威关系另存  
  /glossary/parent/assets/…        │   TagLabel 数组）                │  entity_relationship / tag_usagerelatedTerms: TermRelation[]       │  json + entity_relationship      │  RELATED_TO(15) 边，json={"relationType":…}  
children / termCount / usageCount  │  不落库（读时计算）               │  fieldFetcher 从边/tag_usage 回填  
deleted                            │  json + 生成列                   │  软删除标记  
```  
  
DAO 只写两列（`EntityDAO.java:135`）：  
  
```java  
@ConnectionAwareSqlUpdate(  
    value = "INSERT INTO <table> (<nameHashColumn>, json) VALUES (:nameHashColumnValue, :json)",    connectionType = MYSQL)void insert(@Define("table") String table, @Define("nameHashColumn") String nameHashColumn,  
            @BindFQN("nameHashColumnValue") String nameHashColumnValue, @BindJson("json") String json);  
```  
  
三个映射方向要分清：  
  
1. **POJO → json 列**：`storeEntity()` 时 `JsonUtils.pojoToJson(entity)` 整体序列化；  
2. **POJO 的引用字段 → 边/挂载表**：`storeRelationships()` 把 `parent/glossary/assets/relatedTerms` 等 EntityReference 写成 `entity_relationship` 行，把 `tags` 写成 `tag_usage` 行 —— **JSON 里的引用是冗余/读视图，边和挂载表才是权威**；  
3. **RDB → 检索索引/向量**：`SearchIndex.buildSearchIndexDoc()`（五阶段，含 `TaggableIndex` 把 tag_usage 拆成 `tags/tier/classificationTags/glossaryTags`）写入 `*_search_index`；`VectorEmbeddingHandler`（白名单 `AvailableEntityTypes`）把 glossary/glossaryTerm/metric 嵌入 `embedding` 字段并生成 chunk 文档进 `dataAssetEmbeddings`。  
  
### 1.4 语义资源之间的联系（存储形态）  
  
联系分两类存储：**层级/语义关系 → `entity_relationship` 边表**；**挂载（术语/标签贴到资产）→ `tag_usage` 挂载表**（metric 挂资产例外，走 `APPLIED_TO` 边）。  
  
`Relationship` 枚举（序数持久化在库中，`type/entityRelationship.json`）：  
  
| # | 枚举 | 用途 |  
|---|---|---|  
| 0 | `CONTAINS` | glossary→根术语、父术语→子术语、classification→tag、父 tag→子 tag |  
| 10 | `HAS` | glossary→非根术语（回指） |  
| 14 | `APPLIED_TO` | metric→资产 |  
| 15 | `RELATED_TO` | 术语↔术语类型化语义关系、metric↔metric `relatedMetrics` |  
  
#### 架构图（层级 + 语义关系 + 挂载）  
  
```  
═══════════════════════════ entity_relationship（边表：层级权威存储）═══════════════════════════  
  
  术语域                                     标签域                                      指标域  
  ──────                                     ──────                                      ──────  
  [glossary]                                [classification]                            [metric A]     │ CONTAINS(0) 根术语                        │ CONTAINS(0)（根/子统一）                    ▲ │     ▼                                          ▼                                          │ │ RELATED_TO(15, 双向)  
  [term 根] ──CONTAINS(0)──▶ [term 子]        [tag 根] ──CONTAINS(0)──▶ [tag 子]          │ │  
     ▲ HAS(10) 回指                              │                                          │ │     │                                          └──CONTAINS(0)──▶ [tag 孙]             [metric B]  
  [glossary] ──HAS(10)──▶ [非根 term]           （FQN: PII.Sensitive.Email）  
  
  写入点：GlossaryTermRepository:1598/1604      TagRepository:986/995                 MetricRepository:214  
          （GlossaryRepository / ClassificationRepository 的 storeRelationships 是 no-op）  
  
  
═══════════════════ 术语间类型化语义边（relationType 字符串，settings 注册表治理，非实体）═══════════════════  
  
  [termA] ──RELATED_TO(15, relationType=…)──▶ [termB]             relationType ∈ {relatedTo, synonym, antonym, broader, narrower,                              partOf, hasPart, calculatedFrom, usedToCalculate, seeAlso}             json = {"relationType": "<canonicalType>"}      （1.13+ 落到 relationType 列）  
             from/to 按字典序归一 → (pair, relationType) 唯一一行，双向  
             注册表 = configuration/glossaryTermRelationSettings（settings JSON）  
             inverseRelation 自动换算：broader↔narrower、partOf↔hasPart、calculatedFrom↔usedToCalculate  
  
  
═══════════════════ 挂载关系（语义资源 → 数据资产）═══════════════════  
  
  ┌─ tag_usage 表（★ 不是边；targetFQN 可以是【列】FQN）─────────────────────────┐  
  │                                                                               │  │  [glossaryTerm] ──(source=1 Glossary)──▶ [table / table.column]                │  │  [tag]          ──(source=0 Classification)──▶ [table / table.column]          │  │                                                                               │  │  两级派生：term 自身的 tags 挂在【术语 FQN】上 → 贴到列时读时派生为              │  │            labelType=DERIVED（不落库）；派生逻辑 TagLabelUtil.addDerivedTags    │  └───────────────────────────────────────────────────────────────────────────────┘  
  [metric] ──APPLIED_TO(14)──▶ [table|column|dashboard|...]        （assets 字段，权威在边）  
  [metric] ──RELATED_TO(15)─▶ [metric]                             （relatedMetrics，双向）  
  [metric] ──tags(TagLabel)─▶ tag_usage                            （metric↔glossary 的唯一纽带）  
```  
  
#### 权威性对照  
  
| 概念 | 权威存储 | JSON 字段角色 |  
|---|---|---|  
| glossary→根术语 | 边 `CONTAINS(0)` | `glossary` 引用冗余 |  
| glossary→非根术语 | 边 `HAS(10)` | 读时回填 |  
| 父术语→子术语 | 边 `CONTAINS(0)` | `parent`/`children` 镜像 |  
| classification→tag | 边 `CONTAINS(0)`（各级统一） | `classification`/`parent` 冗余 |  
| 术语间类型化关系 | 边 `RELATED_TO(15)` + `relationType` | `relatedTerms` 读时覆盖 |  
| `synonyms` / `references` | 纯 JSON 列 | 即权威 |  
| 术语/标签挂资产 | **`tag_usage`** | `tags: TagLabel[]` 是读视图 |  
| metric→资产 | 边 `APPLIED_TO(14)` | `assets` 读时填充 |  
| metric↔metric | 边 `RELATED_TO(15)` | `relatedMetrics` 读时填充 |  
| metric→glossary | **无直连**，仅经 `tag_usage` | — |  
  
#### 关系写入点（源码）  
  
```java  
// GlossaryTermRepository.java —— glossary 侧所有边由术语侧写入  
// storeRelationships:644  
@Override  
public void storeRelationships(GlossaryTerm entity) {  
  addGlossaryRelationship(entity);  
  addParentRelationship(entity);  
  validateRelationCardinality(entity.getId(), listOrEmpty(entity.getRelatedTerms()));  
  for (TermRelation termRelation : listOrEmpty(entity.getRelatedTerms())) {  
    String relationType = termRelation.getRelationType() != null ? termRelation.getRelationType() : "relatedTo";    validateRelationType(relationType);                              // 必须在 glossaryTermRelationSettings 注册表内  
    UUID toId = termRelation.getTerm().getId();    String canonicalType = computeCanonicalRelationType(entity.getId(), toId, relationType);  
    String json = String.format("{\"relationType\":\"%s\"}", canonicalType);  
    addRelationship(entity.getId(), toId, GLOSSARY_TERM, GLOSSARY_TERM,  
        Relationship.RELATED_TO, canonicalType, json, true);         // 字典序归一，单行双向  
    RdfUpdater.addGlossaryTermRelation(entity.getId(), toId, relationType);  
  }}  
  
// :1598 —— 根术语 CONTAINS，非根 HAS（回指）  
private void addGlossaryRelationship(GlossaryTerm term) {  
  Relationship relationship = term.getParent() != null ? Relationship.HAS : Relationship.CONTAINS;  
  addRelationship(term.getGlossary().getId(), term.getId(), GLOSSARY, GLOSSARY_TERM, relationship);  
}  
  
// :1604 —— 父术语 CONTAINS 子术语  
private void addParentRelationship(GlossaryTerm term) {  
  if (term.getParent() != null) {  
    addRelationship(term.getParent().getId(), term.getId(), GLOSSARY_TERM, GLOSSARY_TERM, Relationship.CONTAINS);  
  }}  
  
// TagRepository.java —— classification 侧（根/子统一 CONTAINS，无 HAS 回指）  
// :482 storeRelationships → addClassificationRelationship + addParentRelationship  
private void addClassificationRelationship(Tag term) {              // :986  
  addRelationship(term.getClassification().getId(), term.getId(), Entity.CLASSIFICATION, TAG, Relationship.CONTAINS);  
}  
private void addParentRelationship(Tag term) {                      // :995  
  if (term.getParent() != null) {  
    addRelationship(term.getParent().getId(), term.getId(), TAG, TAG, Relationship.CONTAINS);  
  }}  
  
// MetricRepository.storeRelationships:214 —— metric 只有 RELATED_TO + APPLIED_TOpublic void storeRelationships(Metric metric) {  
  for (EntityReference relatedMetric : listOrEmpty(metric.getRelatedMetrics())) {  
    addRelationship(metric.getId(), relatedMetric.getId(), METRIC, METRIC, Relationship.RELATED_TO, true);  
  }  for (EntityReference asset : listOrEmpty(metric.getAssets())) {  
    addRelationship(metric.getId(), asset.getId(), METRIC, asset.getType(), Relationship.APPLIED_TO);  
  }}  
```  
  
---  
  
## 2. Java Class 定义  
  
### 2.1 分层模板（5 个语义资源共用）  
  
```  
JSON Schema ──jsonschema2pojo──▶ POJO (org.openmetadata.schema.entity.data.* / ...classification.*)  
                                     │ implements EntityInterface<EntityStatus>                                     ▼Resource (EntityResource<T,K>) ──▶ Repository (EntityRepository<T>) ──▶ DAO (EntityDAO<T>) ──▶ *_entity 表  
```  
  
三个泛型基类：  
  
```java  
// openmetadata-service/.../jdbi3/EntityRepository.java:322  
public abstract class EntityRepository<T extends EntityInterface> { ... }  
// openmetadata-service/.../resources/EntityResource.java:124  
public abstract class EntityResource<T extends EntityInterface, K extends EntityRepository<T>> { ... }  
  
// openmetadata-service/.../jdbi3/EntityDAO.java:60  
public interface EntityDAO<T extends EntityInterface> { ... }```  
  
### 2.2 POJO（生成类，源 = JSON Schema 的 javaType）  
  
```json  
// glossary.json（其余实体同构）  
{  
  "title": "Glossary",  
  "$comment": "@om-entity-type",  
  "javaType": "org.openmetadata.schema.entity.data.Glossary",  
  "javaInterfaces": ["org.openmetadata.schema.EntityInterface<org.openmetadata.schema.type.EntityStatus>"],  
  "required": ["id", "name", "description"]  
}  
```  
  
| JSON Schema | 生成的 Java 类 | 关键字段 |  
|---|---|---|  
| `entity/data/glossary.json` | `...entity.data.Glossary` | `reviewers`、`mutuallyExclusive`、`termCount`(派生) |  
| `entity/data/glossaryTerm.json` | `...entity.data.GlossaryTerm` | `glossary`、`parent`、`children`、`relatedTerms: TermRelation[]`、`synonyms`、`references`、`tags: TagLabel[]` |  
| `entity/classification/classification.json` | `...entity.classification.Classification` | `mutuallyExclusive`、`termCount`(派生) |  
| `entity/classification/tag.json` | `...entity.classification.Tag` | `classification`、`parent`、`children`、`deprecated` |  
| `entity/data/metric.json` | `...entity.data.Metric` | `metricExpression{language,code}`、`metricType`、`unitOfMeasurement`、`granularity`、`dimensions[]`、`measures[]`、`filters[]`、`relatedMetrics`、`assets`、`derivedFrom` |  
  
### 2.3 Repository（`openmetadata-service/.../jdbi3/`）  
  
```java  
public class GlossaryRepository extends EntityRepository<Glossary> { }                 // GlossaryRepository.java:94  
public class GlossaryTermRepository extends EntityRepository<GlossaryTerm> { }         // GlossaryTermRepository.java:135  
public class ClassificationRepository extends EntityRepository<Classification> { }     // ClassificationRepository.java:80  
public class TagRepository extends EntityRepository<Tag> { }                           // TagRepository.java:95  
public class MetricRepository extends EntityRepository<Metric> { }                     // MetricRepository.java:79  
```  
  
各 Repository 重写：`prepare` / `storeEntity` / `setFullyQualifiedName` / `storeRelationships` / `getUpdater`（内部类 `EntityUpdater` 做 JSON diff + 版本递增）/ `clearEntitySpecificRelationshipsForMany`。写边源码见 §1.4。  
  
### 2.4 Resource（REST，`openmetadata-service/.../resources/`）  
  
```java  
@Path("/v1/glossaries")  
public class GlossaryResource extends EntityResource<Glossary, GlossaryRepository> { }              // :83  
@Path("/v1/glossaryTerms")  
public class GlossaryTermResource extends EntityResource<GlossaryTerm, GlossaryTermRepository> { }  // :111  
@Path("/v1/classifications")  
public class ClassificationResource extends EntityResource<Classification, ClassificationRepository> { }  // :81  
@Path("/v1/tags")  
public class TagResource extends EntityResource<Tag, TagRepository> { }                             // :97  
@Path("/v1/metrics")  
public class MetricResource extends EntityResource<Metric, MetricRepository> { }                    // :81  
```  
  
特有端点：GlossaryTermResource 的 `/{id}/assets/add|remove`、`/{id}/relations`、`/relationsGraph`、`/relationTypes/usage`、`/{id}/moveAsync`；TagResource 的 `/{id}/assets/*`、`/feedback/*`、`/assets/counts`；MetricResource 的 `/bulk`、`/customUnits`、`/{id}/followers`、`/{id}/vote`。  
  
### 2.5 DAO（表名绑定，见 §1.2 的表；均在 `jdbi3/CollectionDAO.java`）  
  
```java  
// CollectionDAO.java:5938/5950/5884  
interface GlossaryDAO extends EntityDAO<Glossary> { getTableName() -> "glossary_entity"; }  
interface GlossaryTermDAO extends EntityDAO<GlossaryTerm> { getTableName() -> "glossary_term_entity"; }  // fqnHash  
interface MetricDAO extends EntityDAO<Metric> { getTableName() -> "metric_entity"; }  
  
// CollectionDAO.java:6741/6770/6889  
interface ClassificationDAO extends EntityDAO<Classification> { getTableName() -> "classification"; }  
interface TagDAO extends EntityDAO<Tag> { getTableName() -> "tag"; }                                     // fqnHash  
interface TagUsageDAO { getTableName() -> "tag_usage"; }                                                 // 挂载行  
```  
  
---  
  
## 3. MCP 自然语言检索链路  
  
### 3.1 MCP 服务形态  
  
`openmetadata-mcp` 模块（Dropwizard + `io.modelcontextprotocol.sdk` 1.1.1），由 `OpenMetadataApplication:474-494` 反射加载 `org.openmetadata.mcp.McpServer`。**无状态** HTTP 服务，Servlet 挂载 `/mcp/*`（默认 `/api/v1/mcp`），capabilities = **tools + prompts only（不注册 MCP resources）**。工具目录数据驱动：`openmetadata-mcp/src/main/resources/json/data/mcp/tools.json`（本版本 **15 个工具**）。  
  
配置：`openmetadata-spec/.../configuration/mcpConfiguration.json`（`enabled`、`path`、`baseUrl`、`allowedOrigins`、`connectTimeout`、`readTimeout` 等 10 个字段）。仓库根目录的 `mcp-proxy.py` 是**未入库**的本地 stdio→HTTP 桥接调试脚本（内含硬编码 token，勿提交）。  
  
### 3.2 请求链路总图  
  
```  
 LLM 客户端（Claude/Cursor/...）  
   │  MCP JSON-RPC（tools/list, tools/call）  
   ▼ [可选] mcp-proxy.py（stdio → HTTP，本地调试）  
   ▼ Jetty Servlet  /mcp/*                                    McpServer.java:167-203   │  OAuthHttpStatelessServerTransportProvider   │    · Bearer JWT 校验（AuthEnrichedMcpContextExtractor 注入 Authorization/Mcp-Client-Name）  
   │    · OAuth 全栈：/.well-known/*, /mcp/callback, UserSSOOAuthProvider  
   ▼ McpStatelessSyncServer（SDK，无会话态）  
   │  SyncToolSpecification handler                       McpServer.java:280-307   │    · jwtFilter.getCatalogSecurityContext(token) → RBAC 主体  
   │    · ImpersonationContext = MCP bot   ▼ DefaultToolContext.callToolWithMetadata                   DefaultToolContext.java:78-133   │  switch (toolName) → new XxxTool().execute(authorizer, securityContext, params)   │  结果超限 → 截断信封（McpResponseTrim / maxResponseChars）  
   │  finally: McpUsageRecorder.record(...)   ▼ 进程内后端（★ 全部 in-process，MCP 不调用 /v1/search/* REST）  
   ├─ SearchRepository.search(...)            —— 关键字/DSL 检索（ES/OS 索引）  
   ├─ VectorIndexService.search(...)          —— KNN 向量检索（dataAssetEmbeddings）  
   ├─ AIContextFinder / AIContextBuilder      —— 知识层（glossaryTerm/metric/page）上下文  
   └─ Entity.getEntityByName / readEntityForCaller —— 实体仓储直读（RDB）  
```  
  
### 3.3 工具清单与语义资源参与点（15 个）  
  
| 工具 | 用途 | 语义资源参与 |  
|---|---|---|  
| `search_metadata` | 关键字/DSL 检索（entityType 路由到索引） | `entityType` 枚举含 `glossary, glossaryTerm, tag, classification, metric`；tools.json 明示：`queryFilter` 里 `tags.tagFQN` term 查询**同时命中分类标签和术语**（两者都进 `tags` 数组）；`SearchMetadataTool:127/711-714` 按 entityType 解析对应 `*_index_mapping.json` |  
| `semantic_search` | KNN 向量检索 + 过滤 | 过滤字段（`SemanticSearchTool:41-57`）含 `tags, tier` + 指标刻面 `metricType, granularity, unitOfMeasurement`；metric 命中返回 `metricExpression`(≤1000 字符) 等四元组区分同名指标；需 `isVectorEmbeddingEnabled`（`:82`） |  
| `find_context` | **知识层语义检索**：glossary 术语定义、metric 定义、Context 文章 → 候选资产 | `AIContextFinder.CONTEXT_ENTITY_TYPES = [GLOSSARY_TERM, METRIC, PAGE]`（硬编码，`AIContextFinder:50-53`）；术语命中 → `tags.tagFQN` 反查挂载资产（`:231-256`）；metric 命中 → `APPLIED_TO` 边反查（`:198-199`）；仅 APPROVED 术语入上下文（`:165-167`）；访问需 VIEW_ALL（`AIContextFinderAccess:37-43`） |  
| `get_entity_details` | 实体详情（`include=context` 出 Context Profile） | `GetEntityTool:371-375` → `AIContextBuilder` 打包：分类标签、`TagSource.GLOSSARY` 术语（APPROVED + PBAC，`AIContextBuilder:829`）、`findFrom(APPLIED_TO, METRIC)` 挂在该资产上的指标（`:947`） |  
| `get_persona_context` | Persona 物化上下文文档 | `PersonaContextBuilder:76`：知识类型 `[GLOSSARY_TERM, PAGE, METRIC]`；段含 `GLOSSARY_TERMS`/`METRICS`；资产指标批量 `findFromBatch(APPLIED_TO)`（`:509`）；术语读资产索引 `glossaryTags` |  
| `get_user_context` | 当前用户身份/资产 | 治理缺口扫描把 `glossaryTerm` 作为缺口资产类型（`UserContextSearch:48-60`） |  
| `company_context` | Context Memory 知识胶囊 | 无语义资源（仅 contextMemory） |  
| `create_entity` / `patch_entity` | 创建/修补实体 | 校验 metric 的 `metricExpression.{language,code}`（`CreateEntityTool:233-258`）、tag/classification 派生（`:171-207`）、`mutuallyExclusive` 不可变告警（`:264-279`） |  
| `describe_entity_type` | 实体 schema 描述 | 含 5 种语义资源的 create 属性 |  
| 其余：`get_entity_lineage`/`create_lineage`/`get_test_definitions`/`create_test_case`/`root_cause_analysis` | 血缘/测试/诊断 | 边上的 `relationshipType` 标签来自外键约束方向映射，非术语关系 |  
  
tools.json 中指导 LLM 使用语义资源的关键描述（原文）：  
  
- `search_metadata`：*"…a 'term' clause on 'tags.tagFQN' in 'queryFilter' matches every asset carrying that classification tag OR glossary term, since both land in the 'tags' array."*  
- `semantic_search`：*"Metric results additionally carry 'metricExpression', 'metricType', 'granularity' and 'unitOfMeasurement' — enough to tell similarly named metrics apart…"*  
- `find_context`：*"…semantically searches the company-knowledge layer (glossary terms, metrics, Context Center articles) and returns the relevant definitions PLUS the candidate data assets each concept points to…"*  
- `get_entity_details`（context）：*"'context' adds the asset's Context Profile - attached glossary terms and articles plus the structural context … which is what to use before generating SQL against an asset."*  
  
### 3.4 典型自然语言问题的检索路径  
  
```  
问："上个月各地区 MRR 是多少？哪些字段可以用来分组？"  
 ① 语义模糊 → semantic_search：KNN 命中 metric monthly_recurring_revenue     （命中自带 metricExpression/metricType=SUM/granularity=MONTH/DOLLARS —— 区分同名指标）  
 ② 深挖 → get_entity_details(metric, include=context)     → AIContextBuilder 返回：挂载资产(APPLIED_TO 边)、glossary 术语(Revenue)、分类标签  
 ③ 概念不清 → find_context("recurring revenue")     → AIContextFinder 在 [glossaryTerm, metric, page] 向量检索  
     → 术语 Revenue → tags.tagFQN 反查打了该术语的表；metric → APPLIED_TO 反查  
 ④ 精确过滤 → search_metadata：queryFilter {"term": {"tags.tagFQN": "BusinessGlossary.Revenue"}}  
     或 {"terms": {"entityType": ["metric"]}} + metricType/granularity 刻面  
 ⑤ 全量反查 → "所有标了 PII.Sensitive 的资产"  
     → search_metadata + tags.tagFQN term 查询（分类标签与术语统一命中）  
```  
  
### 3.5 检索底座（语义资源如何进 ES/OS 与向量）  
  
- 索引（`indexMapping.json`，别名与父子见 §1.1）；全局别名 `all`（`SearchClient.java:37`）。  
- 索引类（`search/indexes/`）：`GlossaryIndex`、`GlossaryTermIndex`（排除 children、注入父 glossary 的 `mutuallyExclusive`）、`ClassificationIndex`、`TagIndex`（`classification.name` boost 7.0）、`MetricIndex`（dimension/measure FQN 折入 `fqnParts`）、`TaggableIndex` —— 后者把 `tag_usage` 拆成 `tags / tier / classificationTags / glossaryTags` 四个字段写进每个资产文档，**挂载关系在索引期物化，检索时直接读 `_source`**；ES 侧还有 `TAG_RESEPARATION_SCRIPT`（`SearchClient:107-133`）在脚本写后重算拆分。  
- 向量：`search/vector/` 下 `VectorDocBuilder` 把术语名/定义/指标表达式编进 `textToEmbed`（metric 额外把 `metricExpression/metricType/granularity/unitOfMeasurement` 写进 chunk 字段，`VectorDocBuilder:379-406`），`VectorEmbeddingHandler` 按 `AvailableEntityTypes` 白名单（含 glossary/glossaryTerm/metric）写入实体文档 `embedding` 字段 + chunk 文档进 `dataAssetEmbeddings`；混合检索权重 `keywordWeight/semanticWeight`（默认 0.6/0.4，RRF）。  
- 指标刻面检索配置：`searchSettings.json` `assetType: "metric"` —— `name.keyword` 20.0 精确、`displayName` 10.0、`fqnParts` 5.0；无 dimensions/measures 检索字段。  
- NLQ：`/v1/search/nlq/query` 存在但内置仅 `NoOpNLQService`，且 MCP 不调用它 —— MCP 的"自然语言"能力由 **LLM 客户端 + 上述工具**组合完成，服务端不做 NL→DSL 翻译。  
  
---  
  
## 4. 一句话总结  
  
**Schema-first + Document-per-row + Edge-table + 挂载表**：5 个语义实体各自一张 `*_entity` 表存整份 JSON（生成列做索引，POJO 即 json 列）；层级与术语类型化关系全部外置到 `entity_relationship`（枚举序数是持久化契约；术语关系的 `relationType` 由 `glossaryTermRelationSettings` settings 注册表治理）；术语/标签挂资产单独走 `tag_usage`（支持列级 FQN），metric 挂资产走 `APPLIED_TO` 边。每个资源另有检索索引投影（`*_search_index`，`TaggableIndex` 索引期物化挂载标签）；glossary/glossaryTerm/metric 额外进向量嵌入（`dataAssetEmbeddings`），tag/classification 只进关键字索引。MCP 通过 15 个声明式工具把这条链暴露给 LLM：`search_metadata` 关键字、`semantic_search` KNN（带指标刻面）、`find_context` 知识层 —— 全部进程内调用，语义资源既是检索对象，也是其它资产检索的**过滤器与上下文增强层**。