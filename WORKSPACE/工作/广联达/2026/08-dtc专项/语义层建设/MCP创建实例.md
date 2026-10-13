# `create_entity` 存储实例解读（POC 真实数据）

> 配套 [MCP创建实体存储结构.md](MCP创建实体存储结构.md)（机制）阅读。
> **数据来源**：GDMP POC 真实库 `openmetadata_liujh`（10.246.150.153:3306，2026-10 查询），全部为实查结果，非构造。
> 查询通道：JDBC 只读（等价 MCP mysql_query 的 SELECT 能力）。

## 1. `mcp_server_entity` —— 真实 5 行（治理 seed）

```sql
SELECT id, name, fqnHash, deleted,
       JSON_EXTRACT(json,'$.version') v,
       JSON_UNQUOTE(JSON_EXTRACT(json,'$.fullyQualifiedName')) fqn,
       JSON_UNQUOTE(JSON_EXTRACT(json,'$.governanceMetadata.registrationStatus')) status
FROM mcp_server_entity;
```

| id | name | fqnHash | v | fqn | status |
|---|---|---|---|---|---|
| 22222222-...-2201 | customer_profile_tools | `264d6455...3179d93f10.e20101afdaa20fb1d8006a814fd0e465` | 0.1 | ai_governance_mcp.customer_profile_tools | **Approved** |
| 22222222-...-2202 | warehouse_sql_query | `264d6455...3179d93f10.dadced9efc3096afe4161b36858da9cc` | 0.1 | ai_governance_mcp.warehouse_sql_query | **PendingApproval** |
| 22222222-...-2203 | security_ticket_triage | `264d6455...3179d93f10.9825b997898929f20ec2978da6cce844` | 0.1 | ai_governance_mcp.security_ticket_triage | Approved |
| 22222222-...-2204 | code_repo_assistant | `264d6455...3179d93f10.4d77b6e0905343a3fc035520f9a6d202` | 0.1 | ai_governance_mcp.code_repo_assistant | Unregistered |
| 22222222-...-2205 | slack_workflow_actions | `264d6455...3179d93f10.8a95cd86af33a63ef6753623726f012c` | 0.1 | ai_governance_mcp.slack_workflow_actions | Registered |

读法：

- `fqnHash` = **FQN 每段哈希用 `.` 连接**（前段 `264d...` = `ai_governance_mcp` 的哈希，后段 = server name 的哈希）——这就是为什么它能当 UNIQUE 键。
- `json` 列即整份实体文档（治理字段 governanceMetadata/dataAccessSummary 全在里面），`_id`/版本/审批状态都从 json 取。
- 5 行 = 启动时 `AIGovernanceAssetSeedLoader` 从 seed 文件插入的治理示例。

## 2. `glossary_entity` / `glossary_term_entity` —— 真实业务数据

`glossary_entity`（5 行，注意：**老式表，没有 fqnHash 列**，`name` 列即哈希列——`GlossaryDAO.getNameHashColumn()="name"`）：

| id | name | deleted | version | description（节选） |
|---|---|---|---|---|
| `3c4083b9...` | 经营管控业务术语库 | 0 | **0.2** | 五冶集团经营管控域业务术语库。沉淀经营预算业务核心概念…… |
| `7f67d4d0...` | 交易业务术语库 | 0 | 0.1 | 存放交易域相关业务名词，用于 AI 问数、术语消歧…… |
| `cdb3cbd8...` | 工程技术库 | 0 | **0.3** | 刘佳辉的术语 |
| `17c79500...` | 合同 | 0 | 0.2 | 合同 |
| `56431c15...` | 新增一个数据库 | 0 | 0.1 | 这个一个数据域的添加操作 |

`glossary_term_entity`（17 行，节选）——FQN 是中文的两段式 `术语库.术语`：

| id | name | fullyQualifiedName | version | description |
|---|---|---|---|---|
| `03c19ea9...` | 订单金额 | 交易业务术语库.订单金额 | 0.2 | 订单下单时记录的金额，包含后续退款订单的金额 |
| `078a8058...` | 产品类型 | 经营管控业务术语库.产品类型 | 0.2 | 营业收入统计分析的产品类型划分维度（二品类）…… |
| `1b2301ca...` | 退款标记 | 交易业务术语库.退款标记 | 0.1 | 标记订单是否退款，true=已退款，false=未退款 |

对应存储文档：INSERT 只写 `name`(哈希列)/`json` 两列（老表）或 `fqnHash`/`json`（新表）。

## 3. `entity_relationship` —— 真实关系边

`ba1bc817-c047-48f6-aa79-898b6a605a28` = glossaryTerm **二级公司**，它的全部边：

| fromId | toId | fromEntity | toEntity | relation | 语义 |
|---|---|---|---|---|---|
| `3c4083b9...`（经营管控业务术语库） | `ba1bc817...`（二级公司） | glossary | glossaryTerm | **0 = CONTAINS** | 术语库包含术语（父子层级） |
| `da5805ee...`（经营管控域） | `ba1bc817...`（二级公司） | domain | glossaryTerm | **10 = HAS** | 术语挂到业务域 |

交易业务术语库（`7f67d4d0...`）的边：

| relation | from | to | 语义 |
|---|---|---|---|
| 10 = HAS | domain（NULL 名，另一域） | 交易业务术语库 | 资产挂域 |
| 0 = CONTAINS ×2 | 交易业务术语库 | 两个 glossaryTerm | 库包含术语 |
| 8 = OWNS | user | 交易业务术语库 | owner 关系 |

**relation 枚举序号对照**（`type/entityRelationship.json` javaEnums 顺序）：`CONTAINS=0, OWNS=8, PARENT_OF=9, HAS=10, FOLLOWS=11`。
**方向事实**：父子层级是 parent→child 用 CONTAINS（glossary→term、service→db→schema→table 同款）；domain→资产用 HAS（`DomainRepository.java:291` `Relationship.HAS`）；domain→domain 父子才是 CONTAINS（`DomainRepository.java:254`）。

## 4. `tag_usage` —— 真实结构（12 列哈希版，与旧迁移脚本不同）

真实列（`information_schema.columns` 实查）：

| 列 | 类型 | 说明 |
|---|---|---|
| source | tinyint | 0=Classification, 1=Glossary |
| tagFQN | varchar(512) | 标签 FQN 原文 |
| labelType | tinyint | 0=Manual, 1=Propagated, 2=Automated, 3=Derived, 4=Generated |
| state | tinyint | 0=Suggested, 1=Confirmed |
| tagFQNHash / targetFQNHash | varchar(768) | **目标 FQN 只存哈希**（多段 `.` 连接） |
| targetfqnhash_lower / tagfqn_lower | varchar(768) | 小写列，加速大小写无关查询 |
| reason | text | 打标原因（自动打标工具填写） |
| appliedAt / appliedBy | timestamp / varchar | 应用时间/人 |
| metadata | json | 附加元数据 |

真实行（样例）：

| source | tagFQN | labelType | state | appliedAt |
|---|---|---|---|---|
| 0 | Tier.Tier2 | 0 (Manual) | 1 (Confirmed) | 2026-09-01 11:19:08 |
| 0 | PII.Sensitive | 0 (Manual) | 1 (Confirmed) | 2026-09-01 11:19:23 |
| 0 | Tier.Tier1 | 0 (Manual) | 1 (Confirmed) | 2026-09-01 11:19:23 |

注意：`targetFQN` 原文**不存**，只有 `targetFQNHash`——所以"按标签找实体"走哈希查询，"反查标签原文"需要拿实体 FQN 现算哈希。

### 4.1 真实案例：术语当标签挂指标（source=1 Glossary）

术语有双重身份：既是实体，也是**可挂的标签**。POC 实查——术语 `经营管控业务术语库.合同评价收益.动态收益` 出现在 6 行 tag_usage 里，全部 `source=1 (Glossary), labelType=0 (Manual), state=1 (Confirmed)`，`appliedBy=admin`、2026-09-17 19:58（1.4 秒内 6 条，UI 批量打标）。反查 `metric_entity WHERE fqnHash IN (targetFQNHash)` 命中 6 个指标：

> 动态收益率 / 动态收益额 / 动态与合同评价收益偏差 / 动态与合同评价收益偏差总金额 / 二级公司全过程动态收益 / 川沪全过程动态收益

含义：**术语↔指标的语义对齐**就存在这张表里（AI 问数"按术语找候选指标"查的就是它）。注意 `metric_entity` 的 FQN 是单段的（指标名本身），所以 targetFQNHash 是单段 MD5——反查要按各表 FQN 段数分别匹配。

**`targetFQNHash` 算法（已端到端验证）**：`fqnHash = FQN 按 '.' 分段 → 每段 UTF-8 MD5（32 位小写 hex）→ 用 '.' 连接`。验证例：`md5("经营管控域")=b732e04f4fc16f03521699e8a78c2347`（= domain_entity.fqnHash）；`md5("ai_governance_mcp")+"."+md5("customer_profile_tools")` = mcp_server_entity.fqnHash；6 个指标名的 MD5 与 tag_usage 的 targetFQNHash 全部吻合。因此反查无需自己算哈希——实体表里已有同算法的 `fqnHash` 列，直接等值连接即可（老表如 glossary_entity 无 fqnHash 列，除外）。

## 5. `entity_extension` —— 真实版本快照

`glossary_entity.id='3c4083b9...'`（经营管控业务术语库，当前 `version=0.2`）：

| extension | json 大小 | 内容 |
|---|---|---|
| `glossary.version.0.1` | 854 字节 | **0.1 时的完整实体文档**（旧描述可从中读出） |

当前 json 里的 `changeDescription`（真实值，解释了 0.1→0.2 改了什么）：

```json
{
  "fieldsAdded": [
    { "name": "domains", "newValue": "[{\"id\":\"da5805ee-...\",\"type\":\"domain\",
       \"name\":\"经营管控域\",\"fullyQualifiedName\":\"经营管控域\", ...}]" }
  ]
}
```

即 0.1→0.2 的变更是"挂上经营管控域"——与 §3 里那条 `domain→glossaryTerm HAS` 边完全对应。三个证据互锁：`json.version=0.2` + `entity_extension` 有 0.1 快照 + `changeDescription.fieldsAdded=[domains]`。

## 6. 验证查询（可直接在你的库上跑）

### 6.1 按 `targetFQNHash` 反查实体（tag_usage → 实体）

`tag_usage` 里只有哈希没有原文，反查靠**等值连接**：OM 写实体时已用同一算法把 `fqnHash` 存进各实体表（算法见 §4.1 末尾），直接等值匹配即可，**不用自己算哈希，也不用查 OpenSearch**（OS 文档没有 fqnHash 字段，`_id` 是 UUID）。

不知道哈希对应哪类实体时，UNION 扫一遍常见实体表（把 `@h` 换成你的 targetFQNHash）：

```sql
SET @h = '3e49d4f002a184d6c9fb014b16ec2037';

SELECT 'metric' AS entity_type, id, name,
       JSON_UNQUOTE(JSON_EXTRACT(json,'$.fullyQualifiedName')) AS fqn
FROM metric_entity WHERE fqnHash = @h
UNION ALL
SELECT 'table', id, name, JSON_UNQUOTE(JSON_EXTRACT(json,'$.fullyQualifiedName'))
FROM table_entity WHERE fqnHash = @h
UNION ALL
SELECT 'dashboard', id, name, JSON_UNQUOTE(JSON_EXTRACT(json,'$.fullyQualifiedName'))
FROM dashboard_entity WHERE fqnHash = @h
UNION ALL
SELECT 'dashboardDataModel', id, name, JSON_UNQUOTE(JSON_EXTRACT(json,'$.fullyQualifiedName'))
FROM dashboard_data_model_entity WHERE fqnHash = @h;
```

实测：`3e49d4f002a184d6c9fb014b16ec2037` 命中 `metric` **动态收益率**（FQN=指标名本身，单段）。

覆盖全部实体类型的动态生成版（在 MySQL 客户端跑，先生成再执行）：

```sql
SET SESSION group_concat_max_len = 1000000;
SELECT GROUP_CONCAT(
  CONCAT('SELECT ''', REPLACE(table_name,'_entity',''), ''' AS entity_type, id, name FROM ', table_name,
         ' WHERE fqnHash = ''', @h, '''')
  SEPARATOR ' UNION ALL ') AS generated_sql
FROM information_schema.columns
WHERE table_schema = 'openmetadata_liujh' AND column_name = 'fqnHash' AND table_name LIKE '%\_entity';
```

注意：
- 有 `fqnHash` 列的表才参与；老表（如 `glossary_entity`）用 `name` 当哈希列，不在生成结果里
- 哈希大小写敏感；命中后看 `deleted` 列区分软删除
- OpenSearch 的用法相反：手里有**术语原文**要找"挂了它的资产"时，走 `search_metadata` 的 `queryFilter` 查 `tags.tagFQN`（OS 文档无哈希字段）

### 6.2 其他验证查询

```sql
-- 美化看实体文档
SELECT JSON_PRETTY(json) FROM glossary_entity WHERE id='3c4083b9-facb-4524-8683-bb0e73df468c';

-- 一个术语的全部关系边（relation 含义见 §3 对照）
SELECT er.relation, er.fromEntity, er.toEntity
FROM entity_relationship er
WHERE er.fromId='ba1bc817-c047-48f6-aa79-898b6a605a28'
   OR er.toId='ba1bc817-c047-48f6-aa79-898b6a605a28';

-- 版本链：当前版本 + 历史快照
SELECT JSON_EXTRACT(json,'$.version') FROM glossary_entity WHERE id='3c4083b9...';
SELECT extension FROM entity_extension WHERE id='3c4083b9...' AND extension LIKE 'glossary.version.%';

-- 变更内容
SELECT JSON_PRETTY(JSON_EXTRACT(json,'$.changeDescription')) FROM glossary_entity WHERE id='3c4083b9...';

-- 某实体挂了什么标签（targetFQNHash = FQN 各段哈希按 '.' 连接）
SELECT * FROM tag_usage WHERE targetFQNHash LIKE '842a05351b321f70f413c59a7fdc3c53.%';
```
