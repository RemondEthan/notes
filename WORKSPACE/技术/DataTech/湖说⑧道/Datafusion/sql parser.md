1. 调用入口
```Rust
// DataFusion SQL 转 AST 流程详解
// 1. 调用入口
// 入口: datafusion/core/src/execution/context/mod.rs
pub async fn sql(&self, sql: &str) -> Result<DataFrame> {
    self.sql_with_options(sql, SQLOptions::new()).await
}
```
2. 核心转换链
``` Rust
// 核心转换链
SessionContext::sql()
    ↓
SessionState::create_logical_plan(sql)
    ↓
SessionState::sql_to_statement(sql, dialect)
    ↓
DFParserBuilder::build()
    ↓
Tokenizer::tokenize_with_location()  // 分词
    ↓
Parser::parse_statements()           // 语法解析
    ↓
Result: VecDeque<Statement>  // AST
```
3. 关键代码

| 功能      | 文件路径                                                   |
| ------- | ------------------------------------------------------ |
| 入口层     | datafusion/core/src/execution/session_state.rs:398-430 |
| 解析器构建   | datafusion/sql/src/parser.rs                           |
| AST 定义  | datafusion/sql/src/parser.rs:277-296 (Statement 枚举)    |
| Planner | datafusion/sql/src/planner.rs                          |
|         |                                                        |
|         |                                                        |
4. 使用的解析器
sqlparser-rs 0.59.0 - 外部 SQL 解析库
```Rust
use sqlparser::{
    ast::{Statement as SQLStatement, Expr, ...},
    dialect::{Dialect, GenericDialect, ...},
    parser::Parser,
    tokenizer::{Token, Tokenizer},
};
```
5. AST结构定义
```Rust
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum Statement {
    // 来自 sqlparser-rs 的标准 SQL AST
    Statement(Box<SQLStatement>),
    // DataFusion 扩展
    CreateExternalTable(CreateExternalTable),
    CopyTo(CopyToStatement),
    Explain(ExplainStatement),
    Reset(ResetStatement),
}
```
6. 解析流程代码
```Rust
/// Parse an SQL string into an DataFusion specific AST  
/// [`Statement`]. See [`SessionContext::sql`] for running queries.///  
/// [`SessionContext::sql`]: crate::execution::context::SessionContext::sql  
#[cfg(feature = "sql")]  
pub fn sql_to_statement(  
    &self,  
    sql: &str,  
    dialect: &Dialect,  
) -> datafusion_common::Result<Statement> {  
    let dialect = dialect_from_str(dialect).ok_or_else(|| {  
        plan_datafusion_err!(  
            "Unsupported SQL dialect: {dialect}. Available dialects: \  
                 Generic, MySQL, PostgreSQL, Hive, SQLite, Snowflake, Redshift, \  
                 MsSQL, ClickHouse, BigQuery, Ansi, DuckDB, Databricks."  
        )  
    })?;  
  
    let recursion_limit = self.config.options().sql_parser.recursion_limit;  
  
    let mut statements = DFParserBuilder::new(sql)  
        .with_dialect(dialect.as_ref())  
        .with_recursion_limit(recursion_limit)  
        .build()?  
        .parse_statements()?;  
  // 只支持单个语句
    if statements.len() > 1 {  
        return datafusion_common::not_impl_err!(  
            "The context currently only supports a single SQL statement"  
        );  
    }  
  
    let statement = statements.pop_front().ok_or_else(|| {  
        plan_datafusion_err!("No SQL statements were provided in the query string")  
    })?;  
    Ok(statement)  
}
```
7. 分词 + 解析
```Rust
// datafusion/sql/src/parser.rs
pub fn build(self) -> Result<DFParser> {
    // 1. 分词
    let mut tokenizer = Tokenizer::new(self.dialect, self.sql);
    let tokens = tokenizer.tokenize_with_location()?;
    
    // 2. 构建解析器
    Ok(DFParser {
        parser: Parser::new(self.dialect)
            .with_tokens_with_locations(tokens)
            .with_recursion_limit(self.recursion_limit),
    })
}
pub fn parse_statements(&mut self) -> Result<VecDeque<Statement>> {
    let mut stmts = VecDeque::new();
    loop {
        while self.parser.consume_token(&Token::SemiColon) {}
        
        if self.parser.peek_token() == Token::EOF {
            break;
        }
        
        let statement = self.parse_statement()?;
        stmts.push_back(statement);
    }
    Ok(stmts)
}
```
8. 支持的 SQL 方言
>Generic、MySQL、PostgreSQL、Hive、SQLite、Snowflake、Redshift、MsSQL、ClickHouse、BigQuery、Ansi、DuckDB、Databricks
9. 完整示例
```Rust
let ctx = SessionContext::new();
let df = ctx.sql("SELECT * FROM t WHERE id = 1").await?;
// 内部流程:
// 1. SQL 字符串 → Token 序列 (Tokenizer)
// 2. Token 序列 → AST (Parser)
// 3. AST → LogicalPlan (SqlToRel planner)
// 4. LogicalPlan → 执行
```
总结: DataFusion 使用 sqlparser-rs 库进行词法分析和语法解析，将 SQL 字符串转换为 AST（Statement 枚举），然后通过 SqlToRel planner 将 AST 转换为 LogicalPlan 进行执行。