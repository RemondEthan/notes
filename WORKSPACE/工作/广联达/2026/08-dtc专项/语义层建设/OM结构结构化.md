# OpenMetadata 语义资源（Semantic Resources）设计视图  
  
> 基线：tag **`2.0.5-release`**（commit `34b871b133`）。语义资源共 **5 种**：`glossary`、`glossaryTerm`、`classification`、`tag`、`metric`。  
> **Java POJO 说明**：以下 POJO 均为 `jsonschema2pojo` 从 JSON Schema 生成的真实编译产物（`make generate`），源码在  
> `openmetadata-spec/target/generated-sources/jsonschema2pojo/org/openmetadata/schema/...`（不入库，随 openmetadata-spec jar 打包）。  
> 文中行号 = 该生成文件实测行号。  
  
---  
  
## 0. 概念界定  
  
| 语义资源 | 描述 | JSON Schema | entityType | REST 路径 | 存储表 | 生成 POJO |  
|---|---|---|---|---|---|---|  
| **Glossary** 术语表 | 术语的顶层容器，提供审批人、互斥规则 | `entity/data/glossary.json` | `glossary` | `/v1/glossaries` | `glossary_entity` | `...entity.data.Glossary` |  
| **GlossaryTerm** 术语 | 业务概念的最小治理单元：定义、同义词、层级、类型化关联，可挂载到表/列 | `entity/data/glossaryTerm.json` | `glossaryTerm` | `/v1/glossaryTerms` | `glossary_term_entity` | `...entity.data.GlossaryTerm` |  
| **Classification** 标签分类 | 标签的命名空间/分类桶（原 TagCategory） | `entity/classification/classification.json` | `classification` | `/v1/classifications` | `classification` | `...entity.classification.Classification` |  
| **Tag** 标签 | 分类下的具体标记，可嵌套子标签 | `entity/classification/tag.json` | `tag` | `/v1/tags` | `tag` | `...entity.classification.Tag` |  
| **Metric** 指标 | 语义层指标：计算表达式、类型/单位/粒度、维度/度量/过滤器 | `entity/data/metric.json` | `metric` | `/v1/metrics` | `metric_entity` | `...entity.data.Metric` |  
  
支撑类型（非实体）：`type/termRelation.json` → `TermRelation`（`{relationType, term}`）；`type/tagLabel.json` → `TagLabel`；术语关系类型注册表 = settings `configuration/glossaryTermRelationSettings`（10 种种子关系，非实体）。  
  
---  
  
- [ ] **语义层数据操作要记录日志，审计使用**
## 1. Glossary —— 术语表  
  
### 1.1 Java POJO（生成类）  
  
**文件**：`openmetadata-spec/target/generated-sources/jsonschema2pojo/org/openmetadata/schema/entity/data/Glossary.java`（1142 行，`public class Glossary implements EntityInterface` 在 `:66`，类注解 `@Generated("jsonschema2pojo")`）  
  
| 字段（Java）                                             | 类型                          | JSON 名               | 默认值/说明                                        | 保留       |
| ---------------------------------------------------- | --------------------------- | -------------------- | --------------------------------------------- | -------- |
| `id`                                                 | `UUID`                      | `id`                 | 服务端生成                                         | ✅        |
| `name`                                               | `String`                    | `name`               | 唯一名                                           | ✅        |
| `fullyQualifiedName`                                 | `String`                    | `fullyQualifiedName` | = name（顶层无父）                                  | ✅        |
| ~~`displayName` / `description`~~                    | ~~`String`~~                | ~~同名~~               |                                               | ~~❌~~    |
| `version`                                            | `Double`                    | `version`            | **默认 `0.1D`**                                 | ✅        |
| ==`createdAt`==                                      | ==`Long`==                  | ==`updatedAt`==      | ==Unix 毫秒==                                   | ==✅ 新增== |
| `updatedAt`                                          | `Long`                      | `updatedAt`          | Unix 毫秒                                       | ✅        |
| `updatedBy` / `impersonatedBy`                       | `String`                    | 同名                   |                                               | ✅        |
| ~~`href`~~                                           | ~~`URI`~~                   | ~~`href`~~           |                                               | ~~❌~~    |
| ~~`reviewers`~~                                      | ~~`List<EntityReference>`~~ | ~~`reviewers`~~      | ~~默认 `[]`；**落库前被剥离**（见 1.4）~~                 | ~~❌~~    |
| `owners`                                             | `List<EntityReference>`     | `owners`             | 默认 `null`；剥离                                  | ✅        |
| `tags`                                               | `List<TagLabel>`            | `tags`               | 默认 `[]`；剥离 → `tag_usage`                      | ✅        |
| `usageCount` / `termCount`                           | `Integer`                   | 同名                   | **派生字段，读时计算**   *先保留，统计成果没准需要*                | ✅        |
| `changeDescription` / `incrementalChangeDescription` | `ChangeDescription`         | 同名                   | 版本 diff                                       | ❌        |
| `deleted`                                            | `Boolean`                   | `deleted`            | 默认 `false`（软删）                                | ✅        |
| `disabled`                                           | `Boolean`                   | `disabled`           | 系统术语表禁删标记   -- **上线状态**                       | ✅        |
| `mutuallyExclusive`                                  | `Boolean`                   | `mutuallyExclusive`  | 默认 `false`；直接子术语互斥  *抄代码的时候或许需要，搞个默认值 false？* | ✅        |
| `domains` / `dataProducts`                           | `List<EntityReference>`     | 同名                   | **数据域**                                       | ✅        |
| `dataContract`                                       | `EntityReference`           | `dataContract`       | 做关联使用  -- 保留，使用方式待定                           | ✅        |
| ~~`votes`~~                                          | ~~`Votes`~~                 | ~~`votes`~~          |                                               | ~~❌~~    |
| `extension`                                          | `Object`                    | `extension`          | 自定义扩展                                         | ✅        |
| ~~`provider`~~                                       | ~~`ProviderType`~~          | ~~`provider`~~       | ~~默认 `"user"`~~                               | ~~❌~~    |
| ~~`entityStatus`~~                                   | ~~`EntityStatus`~~          | ~~`entityStatus`~~   | ~~默认 `"Unprocessed"`~~                        | ~~❌~~    |
| `namespaces`                                         | `List<OntologyNamespace>`   | `namespaces`         | 默认 `[]`；前缀+IRI 命名空间（**留在 json 列**）   整个默认值先留着 | ✅        |
  
### 1.2 数据库 DDL  
  
```sql  
-- v001:400-416（flyway/com.mysql.cj.jdbc.Driver/v001__create_db_connection_info.sql）  
CREATE TABLE glossary_entity (  
    id VARCHAR(36) GENERATED ALWAYS AS (json ->> '$.id') STORED NOT NULL,  
    name VARCHAR(256) GENERATED ALWAYS AS (json ->> '$.name') NOT NULL,    
    json JSON NOT NULL,                              -- ← POJO 序列化后的整份文档  
    updatedAt BIGINT UNSIGNED GENERATED ALWAYS AS (json ->> '$.updatedAt') NOT NULL,    
    updatedBy VARCHAR(256) GENERATED ALWAYS AS (json ->> '$.updatedBy') NOT NULL,    
    deleted BOOLEAN GENERATED ALWAYS AS (json -> '$.deleted'),    
    PRIMARY KEY (id), UNIQUE (name)
    );  
-- 1.1.0 迁移（native/1.1.0/mysql/schemaChanges.sql:14）：  
--   DROP KEY `name`, ADD COLUMN nameHash VARCHAR(256) NOT NULL   ← 唯一手写哈希列  
```  
  
### 1.3 存储方式  
  
| 内容 | 落到哪 | 说明 |  
|---|---|---|  
| 实体本体（POJO 剥除 `reviewers` 后的 JSON） | `glossary_entity.json` 列 | DAO 只写 `(nameHash, json)` 两列（`EntityDAO:134-146`）；`nameHash` = `MD5(name)`（`@BindFQN` → `FullyQualifiedName.buildHash` → `EntityUtil.hash:1001`） |  
| `id/name/updatedAt/updatedBy/deleted` | 生成列 | MySQL 按 `AS (json ->> '$.xxx')` 自动求值，Java 不写 |  
| `reviewers/owners/tags/domains/...` | **不进 json** | 剥离后：reviewers/owners → `entity_relationship` 边；tags → `tag_usage` |  
| glossary→术语 的层级边 | `entity_relationship` | **由术语侧写入**（`GlossaryTermRepository:1598/1604`），Glossary 侧 `storeRelationships` 是 no-op |  
| 版本历史 | `entity_extension`（`glossary.version.<v>`） | 更新时由 `EntityUpdater` 写 |  
| 检索索引 | `glossary_search_index`（别名 `glossary`，child: `glossaryTerm`） | reindex 从 MySQL 物化 |  
| 向量嵌入 | `embedding` + `dataAssetEmbeddings` | `AvailableEntityTypes` 白名单含 `glossary` |  
  
### 1.4 数据样例（`glossary_entity.json` 实际存的内容）  
  
```json  
{  
  "id": "a1b2c3d4-0001-4000-8000-000000000001",  
  "name": "BusinessGlossary",  
  "fullyQualifiedName": "BusinessGlossary",  
  "displayName": "Business Glossary",  
  "description": "Enterprise-wide business terminology.",  
  "version": 0.3,  
  "updatedAt": 1760000000000,  
  "updatedBy": "anna.admin",  
  "termCount": 12,  
  "usageCount": 47,  
  "tags": [],  
  "mutuallyExclusive": false,  
  "provider": "user",  
  "deleted": false,  
  "entityStatus": "Approved",  
  "domains": [{ "type": "domain", "name": "Sales" }],  
  "namespaces": [{ "prefix": "biz", "namespace": "http://example.com/ontology/biz#" }]  
}  
```  
  
### 1.5 写入代码摘要  
  
```  
POST /v1/glossaries  →  GlossaryResource.java:83（@Path("/v1/glossaries")，@Collection order=6）  
  → EntityResource.create():560        限流 + authorize(CREATE)  → EntityRepository.create():2952 → createInternal():2966      prepareInternal():2994  →  GlossaryRepository.prepare():173   // no-op（顶层容器无父校验）  
                                 setFullyQualifiedName              // FQN = name      createNewEntityFlush():5115  ——单事务——  
        storeEntity():181 → store() → serializeForStorage():5419             storageJsonNode():5409：POJO → JSON，剥离 reviewers/owners/tags/...             dao.insert("glossary_entity", "nameHash", fqn, json)   // 只写 2 列  
        storeRelationshipsInternal():3009             storeOwners / applyTags(tag_usage) / storeDomains / ...             storeRelationships():191  // Glossary 侧 no-op      postCreate():3959  生命周期事件、RDF、缓存失效  
```  
  
要点：Glossary 自身**不写任何层级边**——`[glossary] --CONTAINS--> [根术语]` 这条边由每个术语在创建时反向写入（见 §2.6）。  
  
---  
  
## 2. GlossaryTerm —— 术语  
  
### 2.1 Java POJO（生成类）  
  
**文件**：`.../entity/data/GlossaryTerm.java`（1466 行，`public class GlossaryTerm implements EntityInterface` 在 `:76`）  
  
| 字段（Java）                                                            | 类型                          | JSON 名                | 行号               | 说明                                       | 保留    |
| ------------------------------------------------------------------- | --------------------------- | --------------------- | ---------------- | ---------------------------------------- | ----- |
| `id`                                                                | `UUID`                      | `id`                  |                  |                                          | ✅     |
| `name` / `displayName` / `description`                              | `String`                    | 同名                    |                  | name保留，其他不要，数据库可通内容冗余存储                  | ✅     |
| ~~`style`~~                                                         | ~~`Style`~~                 | ~~`style`~~           |                  | ~~图标色~~                                  | ~~❌~~ |
| `fullyQualifiedName`                                                | `String`                    | `fullyQualifiedName`  |                  | `glossary[.parent...].name`              | ✅     |
| ~~`iri`~~                                                           | ~~`URI`~~                   | ~~`iri`~~             |                  | ~~外部本体的规范 IRI~~                          | ~~❌~~ |
| `glossary`                                                          | `EntityReference`           | `glossary`            |                  | **剥离** → 边                               | ✅     |
| `parent`                                                            | `EntityReference`           | `parent`              |                  | **剥离** → 边   都写null即可                    | ✅     |
| `children`                                                          | `List<EntityReference>`     | `children`            |                  | 读时派生，**剥离**   都写null即可                   | ✅     |
| `synonyms`                                                          | `List<String>`              | `synonyms`            | `:145`           | 默认 `[]`；**留在 json 列**                    | ✅     |
| `relatedTerms`                                                      | `List<TermRelation>`        | `relatedTerms`        | `:185`           | 默认 `[]`；**剥离** → RELATED_TO 边            | ✅     |
| ~~`references`~~                                                    | ~~`List<TermReference>`~~   | ~~`references`~~      | ~~`:193`~~       | ~~默认 `[]`；外链，**留在 json 列**~~             | ~~❌~~ |
| ~~`conceptMappings`~~                                               | ~~`List<ConceptMapping>`~~  | ~~`conceptMappings`~~ | ~~`:201`~~       | ~~默认 `[]`；**留在 json 列**~~                | ~~❌~~ |
| `version`                                                           | `Double`                    | `version`             | `:208`           | 默认 `0.1D`                                | ✅     |
| ==`reviewers` / `owners`==                                          | ==`List<EntityReference>`== | ==同名==                | ==`:246/:256`==  | ==**owners保留**==                         | ==✅== |
| `tags`                                                              | `List<TagLabel>`            | `tags`                | `:271`           | 默认 `[]`；**剥离** → `tag_usage`（挂在术语 FQN 上） | ==✅== |
| ~~`usageCount`~~                                                    | ~~`Integer`~~               | ~~`usageCount`~~      |                  | ~~派生~~                                   | ~~❌~~ |
| `entityStatus`                                                      | `EntityStatus`              | `entityStatus`        | `:296`           | 默认 `"Unprocessed"`；仅 Approved 进 AI 上下文   | ==✅== |
| `deleted` / `disabled` / `mutuallyExclusive`                        | `Boolean`                   | 同名                    | `:303/:310/:324` | 子术语互斥                                    | ==✅== |
| `domains` / `dataProducts` / `dataContract` / `votes` / `extension` |                             | 同名                    |                  | 剥离/扩展                                    | ==✅== |
| ~~`childrenCount`~~                                                 | ~~`Integer`~~               | ~~`childrenCount`~~   |                  | ~~派生~~                                   | ~~❌~~ |
|                                                                     |                             |                       |                  |                                          |       |
  
**`TermRelation`（`type/TermRelation.java:26`，141 行）**：  
  
```java  
public class TermRelation {  
  @JsonProperty("relationType")  
  @Pattern(regexp = "^[a-zA-Z][a-zA-Z0-9]*$")  private String relationType = "relatedTo";   // :35  字符串，默认 relatedTo，必须在注册表内  
  
  @JsonProperty("term") @NotNull  
  private EntityReference term;                // :47  目标术语引用（Required）  
}  
```  
  
### 2.2 数据库 DDL  
  
```sql  
-- v001:411-425  
CREATE TABLE glossary_term_entity (  
    id VARCHAR(36) GENERATED ALWAYS AS (json ->> '$.id') STORED NOT NULL,  
    fullyQualifiedName VARCHAR(256) GENERATED ALWAYS AS (json ->> '$.fullyQualifiedName') NOT NULL,    json JSON NOT NULL,    updatedAt BIGINT UNSIGNED GENERATED ALWAYS AS (json ->> '$.updatedAt') NOT NULL,    updatedBy VARCHAR(256) GENERATED ALWAYS AS (json ->> '$.updatedBy') NOT NULL,    deleted BOOLEAN GENERATED ALWAYS AS (json -> '$.deleted'),    PRIMARY KEY (id), UNIQUE (fullyQualifiedName));  
-- 1.1.0（native/1.1.0/mysql/schemaChanges.sql:15）：  
--   DROP COLUMN fullyQualifiedName,  
--   ADD COLUMN fqnHash VARCHAR(768) NOT NULL,                                ← 手写（FQN 逐段 MD5）  
--   ADD COLUMN name VARCHAR(256) GENERATED ALWAYS AS (json ->> '$.name') NOT NULL;  
```  
  
### 2.3 存储方式  
  
| 内容            | 落到哪                                                                                   | 说明                                                                                                                                                                                                                                                                                         |     |
| ------------- | ------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | --- |
| 实体本体          | `glossary_term_entity.json` 列                                                         | 写库前剥除 `glossary/parent/reviewers/relatedTerms`（`GlossaryTermRepository:618-621`）+ 通用剥离表（`EntityRepository:5391-5403`：href/owners/children/tags/domains/dataProducts/dataContract/followers/experts/reviewers/certification）。**json 里保留**：synonyms/references/conceptMappings/style/iri/... |     |
| 查找键           | `fqnHash`（手写）+ `name`/`id`（生成列）                                                       | `getNameHashColumn() = "fqnHash"`（`CollectionDAO:5957-5960`）                                                                                                                                                                                                                               |     |
| glossary→根术语  | 边 `CONTAINS(0)`                                                                       | `addGlossaryRelationship:1598`                                                                                                                                                                                                                                                             |     |
| glossary→非根术语 | 边 `HAS(10)`（回指）                                                                       | 同上                                                                                                                                                                                                                                                                                         |     |
| 父→子术语         | 边 `CONTAINS(0)`                                                                       | `addParentRelationship:1604`                                                                                                                                                                                                                                                               |     |
| 术语↔术语类型化关系    | 边 `RELATED_TO(15)` + `relationType` 列                                                 | `json = {"relationType":"broader"}`；from/to 字典序归一，单行双向；1.13+ relationType 进 PK                                                                                                                                                                                                             |     |
| 术语自己的 tags    | `tag_usage`（source=0/1，target=**术语 FQN**）                                             | `applyTags:5965` → `tagUsageDAO.applyTag`                                                                                                                                                                                                                                                  |     |
| 术语挂到表/列       | `tag_usage`（source=1，target=**表/列 FQN**）                                              | 资产侧 `/{id}/assets/add` 操作；列级派生 `DERIVED` 读时计算                                                                                                                                                                                                                                              |     |
| 检索索引          | `glossary_term_search_index`（别名 `glossaryTerm`，parents: `all`/`glossary`/`dataAsset`） | 含父 glossary 的 `mutuallyExclusive` 注入                                                                                                                                                                                                                                                       |     |
| 向量嵌入          | ✅（`AvailableEntityTypes` 含 `glossaryTerm`）                                            | 文本 = 名称+定义+同义词等                                                                                                                                                                                                                                                                            |     |
  
### 2.4 数据样例（`glossary_term_entity.json` 实际存的内容）  
  
```json  
{  
  "id": "a1b2c3d4-0002-4000-8000-000000000002",  
  "name": "PhoneNumber",  
  "displayName": "Phone Number",  
  "description": "A telephone number at which a person may be contacted.",  
  "fullyQualifiedName": "BusinessGlossary.User.PhoneNumber",  
  "iri": "http://example.com/ontology/biz#PhoneNumber",  
  "style": { "iconColor": "#FF4D4F" },  
  "synonyms": ["Tel", "ContactNumber"],  
  "references": [{ "name": "GDPR Article 4(1)", "endpoint": "https://gdpr-info.eu/art-4-gdpr/" }],  
  "conceptMappings": [],  
  "usageCount": 31, "childrenCount": 2,  
  "version": 1.2, "updatedAt": 1760000100000, "updatedBy": "anna.admin",  
  "entityStatus": "Approved", "mutuallyExclusive": false,  
  "provider": "user", "deleted": false  
}  
```  
  
注意：**json 里没有** `glossary/parent/relatedTerms/reviewers/tags` —— 请求进来时有，但落库序列化前被剥离，分别去往 `entity_relationship` 与 `tag_usage`。  
  
### 2.5 写入代码摘要（完整链路）  
  
```  
POST /v1/glossaryTerms   GlossaryTermResource.java:711-732（@Collection order=7）  
  │  @Valid CreateGlossaryTerm  ← Jackson：请求体 JSON → CreateGlossaryTerm POJO  │  GlossaryTermMapper.createToEntity():16-31  │     CreateGlossaryTerm → GlossaryTerm POJO；relatedTerms 默认补 relationType="relatedTo"  ▼EntityResource.create():560 → repository.create():2957 → createInternal():2966  
  │  ├─ prepareInternal():2994  │    ├─ validateTags(entity)  │    ├─ GlossaryTermRepository.prepare():488-512  │    │     解析 parent（Entity.getEntity）、解析 glossary、validateHierarchy（防环）  
  │    │     populateTermRelations(relatedTerms)；checkDuplicateTerms（新建时查重）  
  │    ├─ setFullyQualifiedName():1070-1081   根: glossary.fqn + "." + name  
  │    │                                    子: parent.fqn + "." + name  
  │    └─ validateExtension / setDefaultStatus  │    entity.setUpdatedBy(user) / setImpersonatedBy(...)  │  └─ createNewEntity():5101 → createNewEntityFlush():5115 ——【单事务 flushInOneTransaction】  
       │       ├─ A. storeEntity():623 → store():5432       │      json = serializeForStorage():5419       │        = storageJsonNode():5409(POJO→JSON树，剥离关系字段) + sanitizeNul  
       │      dao.insert("glossary_term_entity", "fqnHash", fqn, json)       │        @BindFQN → MD5 分段哈希；@BindJson → 上面的 json 字符串  
       │      （MySQL 侧：生成列 id/name/fqn 按 AS 表达式从 json 自动求值）  
       │       └─ B. storeRelationshipsInternal():3009              storeOwners / applyTags():5965（术语自身 tags → tag_usage）  
              storeDomains / storeDataProducts / storeReviewers / applyCertification              storeRelationships():644                addGlossaryRelationship():1598   根→GLOSSARY:CONTAINS(0) / 非根→GLOSSARY:HAS(10)  
                addParentRelationship():1604     父→子: CONTAINS(0)  
                relatedTerms 循环: RELATED_TO(15)，json={"relationType":...}，  
                                   relationType 必须在 glossaryTermRelationSettings 注册，  
                                   computeCanonicalRelationType 字典序归一，双向单行  
```  
  
### 2.6 关系写入源码（核心片段）  
  
```java  
// GlossaryTermRepository.java:644-647  
public void storeRelationships(GlossaryTerm entity) {  
  addGlossaryRelationship(entity);  
  addParentRelationship(entity);  
  ...  // relatedTerms → RELATED_TO(15)，见 §6}  
  
// :1598 —— 根术语用 CONTAINS，非根用 HAS（回指）  
private void addGlossaryRelationship(GlossaryTerm term) {  
  Relationship relationship = term.getParent() != null ? Relationship.HAS : Relationship.CONTAINS;  
  addRelationship(term.getGlossary().getId(), term.getId(), GLOSSARY, GLOSSARY_TERM, relationship);  
}  
  
// :1604  
private void addParentRelationship(GlossaryTerm term) {  
  if (term.getParent() != null) {  
    addRelationship(term.getParent().getId(), term.getId(), GLOSSARY_TERM, GLOSSARY_TERM, Relationship.CONTAINS);  
  }}  
```  
  
---  
  
## 3. Classification —— 标签分类  
  
### 3.1 Java POJO（生成类）  
  
**文件**：`.../entity/classification/Classification.java`（942 行，`public class Classification implements EntityInterface` 在 `:57`）  
  
| 字段 | 类型 | JSON 名 | 默认值/说明 |  
|---|---|---|---|  
| `id` / `name` / `fullyQualifiedName` / `displayName` / `description` | `UUID`/`String` | 同名 | FQN = name |  
| `termCount` / `usageCount` | `Integer` | 同名 | 派生（tag 数/挂载数） |  
| `version` | `Double` | `version` | 默认 `0.1D` |  
| `updatedAt` / `updatedBy` / `impersonatedBy` / `href` | | 同名 | |  
| `deleted` / `disabled` | `Boolean` | 同名 | 默认 `false`；系统分类不可删用 disabled |  
| `mutuallyExclusive` | `Boolean` | `mutuallyExclusive` | 默认 `false`；true=分级（tier 只能选一），false=分类（可多贴） |  
| `domains` / `owners` / `reviewers` | `List<EntityReference>` | 同名 | 剥离 → 边 |  
| `autoClassificationConfig` | `AutoClassificationConfig` | `autoClassificationConfig` | 自动识别配置（enabled/minimumConfidence/...） |  
| `provider` / `entityStatus` | 枚举 | 同名 | 默认 `"user"` / `"Unprocessed"` |  
| 变更族 | `ChangeDescription` ×2 | | |  
  
### 3.2 数据库 DDL  
  
```sql  
-- v001:350-365（当时表名 tag_category）  
CREATE TABLE tag_category (  
    id VARCHAR(36) GENERATED ALWAYS AS (json ->> '$.id') STORED NOT NULL,  
    name VARCHAR(256) GENERATED ALWAYS AS (json ->> '$.name') NOT NULL,    json JSON NOT NULL,   -- JSON stores category information and does not store children    updatedAt BIGINT UNSIGNED GENERATED ALWAYS AS (json ->> '$.updatedAt') NOT NULL,    updatedBy VARCHAR(256) GENERATED ALWAYS AS (json ->> '$.updatedBy') NOT NULL,    deleted BOOLEAN GENERATED ALWAYS AS (json -> '$.deleted'),    UNIQUE (name));  
-- v008:1  RENAME TABLE tag_category TO classification;  
-- 1.1.0:4  DROP KEY `name`, ADD COLUMN nameHash VARCHAR(256) NOT NULL  
```  
  
### 3.3 存储方式  
  
| 内容 | 落到哪 | 说明 |  
|---|---|---|  
| 实体本体 | `classification.json` 列 | 剥离 owners/reviewers/domains 等通用表字段 |  
| 查找键 | `nameHash`（手写）+ `id/name`（生成列） | `getNameHashColumn() = "nameHash"` |  
| classification→tag | `entity_relationship` `CONTAINS(0)` | **由 Tag 侧写入**（`TagRepository:986`）；Classification 侧 `storeRelationships:255` no-op |  
| 子分类 | 不支持（分类无嵌套，嵌套发生在 Tag） | |  
| 检索索引 | `classification_search_index`（child: `tag`） | |  
| 向量嵌入 | ❌ 不嵌入 | `AvailableEntityTypes` 不含 classification |  
  
### 3.4 数据样例（`classification.json`）  
  
```json  
{  
  "id": "a1b2c3d4-0010-4000-8000-000000000010",  
  "name": "PII", "fullyQualifiedName": "PII",  
  "displayName": "PII Classification",  
  "description": "Personally Identifiable Information classification tags.",  
  "termCount": 3, "usageCount": 128,  
  "mutuallyExclusive": true,  
  "autoClassificationConfig": {  
    "enabled": true, "conflictResolution": "highest_confidence",  
    "minimumConfidence": 0.6, "requireExplicitMatch": true  
  },  "version": 0.1, "updatedAt": 1760000200000, "updatedBy": "system",  
  "provider": "system", "deleted": false, "entityStatus": "Approved"  
}  
```  
  
### 3.5 写入代码摘要  
  
```  
POST /v1/classifications  ClassificationResource.java:81（@Collection order=4，先于 Tag/Glossary 初始化）  
  → EntityResource.create():560 → createInternal():2966      ClassificationRepository.prepare():245   // no-op      storeEntity():250 → store() → dao.insert("classification", "nameHash", ...)      storeRelationshipsInternal():3009        storeOwners / storeDomains / storeReviewers ...        storeRelationships():255   // no-op —— 分类→tag 的边由每个 Tag 创建时写入  
```  
  
---  
  
## 4. Tag —— 标签  
  
### 4.1 Java POJO（生成类）  
  
**文件**：`.../entity/classification/Tag.java`（1270 行，`public class Tag implements EntityInterface` 在 `:69`）  
  
| 字段 | 类型 | JSON 名 | 行号 | 说明 |  
|---|---|---|---|---|  
| `id` / `name` / `displayName` / `description` | | 同名 | | |  
| `fullyQualifiedName` | `String` | | | `Classification.tag[.childTag]` |  
| `style` | `Style` | `style` | | |  
| `classification` | `EntityReference` | `classification` | | **剥离** → 边 |  
| `parent` | `EntityReference` | `parent` | | 父 tag；**剥离** → 边 |  
| `children` | `List<EntityReference>` | `children` | `:152` | 派生，剥离 |  
| `version` | `Double` | `version` | `:159` | 默认 `0.1D` |  
| `usageCount` | `Integer` | | | 派生 |  
| `deprecated` | `Boolean` | `deprecated` | `:201` | 默认 `false` |  
| `deleted` / `disabled` / `mutuallyExclusive` | `Boolean` | 同名 | `:208/:238/:245` | |  
| `domains` / `dataProducts` / `dataContract` / `owners` / `reviewers` | | 同名 | | 剥离 |  
| `recognizers` | `List<Recognizer>` | `recognizers` | `:312` | 默认 `[]`；自动识别规则（**留在 json 列**） |  
| `autoClassificationEnabled` | `Boolean` | 同名 | `:319` | 默认 `false` |  
| `autoClassificationPriority` | `Integer` | 同名 | `:328` | 默认 `50`（冲突仲裁优先级） |  
| `provider` / `entityStatus` | 枚举 | 同名 | | |  
  
### 4.2 数据库 DDL  
  
```sql  
-- v001:360-374  
CREATE TABLE tag (  
    id VARCHAR(36) GENERATED ALWAYS AS (json ->> '$.id') STORED NOT NULL,  
    fullyQualifiedName VARCHAR(256) GENERATED ALWAYS AS (json ->> '$.fullyQualifiedName') NOT NULL,    json JSON NOT NULL,   -- JSON stores all tag attributes and does not store children    updatedAt BIGINT UNSIGNED GENERATED ALWAYS AS (json ->> '$.updatedAt') NOT NULL,    updatedBy VARCHAR(256) GENERATED ALWAYS AS (json ->> '$.updatedBy') NOT NULL,    deleted BOOLEAN GENERATED ALWAYS AS (json -> '$.deleted'),    UNIQUE (fullyQualifiedName));  
-- 1.1.0:31  DROP COLUMN fullyQualifiedName, ADD fqnHash VARCHAR(768) NOT NULL,  
--            ADD name VARCHAR(256) GENERATED ALWAYS AS (json ->> '$.name') NOT NULL  
```  
  
### 4.3 存储方式  
  
| 内容 | 落到哪 | 说明 |  
|---|---|---|  
| 实体本体 | `tag.json` 列 | 剥离 classification/parent/children/owners/...（通用表） |  
| 查找键 | `fqnHash` + `name/id` 生成列 | `getNameHashColumn() = "fqnHash"`（`CollectionDAO:6770`） |  
| classification→tag | 边 `CONTAINS(0)` | `addClassificationRelationship:986`（**根/子统一用 CONTAINS**，无 HAS 回指——与术语不同） |  
| 父 tag→子 tag | 边 `CONTAINS(0)` | `addParentRelationship:995` |  
| tag→表/列 | `tag_usage`（source=0，target=表/列 FQN） | `/{id}/assets/add`；列级贴标是 tag_usage 的主要用途 |  
| 检索索引 | `tag_search_index`（parents: `classification`/`all`/`dataAsset`） | |  
| 向量嵌入 | ❌ | |  
  
### 4.4 数据样例（`tag.json`）  
  
```json  
{  
  "id": "a1b2c3d4-0011-4000-8000-000000000011",  
  "name": "Sensitive", "displayName": "Sensitive",  
  "fullyQualifiedName": "PII.Sensitive",  
  "description": "Data that must be protected under privacy regulations.",  
  "style": { "iconColor": "#FAAD14" },  
  "usageCount": 96, "deprecated": false, "mutuallyExclusive": false,  
  "autoClassificationEnabled": false, "autoClassificationPriority": 50,  
  "recognizers": [],  
  "version": 0.4, "updatedAt": 1760000300000, "updatedBy": "anna.admin",  
  "provider": "user", "deleted": false  
}  
```  
  
### 4.5 写入代码摘要  
  
```  
POST /v1/tags   TagResource.java:97（@Collection order=5）  
  → createInternal():2966      TagRepository.prepare():186            解析 classification/parent 引用、校验  
      setFullyQualifiedName():488            FQN = classification.fqn + "." + name [+ "." + parent.name...]      storeEntity():385 → store() → dao.insert("tag", "fqnHash", fqn, json)      storeRelationshipsInternal():3009 → storeRelationships():482          addClassificationRelationship():986   addRelationship(classification.id, tag.id, CONTAINS)          addParentRelationship():995           父 tag → 子 tag: CONTAINS```  
  
```java  
// TagRepository.java:986-990（根/子统一 CONTAINS）  
private void addClassificationRelationship(Tag term) {  
  addRelationship(term.getClassification().getId(), term.getId(), Entity.CLASSIFICATION, TAG, Relationship.CONTAINS);  
}  
```  
  
---  
  
## 5. Metric —— 指标  
  
### 5.1 Java POJO（生成类）  
  
**文件**：`.../entity/data/Metric.java`（1462 行，`public class Metric implements EntityInterface` 在 `:80`）  
  
| 字段 | 类型 | JSON 名 | 行号 | 说明 |  
|---|---|---|---|---|  
| `id` / `name` / `fullyQualifiedName` / `displayName` / `description` | | 同名 | | FQN = name（扁平，无层级） |  
| `metricExpression` | `MetricExpression` | `metricExpression` | | `{language, code}`；**权威计算定义，留在 json 列** |  
| `metricType` | `MetricType` | 同名 | | 枚举如 `SUM` |  
| `unitOfMeasurement` / `customUnitOfMeasurement` | 枚举/`String` | 同名 | | 自定义单位时用后者 |  
| `granularity` | `MetricGranularity` | 同名 | | 如 `MONTH` |  
| `dimensions` | `List<MetricDimension>` | `dimensions` | `:163` | 语义层维度（name/type/expression），**留在 json 列** |  
| `measures` | `List<MetricMeasure>` | `measures` | `:171` | 度量（name/aggregation） |  
| `filters` | `List<MetricFilter>` | `filters` | `:179` | 过滤器（where） |  
| `relatedMetrics` | `List<EntityReference>` | `relatedMetrics` | `:189` | **剥离** → RELATED_TO(15) 边（双向） |  
| `assets` | `List<EntityReference>` | `assets` | `:199` | **剥离** → APPLIED_TO(14) 边（metric→表/列/看板） |  
| `version` | `Double` | `version` | `:206` | 默认 `0.1D` |  
| `updatedAt` / `updatedBy` / `impersonatedBy` / `href` | | 同名 | | |  
| `owners` / `reviewers` / `followers` | `List<EntityReference>` | 同名 | `:244/:254/:264` | 剥离 |  
| `tags` | `List<TagLabel>` | `tags` | `:272` | 默认 `[]`；剥离 → `tag_usage`（**metric↔glossary 的唯一纽带**） |  
| `deleted` | `Boolean` | `deleted` | `:295` | 默认 `false` |  
| `domains` / `dataProducts` / `dataContract` / `votes` / `extension` | | 同名 | | |  
| `entityStatus` / `provider` | 枚举 | 同名 | `:361/:368` | |  
| `certification` | `AssetCertification` | `certification` | | 认证徽章 |  
| `derivedFrom` | `EntityReference` | `derivedFrom` | | 派生来源引用 |  
  
> 2.0.5 的 Metric **没有** `parent/children/metricGroup/experts` 字段（指标层级属于后续版本）。  
  
### 5.2 数据库 DDL  
  
```sql  
-- v001:147-161  
CREATE TABLE metric_entity (  
    id VARCHAR(36) GENERATED ALWAYS AS (json ->> '$.id') STORED NOT NULL,  
    fullyQualifiedName VARCHAR(256) GENERATED ALWAYS AS (json ->> '$.fullyQualifiedName') NOT NULL,    json JSON NOT NULL,    updatedAt BIGINT UNSIGNED GENERATED ALWAYS AS (json ->> '$.updatedAt') NOT NULL,    updatedBy VARCHAR(256) GENERATED ALWAYS AS (json ->> '$.updatedBy') NOT NULL,    deleted BOOLEAN GENERATED ALWAYS AS (json -> '$.deleted'),    PRIMARY KEY (id), UNIQUE (fullyQualifiedName));  
-- 1.1.0:20  DROP COLUMN fullyQualifiedName, ADD fqnHash VARCHAR(768) NOT NULL,  
--            ADD name VARCHAR(256) GENERATED ALWAYS AS (json ->> '$.name') NOT NULL  
```  
  
### 5.3 存储方式  
  
| 内容 | 落到哪 | 说明 |  
|---|---|---|  
| 实体本体 | `metric_entity.json` 列 | metricExpression/dimensions/measures/filters 等语义层内容**全部在 json 列**；剥离 owners/reviewers/followers/tags/relatedMetrics/assets/certification 等 |  
| 查找键 | `fqnHash` + `name/id` 生成列 | `getNameHashColumn() = "fqnHash"`（`CollectionDAO:5884`） |  
| metric→资产 | 边 `APPLIED_TO(14)` | `storeRelationships:214` |  
| metric↔metric | 边 `RELATED_TO(15)` | relatedMetrics，双向单行 |  
| metric→glossary | **无直连边**，仅经 `tag_usage`（tags 贴术语） | |  
| 版本历史 | `entity_extension` | |  
| 检索索引 | `metric_search_index`（别名 `metric`，parents: `all`/`dataAsset`/`dataAssetEmbeddings`） | `MetricIndex` 把 dimension/measure FQN 折入 `fqnParts` |  
| 向量嵌入 | ✅ | chunk 文档携带 `metricExpression/metricType/granularity/unitOfMeasurement` 四元组（`VectorDocBuilder:379-406`），用于区分同名指标 |  
  
### 5.4 数据样例（`metric_entity.json` 实际存的内容）  
  
```json  
{  
  "id": "a1b2c3d4-0020-4000-8000-000000000020",  
  "name": "monthly_recurring_revenue",  
  "fullyQualifiedName": "monthly_recurring_revenue",  
  "displayName": "Monthly Recurring Revenue",  
  "description": "Normalized recurring revenue billed in a month.",  
  "metricExpression": { "language": "SQL", "code": "SELECT SUM(amount) FROM billing.subscriptions WHERE status = 'active'" },  
  "metricType": "SUM",  
  "unitOfMeasurement": "DOLLARS",  
  "granularity": "MONTH",  
  "dimensions": [  
    { "name": "region", "type": "CATEGORICAL", "description": "Sales region", "expression": "customer_region" },  
    { "name": "billing_month", "type": "TIME", "expression": "date_trunc('month', billed_at)" }  
  ],  "measures": [{ "name": "mrr_amount", "aggregation": "SUM(amount)" }],  
  "filters": [{ "where": "status = 'active' AND deleted = false" }],  
  "certification": { "certificationStatus": "Gold" },  
  "version": 2.1, "updatedAt": 1760000400000, "updatedBy": "bob.analytics",  
  "entityStatus": "Approved", "provider": "user", "deleted": false  
}  
```  
  
注意：json 里**没有** `assets/relatedMetrics/tags/owners` —— 分别去往 `APPLIED_TO` 边、`RELATED_TO` 边、`tag_usage`、owner 边。  
  
### 5.5 写入代码摘要  
  
```  
POST /v1/metrics   MetricResource.java:81（@Collection name="metrics"，无 order）  
  → createInternal():2966      MetricRepository.prepare():127-135          validateRelatedTerms(relatedMetrics)、validateCustomUnitOfMeasurement  
          metric.setAssets(EntityUtil.populateEntityReferences(assets))      setFullyQualifiedName():100-103     FQN = name；并给 dimensions/measures 算子 FQN                                          （<metric>.dimension.<name> / <metric>.measure.<name>）  
      storeEntity():187 → store() → dao.insert("metric_entity", "fqnHash", fqn, json)      storeRelationshipsInternal():3009 → storeRelationships():214          for relatedMetrics: addRelationship(id, relatedId, METRIC, METRIC, RELATED_TO, true)  // 双向  
          for assets:         addRelationship(id, assetId, METRIC, asset.getType(), APPLIED_TO)          tags → tag_usage（applyTags）  
```  
  
---  
  
## 6. 语义资源之间的联系（跨资源总览）  
  
### 6.1 `entity_relationship` 边（层级 + 语义关系的权威存储）  
  
```sql  
-- v001:4-18；1.13+ 增加 relationType varchar(64)，PK 变为 (fromId, toId, relation, relationType)fromId, toId, fromEntity, toEntity, relation(TINYINT), jsonSchema, json, deleted[, relationType]  
```  
  
| # | Relationship 枚举 | 语义资源中的用途 |  
|---|---|---|  
| 0 | `CONTAINS` | glossary→根术语、父术语→子术语、classification→tag、父 tag→子 tag |  
| 10 | `HAS` | glossary→非根术语（回指） |  
| 14 | `APPLIED_TO` | metric→资产 |  
| 15 | `RELATED_TO` | 术语↔术语类型化关系（relationType 字符串）、metric↔metric |  
  
### 6.2 总架构图  
  
```  
══════════ entity_relationship（层级 + 语义关系，权威存储）══════════  
  
 [glossary] ──CONTAINS(0)──▶ [term 根]        [classification] ──CONTAINS(0)──▶ [tag 根]  
      │ HAS(10) 回指                              │ CONTAINS(0)（根/子统一）         │ CONTAINS(0)      ▼                                          ▼                                 ▼ [glossary] ──HAS(10)──▶ [term 非根]          [tag 子] ──CONTAINS(0)──▶ [tag 孙]  
 写入：GlossaryTermRepository:1598/1604       写入：TagRepository:986/995  
  
 [termA] ══RELATED_TO(15, relationType)══▶ [termB]    relationType ∈ relatedTo|synonym|antonym|broader|narrower|partOf|hasPart|                   calculatedFrom|usedToCalculate|seeAlso    注册表 = glossaryTermRelationSettings（settings JSON）；字典序归一，双向单行  
    读取逆关系换算：GlossaryTermRepository:437-443  
  
 [metric A] ──RELATED_TO(15)──▶ [metric B] [metric]   ──APPLIED_TO(14)──▶ [table|column|dashboard|...]     写入：MetricRepository:214  
  
══════════ tag_usage（挂载：语义资源 → 数据资产；★ 不是边）══════════  
  
 [tag]         ──(source=0 Classification)──▶ [table / table.column] [glossaryTerm]──(source=1 Glossary)───────▶ [table / table.column] [glossaryTerm 自身的 tags] ──(挂在术语 FQN)──▶ 读取时派生 DERIVED 到列（不落库）  
 [metric 的 tags]           ──(挂在 metric FQN)──▶ metric↔glossary 的唯一纽带  
```  
  
### 6.3 权威性对照  
  
| 概念 | 权威存储 | JSON/POJO 字段角色 |  
|---|---|---|  
| glossary → 术语 | 边 `CONTAINS/HAS` | `glossary` 引用（写入前剥离） |  
| 父术语 → 子术语 | 边 `CONTAINS(0)` | `parent/children` 剥离/派生 |  
| classification → tag / 父 tag → 子 tag | 边 `CONTAINS(0)` | `classification/parent` 剥离 |  
| 术语间类型化关系 | 边 `RELATED_TO(15)` + `relationType` 列 | `relatedTerms: List<TermRelation>` 剥离，读时回填 |  
| `synonyms`/`references`/`conceptMappings` | **纯 json 列** | 权威 |  
| `metricExpression`/`dimensions`/`measures`/`filters` | **纯 json 列** | 权威 |  
| 术语/标签贴资产 | **`tag_usage`** | `tags: List<TagLabel>` 是读视图 |  
| metric→资产 / metric↔metric | 边 `APPLIED_TO` / `RELATED_TO` | `assets`/`relatedMetrics` 剥离，读时填充 |  
  
---  
  
## 7. 存储去向总览（RDB / 索引 / 向量）  
  
| 语义资源 | RDB 权威表 | 手写哈希列 | ES/OS 索引（别名） | 向量嵌入 |  
|---|---|---|---|---|  
| glossary | `glossary_entity` | `nameHash` | `glossary_search_index`（`glossary`） | ✅ |  
| glossaryTerm | `glossary_term_entity` | `fqnHash` | `glossary_term_search_index`（`glossaryTerm`） | ✅ |  
| classification | `classification` | `nameHash` | `classification_search_index`（`classification`） | ❌ |  
| tag | `tag` | `fqnHash` | `tag_search_index`（`tag`） | ❌ |  
| metric | `metric_entity` | `fqnHash` | `metric_search_index`（`metric`） | ✅ |  
  
公共投影：`entity_extension`（版本历史）、`change_event`（webhook/索引增量）、ES 别名 `all`/`dataAsset`（聚合检索）。  
写入端只写 MySQL（实体行 + 边 + tag_usage）；索引与向量由 reindex 工作流从 MySQL 物化，查询 API（`/v1/search/query` 等）只读 OpenSearch。  
  
---  
  
## 8. MCP 自然语言检索链路（摘要）  
  
`openmetadata-mcp`（15 个声明式工具，`json/data/mcp/tools.json`）：  
  
```  
LLM 客户端 → /api/v1/mcp/*（McpServer.java:167 servlet，:280 tool handler，JWT 校验）  
  → DefaultToolContext.callToolWithMetadata:63（switch 调度 :78-133）  
  → 进程内：SearchRepository（关键字，OpenSearch）/ VectorIndexService（KNN）  
            / AIContextFinder|Builder（知识层）/ Entity 仓储（RDB）  
```  
  
语义资源参与点：  
  
| 工具 | 参与 |  
|---|---|  
| `search_metadata` | `entityType` 含 5 种；`tags.tagFQN` 查询同时命中分类标签与术语；`resolveIndex:711-714` |  
| `semantic_search` | KNN + `metricType/granularity/unitOfMeasurement` 刻面；命中回传 `metricExpression` 四元组（`:41-57/82`） |  
| `find_context` | `CONTEXT_ENTITY_TYPES=[GLOSSARY_TERM, METRIC, PAGE]`（`AIContextFinder:50-51`）；术语→tag_usage 反查资产，metric→APPLIED_TO 反查；仅 APPROVED（`:165-167`） |  
| `get_entity_details`（`include=context`） | `AIContextBuilder:829`（glossary 术语）/`:947`（APPLIED_TO 挂载指标） |  
| `get_persona_context` | `PersonaContextBuilder:76`（知识类型）/`:509`（findFromBatch APPLIED_TO） |  
| `create_entity` | `metricExpression` 校验（`CreateEntityTool:258`）、tag/classification 派生（`:173-199`） |  
  
---  
  
## 9. 一句话总结  
  
**Schema-first + Document-per-row + Edge-table + 挂载表**：每个语义资源 = 一份 JSON Schema → 一个生成 POJO（编译产物在 `target/generated-sources/jsonschema2pojo/`）→ 一张 `*_entity` 表（DAO 只写 `json` + 哈希列，生成列由 MySQL `AS (json ->> '$.x')` 自动投影）；**层级与语义关系全部外置**：术语/标签树走 `entity_relationship`（`CONTAINS/HAS`），术语类型化关系走 `RELATED_TO(15)+relationType`（字符串，settings 注册表治理），挂载走 `tag_usage`，metric 挂资产走 `APPLIED_TO`；每个资源另有 OpenSearch 索引投影，glossary/glossaryTerm/metric 额外进向量嵌入，MCP 的 15 个工具把这条链暴露给 LLM 做自然语言检索。