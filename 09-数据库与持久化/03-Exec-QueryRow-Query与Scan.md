# 03: Exec、QueryRow、Query 与 Scan

`database/sql` 的日常操作主要是三类：执行语句、查询一行、查询多行。

---

## 1. ExecContext

`ExecContext` 用于不需要返回数据行的 SQL：

```go
result, err := db.ExecContext(ctx,
    "INSERT INTO users(name, email) VALUES(?, ?)",
    name,
    email,
)
if err != nil {
    return err
}
```

适合：

- `INSERT`
- `UPDATE`
- `DELETE`
- `CREATE TABLE`

---

## 2. RowsAffected

```go
affected, err := result.RowsAffected()
if err != nil {
    return err
}
fmt.Println(affected)
```

更新或删除时，可以检查影响行数。

```go
if affected == 0 {
    return ErrNotFound
}
```

---

## 3. LastInsertId

```go
id, err := result.LastInsertId()
if err != nil {
    return err
}
```

并不是所有数据库和驱动都支持 `LastInsertId`。PostgreSQL 常用 `RETURNING id`。

```sql
INSERT INTO users(name) VALUES($1) RETURNING id
```

---

## 4. QueryRowContext

查询一行：

```go
var user User
err := db.QueryRowContext(ctx,
    "SELECT id, name, email FROM users WHERE id = ?",
    id,
).Scan(&user.ID, &user.Name, &user.Email)
```

`QueryRowContext` 会延迟到 `Scan` 时返回错误。

---

## 5. sql.ErrNoRows

```go
if errors.Is(err, sql.ErrNoRows) {
    return User{}, ErrNotFound
}
if err != nil {
    return User{}, err
}
```

没查到数据是常见业务分支，不应该直接当成 500 类错误。

---

## 6. QueryContext

查询多行：

```go
rows, err := db.QueryContext(ctx,
    "SELECT id, name, email FROM users ORDER BY id",
)
if err != nil {
    return nil, err
}
defer rows.Close()
```

`Rows` 必须关闭。

---

## 7. 遍历 Rows

```go
var users []User
for rows.Next() {
    var user User
    if err := rows.Scan(&user.ID, &user.Name, &user.Email); err != nil {
        return nil, err
    }
    users = append(users, user)
}
if err := rows.Err(); err != nil {
    return nil, err
}
return users, nil
```

循环结束后检查 `rows.Err()`，它可能包含遍历过程中的错误。

---

## 8. Scan 顺序必须匹配

```go
SELECT id, name, email FROM users
```

对应：

```go
rows.Scan(&user.ID, &user.Name, &user.Email)
```

列顺序和 Scan 目标顺序必须一致。查询字段发生变化时，要同步更新 Scan。

---

## 9. Select *

避免在业务代码里长期使用：

```sql
SELECT * FROM users
```

明确列名更稳定：

```sql
SELECT id, name, email, created_at FROM users
```

表结构变更时，`SELECT *` 更容易让 Scan 顺序和数量出问题。

---

## 10. 错误包装

```go
if err != nil {
    return fmt.Errorf("query user %d: %w", id, err)
}
```

数据库错误要带上操作上下文，但不要把敏感 SQL 参数、密码、token 写进错误。

---

## 练习

1. 用 `ExecContext` 插入一条用户记录。
2. 用 `RowsAffected` 判断更新是否命中。
3. 用 `QueryRowContext` 查询单个用户，并处理 `sql.ErrNoRows`。
4. 用 `QueryContext` 查询用户列表，关闭 rows 并检查 `rows.Err()`。
5. 解释 `QueryRowContext` 为什么通常在 `Scan` 时才返回错误。
