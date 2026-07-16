# 05: Rows、NULL、时间与类型转换

数据库里的 NULL、时间和类型转换经常让 Go 初学者踩坑。`Scan` 的目标类型必须能表达数据库返回的值。

---

## 1. Rows 生命周期

```go
rows, err := db.QueryContext(ctx, query)
if err != nil {
    return err
}
defer rows.Close()
```

`Rows` 持有数据库连接资源。忘记关闭会影响连接池。

---

## 2. 遍历后检查 Err

```go
for rows.Next() {
    // scan
}
if err := rows.Err(); err != nil {
    return err
}
```

遍历中发生的错误可能在 `rows.Err()` 中体现。

---

## 3. NULL 不能扫进普通 string

如果数据库列可能是 NULL：

```sql
nickname TEXT NULL
```

不能直接假设：

```go
var nickname string
```

NULL 扫进 string 会报错。使用 `sql.NullString`：

```go
var nickname sql.NullString
if nickname.Valid {
    fmt.Println(nickname.String)
}
```

---

## 4. 常见 Null 类型

```go
sql.NullString
sql.NullInt64
sql.NullFloat64
sql.NullBool
sql.NullTime
```

这些类型都有 `Valid` 字段。`Valid=false` 表示数据库值是 NULL。

---

## 5. 指针表达可选值

也可以在业务结构里用指针表达可选值：

```go
type User struct {
    Nickname *string
}
```

但 `Scan` 时通常仍要中转：

```go
var ns sql.NullString
if err := row.Scan(&ns); err != nil {
    return err
}
if ns.Valid {
    user.Nickname = &ns.String
}
```

---

## 6. 时间字段

```go
type User struct {
    CreatedAt time.Time
}
```

MySQL 使用时间字段时，DSN 通常需要：

```text
parseTime=true
```

否则时间列可能不能直接扫进 `time.Time`。

---

## 7. created_at 和 updated_at

可以由应用写入：

```go
now := time.Now().UTC()
_, err := db.ExecContext(ctx, `
    INSERT INTO users(name, created_at, updated_at)
    VALUES(?, ?, ?)
`, name, now, now)
```

也可以由数据库默认值和触发器维护。选择哪种方式要在项目中统一。

---

## 8. 扫描数字

数据库整数可以扫进 `int64` 更稳：

```go
var id int64
```

`COUNT(*)` 通常也扫进 `int64`。

```go
var count int64
```

---

## 9. RawBytes

`sql.RawBytes` 引用数据库驱动内部缓冲，下一次 Scan 后内容可能失效。

初学阶段尽量避免使用它。需要字节数据时，用 `[]byte`。

---

## 10. 类型设计检查

1. 数据库列是否允许 NULL。
2. Go 类型能否表达 NULL。
3. 时间列是否能扫进 `time.Time`。
4. 数字范围是否合适。
5. Scan 顺序和 SQL 列顺序是否一致。

---

## 练习

1. 创建包含 nullable nickname 的 users 表。
2. 用 `sql.NullString` 读取 nickname。
3. 把 `sql.NullString` 转成 `*string`。
4. 插入 `created_at` 和 `updated_at`。
5. 用 `COUNT(*)` 查询数量并扫进 `int64`。
