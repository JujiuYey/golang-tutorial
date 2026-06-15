# 01: database/sql 心智模型

`database/sql` 是 Go 标准库提供的数据库访问抽象。具体的 MySQL、PostgreSQL、SQLite 协议由数据库驱动负责实现。

---

## 1. database/sql 负责什么

`database/sql` 负责：

- 提供统一 API。
- 管理连接池。
- 执行 SQL。
- 扫描查询结果。
- 管理事务入口。

它不负责：

- 具体数据库协议。
- 自动生成业务 SQL。
- schema 迁移。
- ORM 模型映射。

具体数据库能力由驱动提供。

---

## 2. 驱动负责什么

常见驱动：

```go
import _ "github.com/go-sql-driver/mysql"
import _ "github.com/lib/pq"
import _ "modernc.org/sqlite"
```

下划线导入表示只执行包的初始化逻辑。数据库驱动会在 init 中注册自己。

```go
db, err := sql.Open("sqlite", "file:test.db")
```

第一个参数 `"sqlite"` 是驱动名。它必须和驱动注册的名字匹配。

---

## 3. sql.DB 不是单个连接

```go
var db *sql.DB
```

`*sql.DB` 表示数据库句柄和连接池。它会按需创建、复用和回收连接。

不要为每个请求都 `sql.Open` 一次。通常在程序启动时创建一个 `*sql.DB`，在程序退出时关闭。

---

## 4. sql.Open 不一定连接数据库

```go
db, err := sql.Open("sqlite", "file:test.db")
if err != nil {
    return err
}
```

`Open` 主要验证参数并创建句柄，不一定真正建立网络连接。

要验证数据库可用，使用：

```go
ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
defer cancel()

if err := db.PingContext(ctx); err != nil {
    return err
}
```

---

## 5. 三类常用操作

执行不返回行的 SQL：

```go
result, err := db.ExecContext(ctx, query, args...)
```

查询一行：

```go
err := db.QueryRowContext(ctx, query, args...).Scan(&dest)
```

查询多行：

```go
rows, err := db.QueryContext(ctx, query, args...)
```

`QueryContext` 返回的 `Rows` 必须关闭。

---

## 6. context 很重要

数据库操作可能阻塞：

- 等待连接池可用连接。
- 等待数据库执行 SQL。
- 等待网络返回结果。

使用带 context 的方法：

```go
ExecContext
QueryContext
QueryRowContext
BeginTx
```

这样上游请求取消或超时时，数据库调用也有机会停止。

---

## 7. SQL 和参数分开

```go
row := db.QueryRowContext(ctx, "SELECT name FROM users WHERE id = ?", id)
```

不要拼接用户输入到 SQL 字符串里。参数化查询可以避免 SQL 注入，也能让驱动正确处理转义和类型。

---

## 8. 占位符因数据库而异

MySQL、SQLite 常见：

```sql
WHERE id = ?
```

PostgreSQL 常见：

```sql
WHERE id = $1
```

写课程和项目时，要明确使用的数据库和驱动。

---

## 9. ORM 和 database/sql 的关系

ORM 或查询构建器通常建立在数据库驱动和连接之上，帮你生成 SQL、映射结构体、管理关联。

学习 Go 数据库编程时，建议先掌握 `database/sql` 的底层语义：

- 连接池。
- context。
- Rows 关闭。
- Scan。
- 事务。
- 参数化 SQL。

理解这些后，再使用 ORM 会更稳。

---

## 10. 数据库代码检查

看到数据库代码时先问：

1. 是否传入 context。
2. 是否使用参数化 SQL。
3. `Rows` 是否关闭。
4. `sql.ErrNoRows` 是否单独处理。
5. 事务是否覆盖所有相关写操作。
6. 连接池是否配置。

---

## 练习

1. 解释 `database/sql` 和数据库驱动的关系。
2. 解释为什么 `*sql.DB` 不是单个连接。
3. 写出 `ExecContext`、`QueryRowContext`、`QueryContext` 分别适合的场景。
4. 说明 MySQL 和 PostgreSQL 占位符的差异。
5. 找一段数据库代码，检查它有没有传 context、关 rows、处理 error。
