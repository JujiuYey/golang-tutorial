# 06: SQL 参数、注入风险与动态查询

数据库代码必须重视 SQL 注入。原则很简单：用户输入走参数，不拼进 SQL 字符串。

---

## 1. 错误示例

```go
query := "SELECT id, name FROM users WHERE email = '" + email + "'"
row := db.QueryRowContext(ctx, query)
```

如果 `email` 来自用户输入，这种写法有 SQL 注入风险。

---

## 2. 参数化查询

```go
row := db.QueryRowContext(ctx,
    "SELECT id, name FROM users WHERE email = ?",
    email,
)
```

SQL 和参数分开交给驱动，驱动负责处理类型和转义。

---

## 3. 占位符差异

MySQL、SQLite：

```sql
WHERE email = ?
```

PostgreSQL：

```sql
WHERE email = $1
```

不要在不了解驱动的情况下混用。

---

## 4. LIKE 查询

```go
keyword := "%" + strings.TrimSpace(input) + "%"
rows, err := db.QueryContext(ctx,
    "SELECT id, name FROM users WHERE name LIKE ?",
    keyword,
)
```

即使是 LIKE，也应该把值作为参数传入。

---

## 5. 动态条件

可以安全地拼接固定 SQL 片段，同时把用户输入放进参数。

```go
query := "SELECT id, name FROM users WHERE 1=1"
var args []any

if name != "" {
    query += " AND name LIKE ?"
    args = append(args, "%"+name+"%")
}
if minAge > 0 {
    query += " AND age >= ?"
    args = append(args, minAge)
}

rows, err := db.QueryContext(ctx, query, args...)
```

拼接的是你自己控制的 SQL 片段，不是用户输入。

---

## 6. 动态排序

排序字段不能用参数占位符代替列名。

```go
// ORDER BY ? 不能按列名工作
```

需要白名单：

```go
allowed := map[string]string{
    "name":       "name",
    "created_at": "created_at",
}

column, ok := allowed[sortBy]
if !ok {
    column = "created_at"
}

query += " ORDER BY " + column + " DESC"
```

方向也要白名单：

```go
direction := "ASC"
if desc {
    direction = "DESC"
}
```

---

## 7. IN 查询

标准库没有自动展开 `IN (?)` 列表的统一能力。常见做法是根据参数数量生成占位符：

```go
func placeholders(n int) string {
    if n <= 0 {
        return ""
    }
    return strings.TrimRight(strings.Repeat("?,", n), ",")
}
```

然后：

```go
ids := []int64{1, 2, 3}
query := "SELECT id, name FROM users WHERE id IN (" + placeholders(len(ids)) + ")"
args := make([]any, 0, len(ids))
for _, id := range ids {
    args = append(args, id)
}
```

空列表要单独处理。

---

## 8. 预编译语句

```go
stmt, err := db.PrepareContext(ctx, "SELECT name FROM users WHERE id = ?")
if err != nil {
    return err
}
defer stmt.Close()

row := stmt.QueryRowContext(ctx, id)
```

数据库和驱动可能会复用执行计划。实际是否有收益取决于场景。先掌握参数化查询，遇到高频 SQL 再考虑 prepare。

---

## 9. 不要打印敏感 SQL 参数

错误日志里不要输出密码、token、身份证号等敏感参数。

可以记录：

- 操作名。
- 表名或功能名。
- 请求 ID。
- 错误类型。

谨慎记录完整 SQL 和参数。

---

## 10. 安全检查

1. 用户输入是否都走参数。
2. 动态列名是否白名单。
3. 动态排序方向是否白名单。
4. IN 空列表是否处理。
5. 日志是否避免敏感参数。

---

## 练习

1. 把字符串拼接 SQL 改成参数化查询。
2. 实现带 name 和 minAge 的动态查询。
3. 实现排序字段白名单。
4. 实现 `placeholders(n int)`。
5. 说明为什么列名不能用普通参数占位符。
