# 09: 数据库与持久化

本模块讲 Go 中的数据库访问和持久化。主线使用标准库 `database/sql` 建立心智模型，再扩展到连接池、事务、SQL 参数、迁移、测试、SQLite 和仓储边界。

---

## 文章列表

1. [database/sql 心智模型](./01-database-sql心智模型.md)
2. [连接数据库、Ping 与连接池配置](./02-连接数据库Ping与连接池.md)
3. [Exec、QueryRow、Query 与 Scan](./03-Exec-QueryRow-Query与Scan.md)
4. [CRUD：增删改查的基本写法](./04-CRUD增删改查.md)
5. [Rows、NULL、时间与类型转换](./05-Rows-NULL时间与类型转换.md)
6. [SQL 参数、注入风险与动态查询](./06-SQL参数注入风险与动态查询.md)
7. [事务：BeginTx、Commit、Rollback](./07-事务BeginTx-Commit-Rollback.md)
8. [迁移、schema 与索引](./08-迁移schema与索引.md)
9. [Repository 边界与测试替身](./09-Repository边界与测试替身.md)
10. [数据库测试：SQLite、临时库与清理](./10-数据库测试SQLite临时库与清理.md)
11. [文件持久化：JSON、CSV 与原子写入](./11-文件持久化JSON-CSV与原子写入.md)
12. [数据库与持久化综合练习](./12-数据库与持久化综合练习.md)

---

## 学习目标

完成本模块后，你应该能够：

- 理解 `database/sql`、驱动、连接池之间的关系。
- 正确使用 `Open`、`PingContext`、`SetMaxOpenConns` 等配置。
- 使用 `ExecContext`、`QueryRowContext`、`QueryContext` 做基础数据库操作。
- 正确关闭 `Rows`，检查 `Rows.Err()`。
- 处理 `sql.ErrNoRows`、NULL 值、时间字段和自增 ID。
- 使用参数化 SQL，避免 SQL 注入。
- 正确写事务，保证失败时 rollback，成功时 commit。
- 理解迁移、schema、索引和唯一约束的作用。
- 设计小的 repository 接口，方便测试和替换实现。
- 用 SQLite 或临时数据库写数据库测试。
- 在简单场景下用 JSON/CSV 文件做持久化，并理解原子写入。

---

## 学习建议

数据库代码要特别重视边界：

1. SQL 是否参数化。
2. 查询结果是否关闭。
3. 没查到数据和真正错误是否区分。
4. 事务失败路径是否 rollback。
5. context 是否传入数据库调用。
6. 测试是否覆盖约束、错误路径和事务回滚。

---

## 下一步

完成本模块后，继续学习：

- **10**: Web 服务
