# 数据工具链本地智能开发技术方案

---

## 1. 方案概述

### 1.1 背景

数据工具链（DTC）在线上已具备一体化大数据开发、数据分析等能力，现希望将部分能力下沉到线下AI Coding工具（Cursor、Claude Code、Trae 等），让用户在本地工程内通过自然语言完成，打造线上线下协同的数据开发体验。

### 1.2 目标

- 在本地AI Coding工具中提供完整的TextToSQL开发能力
- 实现云端元数据与知识的本地同步与增量更新
- 构建基于Skill + MCP的插件化架构，支持灵活扩展
- 保障数据安全与权限控制

###  核心架构

```
┌─────────────────────────────────────────────────────┐
│                   AI Coding Tool                    │
│  ┌──────────────────────────────────────────────┐   │
│  │                  Skill 层                     │  │
│  │  环境安装 │ 元数据同步 │ 知识管理 │ TextToSQL     │   │
│  └──────────────────────────────────────────────┘   │
│                         ↓ MCP 接口调用               │
└─────────────────────────────────────────────────────┘
                         ↓ HTTPS (Token鉴权)
┌─────────────────────────────────────────────────────┐
│              大数据开发平台 (云端)                    │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐             │
│  │ 元数据接口 │ │ 知识接口 │ │ SQL执行接口│            │
│  └──────────┘ └──────────┘ └──────────┘             │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐             │
│  │ 视图接口  │ │ 数据源接口│ │ Skill接口 │             │
│  └──────────┘ └──────────┘ └──────────┘             │
└─────────────────────────────────────────────────────┘
```

---

## 2. 技术架构设计

### 2.1 整体架构分层

| 层级      | 组件                           | 职责                    |
| ------- | ---------------------------- | --------------------- |
| **交互层** | AI Coding工具                  | 用户自然语言交互、代码生成与展示      |
| **技能层** | Skill                        | 业务逻辑编排、MCP接口调用、本地文件管理 |
| **协议层** | MCP (Model Context Protocol) | 标准化接口通信、流式数据传输        |
| **服务层** | 云端API                        | 元数据管理、知识管理、SQL执行、视图发布 |
| **存储层** | 本地.dtc目录 + 云端数据库             | 元数据缓存、知识持久化           |

### 2.2 鉴权机制

- **Token配置**：用户需在系统环境变量中配置平台Token（如：`DTC_PLATFORM_TOKEN`）
- **请求鉴权**：所有MCP接口调用必须在请求头携带Token：`Authorization: Bearer {token}`
- **权限隔离**：云端根据Token识别用户身份，仅返回该用户有权限的工程与数据源元数据
- **Token刷新**：支持Token过期自动提示更新机制

---

## 3. Skill 设计

### 3.2 Skill 清单

| Skill名称 | 描述 | 作用 | 触发条件 |
|-----------|------|------|----------|
| **dtc** |DTC线下技能引用| DTC环境安装与更新 | 初始化本地DTC开发环境，安装MCP配置，同步元数据与知识 | 用户输入"安装DTC环境"、"初始化DTC"、"更新DTC环境"等 |
| **dtc-metadata-sync** | 元数据同步管理 | 从云端拉取工程、数据源、表DDL元数据，增量/全量同步 | 用户输入"同步元数据"、"更新表结构"、"刷新数据源"等 |
| **dtc-text2sql** | TextToSQL智能转换 | 基于本地元数据，将自然语言转换为标准SQL查询语句 | 用户输入自然语言描述的数据查询需求，如"查询最近7天的订单数据" |
| **dtc-view-publish** | 视图发布管理 | 将生成的SQL发布为云端视图，支持视图版本管理 | 用户输入"发布为视图"、"创建视图"、"保存SQL为视图"等 |
| **dtc-sql-preview** | SQL数据预览 | 执行SQL并返回预览数据，支持结果格式化 | 用户输入"预览数据"、"执行SQL"、"查看结果"等 |
| **dtc-knowledge-mgmt** | 知识管理 | 提取对话与SQL生成知识，本地存储并同步至云端 | 自动生成；用户也可输入"查看历史知识"、"查询业务含义"等 |

### 3.3 Skill 设计规范

- **单一职责**：每个Skill只负责一个核心业务场景
- **幂等性**：Skill执行需支持重复执行，不会产生副作用
- **可观测性**：执行过程需输出清晰的进度与状态信息
- **容错处理**：网络异常、权限不足等场景需有明确的错误提示

#### 3.3.2 Skill 执行流程模板

```
1. 解析用户意图 → 2. 验证环境配置 → 3. 调用MCP接口 → 4. 处理响应 → 5. 本地操作 → 6. 返回结果
```

#### 3.3.3 环境依赖检查

每个Skill执行前需检查：
- 环境变量 `DTC_PLATFORM_TOKEN` 是否配置
- `.dtc` 目录是否存在且结构完整
- MCP Server是否可用
- 网络连接状态

---

## 4. MCP 接口设计

### 4.2 MCP 接口清单

| MCP接口名称 | 描述 |
|-------------|------|
| **mcp_discover_skills** | 发现可用Skill |
| **mcp_install_skill** | 安装指定Skill |
| **mcp_stream_metadata** | 流式获取元数据 |
| **mcp_publish_view** | 发布视图 |
| **mcp_execute_sql** | 执行SQL预览 |
| **mcp_upload_knowledge** | 上传知识至云端 |
| **mcp_download_knowledge** | 下载云端知识 |
| **mcp_check_version** | 检查组件版本 |

### 4.3 MCP 通信规范

- **鉴权**：所有请求Header携带 `Authorization: Bearer {DTC_PLATFORM_TOKEN}`
- **超时**：同步接口超时时间30秒，流式接口首字节超时60秒

---

## 5. .dtc 目录结构设计

### 5.1 设计原则

`.dtc`目录用于管理本地元数据与知识，遵循以下原则：
- **分层清晰**：元数据与知识分离
- **可追溯**：记录同步时间、版本号、校验信息

### 5.2 目录结构
```
.cursor
└── skills/                       # Skills目录
    └── dtc                 
        └── SKILL.md              # DTC Skill
```


```
.dtc/
├── README.md                          # DTC环境说明文档
├── config.yaml                        # DTC环境配置
│
├── metadata/                          # 元数据存储（解耦设计）
│   ├── index.md                       # 元数据索引概览
│   ├── mappings/                      # 映射关系定义
│   │   ├── project_datasource.md      # 工程-数据源映射关系
│   │   └── datasource_database.md     # 数据源-数据库映射关系
│   │
│   ├── projects/                      # 工程元数据（独立存储）
│   │   ├── <project_name>/
│   │   │   ├── _overview.md           # 工程基本信息与配置
│   │   │   └── permissions.md         # 工程权限信息
│   │   └── ...
│   │
│   ├── datasources/                   # 数据源元数据（独立存储）
│   │   ├── <datasource_name>/
│   │   │   ├── _info.md               # 数据源连接信息（脱敏）
│   │   │   └── type_info.md           # 数据源类型配置（MySQL/PG/Hive等）
│   │   └── ...
│   │
│   ├── databases/                     # 数据库元数据（独立存储）
│   │   ├── <database_name>/
│   │   │   ├── _schema.md             # 数据库概览（表列表）
│   │   │   └── tables/
│   │   │       ├── <table_name>.md    # 表DDL与字段注释
│   │   │       └── ...
│   │   └── ...
│   │
│   └── sync_log.json                  # 同步日志（时间、状态、变更统计）
│
├── knowledge/                         # 知识存储
│   ├── index.md                       # 知识索引（按主题分类）
│   ├── queries/                       # 查询知识
│   │   ├── <query_hash>.md            # 查询SQL与业务含义
│   │   └── ...
│   ├── views/                         # 视图知识
│   │   ├── <view_name>.md             # 视图SQL与业务说明
│   │   └── ...
│   ├── sync_log.json                  # 知识同步日志
│   └── uploaded/                      # 已上传云端知识标记
│       └── ...
│
├── skills/                            # 本地Skill缓存
│   ├── <skill_name>/
│   │   ├── SKILL.md
│   │   └── version.json
│   └── ...
│
└── cache/                             # 缓存数据
    ├── last_sync_timestamp.json       # 最后同步时间戳
    └── temp/                          # 临时文件
```

### 5.3 核心文件说明

#### 5.3.1 config.yaml

```yaml
platform:
  base_url: "https://aecloud-test.glodon.com"
  
auth:
  token_env: "DTC_PLATFORM_TOKEN"
  
sync:
  metadata:
    strategy: "incremental"  # incremental | full
    last_sync: "2024-01-15T10:30:00Z"
  knowledge:
    auto_upload: true
    last_sync: "2024-01-15T10:30:00Z"
    
skills:
  auto_update: true
  check_interval_hours: 24
```

#### 5.3.3 metadata/index.md

```markdown
# 元数据索引概览

## 统计信息

| 类别 | 数量 | 最后同步时间 |
|------|------|-------------|
| 工程 | 5 | 2024-01-15 10:30 |
| 数据源 | 8 | 2024-01-15 10:30 |
| 数据库 | 12 | 2024-01-15 10:32 |
| 数据表 | 256 | 2024-01-15 10:32 |

## 映射关系概览

- 工程-数据源映射: 15 条关联关系
- 数据源-数据库映射: 20 条关联关系
```

#### 5.3.4 映射关系文件

**project_datasource.md** - 工程与数据源映射关系:

```markdown
# 工程-数据源映射关系

## 映射表

| 工程名称 | 数据源名称 | 关联时间 |
|---------|-----------|---------|
| project_sales | mysql_prod | 2024-01-10 |
| project_sales | hive_warehouse | 2024-01-10 |
| project_sales | redis_cache | 2024-01-12 |
| project_user | mysql_prod | 2024-01-11 |
| project_user | pg_analytics | 2024-01-11 |
| project_finance | mysql_prod | 2024-01-12 |
```

**datasource_database.md** - 数据源与数据库映射关系:

```markdown
# 数据源-数据库映射关系

## 映射表

| 数据源名称 | 数据库名称 | 关联时间 | 备注 |
|-----------|-----------|---------|------|
| mysql_prod | sales_db | 2024-01-10 |
| mysql_prod | user_db | 2024-01-11 |
| mysql_prod | finance_db | 2024-01-12 |
| hive_warehouse | dw_ods | 2024-01-10 |
| hive_warehouse | dw_dwd | 2024-01-10 |
| hive_warehouse | dw_dws | 2024-01-10 |
| pg_analytics | analytics_db | 2024-01-11 |
| oracle_fin | fin_core | 2024-01-12 |
```

#### 5.3.5 工程元数据示例 `<project_name>/_overview.md`

```markdown
# 工程: project_sales

## 基本信息
- **工程ID**: 123123
- **工程名称**: project_sales
- **描述**: XXX工程
- **创建时间**: 2024-01-10

## 关联数据源
参见 [工程-数据源映射](../mappings/project_datasource.md)

| 数据源名称 | 类型 |
|-----------|------|
| mysql_prod | MySQL |
| hive_warehouse | Hive |
| redis_cache | Redis |
```

#### 5.3.6 数据源元数据示例 `<datasource_name>/_info.md`

```markdown
# 数据源: mysql_prod

## 基本信息
- **数据源ID**: 123123
- **数据源名称**: mysql_prod
- **类型**: StarRocks
- **最后同步**: 2024-01-15 10:30

## 关联工程
参见 [工程-数据源映射](../mappings/project_datasource.md)

| 工程名称 | 工程编码 |
|---------|---------|
| project_sales | project_sales |
| project_user | project_user |
| project_finance | project_finance |

## 关联数据库
参见 [数据源-数据库映射](../mappings/datasource_database.md)

| 数据库名称 | 表数量 |
|-----------|--------|
| sales_db | 45 |
| user_db | 32 |
| finance_db | 28 |
```

#### 5.3.7 表元数据示例 `<database_name>/tables/<table_name>.md`

```markdown
# 表: orders

## 基本信息
- **所属数据库**: sales_db
- **数据源**: mysql_prod
- **描述**: 订单主表
- **最后更新**: 2024-01-15
- **关联工程**: project_sales, project_finance (通过数据源映射)

## DDL

    CREATE TABLE orders (
      order_id BIGINT PRIMARY KEY COMMENT '订单ID',
      user_id BIGINT NOT NULL COMMENT '用户ID',
      amount DECIMAL(10,2) COMMENT '订单金额',
      status TINYINT DEFAULT 0 COMMENT '订单状态: 0-待支付, 1-已支付, 2-已取消',
      create_time DATETIME DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间'
    ) COMMENT '订单主表';

## 字段说明

| 字段名 | 类型 | 约束 | 说明 |
|--------|------|------|------|
| order_id | BIGINT | PK | 订单ID |
| user_id | BIGINT | NOT NULL | 用户ID |
| amount | DECIMAL(10,2) | - | 订单金额 |
| status | TINYINT | DEFAULT 0 | 订单状态 |
| create_time | DATETIME | DEFAULT CURRENT_TIMESTAMP | 创建时间 |
```

#### 5.3.8 知识示例 `<query_hash>.md`

```markdown
# 知识: 查询最近7天已支付订单

## 元信息
- **创建时间**: 2024-01-15 14:30
- **来源**: 用户对话
- **关联表**: orders
- **关联工程**: project_sales

## 业务场景
查询最近7天内状态为已支付的订单，按订单金额降序排列。

## SQL

    SELECT order_id, user_id, amount, create_time
    FROM orders
    WHERE status = 1
      AND create_time >= DATE_SUB(NOW(), INTERVAL 7 DAY)
    ORDER BY amount DESC;

## 备注
- 仅查询已支付订单(status=1)
- 时间范围可调整INTERVAL参数
```

---

## 6. 用户操作流程

### 6.1 初始安装流程

```
用户配置环境变量 DTC_PLATFORM_TOKEN
        ↓
用户在AI Coding工具中安装 dtc-env-setup Skill
        ↓
用户输入自然语言："安装DTC开发环境"
        ↓
Skill触发:
  1. 检查环境变量Token
  2. 调用 mcp_discover_skills 获取可用Skill列表
  3. 调用 mcp_install_skill 安装各Skill
  4. 调用 mcp_stream_metadata 拉取元数据
  5. 调用 mcp_download_knowledge 拉取知识
  6. 创建 .dtc 目录结构
  7. 生成 config.yaml
        ↓
返回安装完成提示，展示已安装的Skill与同步的元数据统计
```

### 6.2 TextToSQL使用流程

```
用户输入自然语言："查询最近7天华东地区的销售订单"
        ↓
dtc-text2sql Skill触发:
  1. 读取 .dtc/metadata 目录下的元数据
  2. 理解用户意图，匹配相关表与字段
  3. 生成SQL语句
  4. 返回SQL及解释说明
        ↓
知识自动提取:
  dtc-knowledge-mgmt 将对话与SQL提炼为知识
  存储至 .dtc/knowledge/queries/
  异步调用 mcp_upload_knowledge 上传云端
```

### 6.3 视图发布流程

```
用户输入："将刚才的SQL发布为视图，名称为 v_recent_east_orders"
        ↓
dtc-view-publish Skill触发:
  1. 提取SQL内容与用户指定的视图名称
  2. 调用 mcp_publish_view 接口
  3. 等待云端返回视图ID与状态
        ↓
生成本地知识 .dtc/knowledge/views/v_recent_east_orders.md
返回发布成功信息与视图访问链接
```

### 6.4 SQL预览流程

```
用户输入："预览这条SQL的前100条数据"
        ↓
dtc-sql-preview Skill触发:
  1. 提取SQL语句
  2. 调用 mcp_execute_sql 接口
  3. 解析返回结果，格式化展示
        ↓
展示数据预览表格，支持分页查看
```

### 6.5 增量更新流程

```
用户输入："更新DTC环境"
        ↓
dtc-env-setup Skill触发:
  1. 调用 mcp_check_version 检查各组件版本
  2. 对比本地 version.json
  3. 如有Skill更新：调用 mcp_install_skill 更新
  4. 元数据增量同步（按实体类型分别同步）:
     - 读取 sync_log.json 获取各实体类型最后同步时间
     - 调用 mcp_stream_metadata(entity_type=project, last_sync_time) 拉取变更工程
     - 调用 mcp_stream_metadata(entity_type=datasource, last_sync_time) 拉取变更数据源
     - 调用 mcp_stream_metadata(entity_type=database, last_sync_time) 拉取变更数据库与表DDL
     - 调用 mcp_stream_metadata(entity_type=mapping, last_sync_time) 拉取变更映射关系
     - 更新对应 .md 文件与映射文件
  5. 知识增量同步：
     - 调用 mcp_download_knowledge 拉取新增知识
     - 合并至 .dtc/knowledge/ 目录
  6. 更新 version.json 与 config.yaml
        ↓
返回更新摘要：新增/变更的Skill数量、元数据变更统计（工程/数据源/数据库/映射）、知识更新数量
```

### 6.6 全量更新流程（备选）

```
用户输入："全量更新DTC环境"
        ↓
执行流程同增量更新，但：
  - 清空 .dtc/metadata/ 目录
  - 调用 mcp_stream_metadata(entity_type=project) 全量拉取工程元数据
  - 调用 mcp_stream_metadata(entity_type=datasource) 全量拉取数据源元数据
  - 调用 mcp_stream_metadata(entity_type=database) 全量拉取数据库与表DDL
  - 调用 mcp_stream_metadata(entity_type=mapping) 全量拉取映射关系
  - 重新生成所有 .md 文件与映射文件
        ↓
返回全量更新完成提示
```

---

## 7. 关键技术细节

### 7.1 元数据流式下载

**问题**：元数据量大，单次下载可能超时或内存溢出

**解决方案**：
- 采用SSE (Server-Sent Events) 流式传输
- 按库为单位生成 `.md` 文件，边接收边写入
- 支持断点续传，记录已接收的库列表
- 使用checksum校验完整性

### 7.2 增量更新策略

**元数据增量**：
- 云端记录各实体类型（工程/数据源/数据库/映射）的变更日志
- 客户端按实体类型分别携带 `last_sync_timestamp` 请求增量数据
- 变更数据合并至本地对应目录：
  - 工程变更：更新 `projects/<project_name>/` 目录
  - 数据源变更：更新 `datasources/<datasource_name>/` 目录
  - 数据库变更：更新 `databases/<database_name>/` 目录（含表DDL）
  - 映射变更：更新 `mappings/` 目录下的映射关系文件

**知识增量**：
- 云端记录知识变更时间
- 客户端拉取 `updated_at > last_sync_time` 的知识
- 本地采用upsert策略更新

### 7.3 知识自动提取

**触发时机**：
- TextToSQL生成SQL后自动提取
- 用户手动标注业务含义时提取

**提取内容**：
- 用户原始自然语言描述
- 生成的SQL语句
- 涉及的表与字段
- 业务场景标签（自动分类）

**存储策略**：
- 本地即时存储，确保离线可用
- 异步上传云端，失败自动重试
- 已上传标记避免重复上传

### 7.4 权限与安全

- Token仅存储于环境变量，不落盘
- 元数据下载前进行权限校验
- 敏感字段（如密码、密钥）自动脱敏
- SQL执行前进行安全校验（禁止DROP/DELETE等危险操作）

---

## 8. 异常处理

| 异常场景 | 处理策略 |
|---------|---------|
| Token未配置 | 提示用户配置环境变量，提供配置指引 |
| Token过期/无效 | 提示重新获取Token，暂停所有MCP调用 |
| 网络断开 | 降级使用本地元数据，网络恢复后自动同步 |
| 元数据同步中断 | 记录断点，下次同步时续传 |
| SQL执行失败 | 返回错误详情，提供SQL语法检查建议 |
| 知识上传失败 | 本地标记未上传，后台定时重试 |

---