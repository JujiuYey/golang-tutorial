# 03: Exec、QueryRow、Query 与 Scan

这篇把数据库日常的三个动词练到手:改数据用 `Exec`、查一行用 `QueryRow`、查多行用 `Query`,以及把结果**逐列**接进 Go 变量的 `Scan`。学完你能写出错误处理完整的查询代码,并躲开"没查到当成故障"和"Scan 数量对不上"两个高频坑。

> 示例用 SQLite(`modernc.org/sqlite`)实跑,`database/sql` 的这套用法对 MySQL/PostgreSQL 一致。

---

## 1. ExecContext:不要结果行的 SQL

`INSERT`/`UPDATE`/`DELETE`/`CREATE TABLE` 这类"干活但不吐数据行"的 SQL,走 `ExecContext`:

```go
result, err := db.ExecContext(ctx,
    "INSERT INTO users(name, email) VALUES(?, ?)",
    "alice", "alice@example.com",
)
if err != nil {
    fmt.Println(err)
    return
}
```

它返回一个 `sql.Result`,里面只有两条信息——下两节的 `LastInsertId` 和 `RowsAffected`。

---

## 2. RowsAffected:这条 SQL 动了几行

```go
result, err := db.ExecContext(ctx,
    "UPDATE users SET name = ? WHERE id = ?", "bob", 999)
// ... err 检查省略

affected, _ := result.RowsAffected()
fmt.Println("affected =", affected)
if affected == 0 {
    fmt.Println("返回:", ErrNotFound)
}
```

```text
affected = 0
返回: not found
```

### 🕳️ 坑:更新 0 行不是错误

以为会怎样:UPDATE 一个不存在的 id,`err != nil`。
实际怎样:`err == nil`,SQL 执行得好好的——只是一行都没匹配上。
为什么:对数据库来说"动了 0 行"是完全合法的执行结果。想把它当"目标不存在"处理,必须自己查 `RowsAffected`。

**一句话总结:UPDATE/DELETE 后查一眼 RowsAffected,别让"改了个寂寞"静默通过。**

---

## 3. LastInsertId:刚插入那行的自增 id

```go
id, _ := result.LastInsertId()
affected, _ := result.RowsAffected()
fmt.Println("LastInsertId =", id, "RowsAffected =", affected)
```

```text
LastInsertId = 1 RowsAffected = 1
```

对应 `mysql2` 返回结果里的 `insertId`。但注意:**不是所有数据库都支持**——PostgreSQL 的驱动就不走这条路,惯用法是让 INSERT 自己把 id 吐出来:

```sql
INSERT INTO users(name) VALUES($1) RETURNING id
```

然后用下一节的 `QueryRowContext(...).Scan(&id)` 接住。

---

## 4. QueryRowContext:查一行,错误延迟到 Scan

```go
var user User
err := db.QueryRowContext(ctx,
    "SELECT id, name, email FROM users WHERE id = ?",
    id,
).Scan(&user.ID, &user.Name, &user.Email)
```

拆开这个链式调用:

```text
db.QueryRowContext(ctx, sql, id).Scan(&user.ID, &user.Name, &user.Email)
// ────────────┬───────────────  ──┬────────────────────────────────────
//  ①返回 *sql.Row,注意:不返回 err   ②真正取数、填变量、报错都在这一步
```

`Scan` 传的是**指针**(`&`)——你先备好变量,它把每一列的值写进去,顺序一一对应。这是 Go 和 `mysql2` 直接给你返回对象数组最大的手感差异:没有自动映射,列到字段的连线由你亲手焊。

`QueryRowContext` 本身不返回错误,所有问题都攒到 `Scan` 才爆:

```go
row := db.QueryRowContext(ctx, "SELECT name FROM no_such_table WHERE id = ?", 1)
fmt.Println("拿到 row 了,还没报错")

var name string
err := row.Scan(&name)
fmt.Println("Scan 时才报错:", err)
```

```text
拿到 row 了,还没报错
Scan 时才报错: SQL logic error: no such table: no_such_table (1)
```

为什么这样设计:为了让你能写成一行链式调用,`QueryRow` 把错误存进 `Row` 里、由 `Scan` 统一交付。所以**永远别忽略 Scan 的返回值**。

---

## 5. sql.ErrNoRows:没查到是业务分支,不是故障

查一行但一行都没有时,`Scan` 返回专门的哨兵错误 `sql.ErrNoRows`(03 模块学过的 sentinel error,用 `errors.Is` 认):

```go
var name string
err := db.QueryRowContext(ctx,
    "SELECT name FROM users WHERE id = ?", 42).Scan(&name)

if errors.Is(err, sql.ErrNoRows) {
    fmt.Println("没查到,转成业务错误:", ErrNotFound)
    return
}
if err != nil {
    fmt.Println("真正的数据库故障:", err)
    return
}
```

```text
没查到,转成业务错误: not found
```

"用户不存在"该给 404,数据库连不上才是 500——两条路必须分开处理。JS 对比:`mysql2` 查不到返回空数组 `[]`,你判 `rows.length === 0`;Go 把这件事做成了一个明确的错误值。

---

## 6. QueryContext:查多行,必须关

```go
rows, err := db.QueryContext(ctx,
    "SELECT id, name, email FROM users ORDER BY id",
)
if err != nil {
    return nil, err
}
defer rows.Close()
```

`Rows` 在遍历期间**一直占着池子里的一个连接**,忘记 Close 的代价是连接慢性泄漏、池子越跑越空(第 05 篇细讲)。标准姿势:err 检查之后立刻 `defer rows.Close()`——defer 篇的"先确认资源到手,再预约清理"。

---

## 7. 遍历 Rows:Next、Scan、Err 三件套

```go
var users []User
for rows.Next() {
    var u User
    if err := rows.Scan(&u.ID, &u.Name, &u.Email); err != nil {
        return nil, err
    }
    users = append(users, u)
}
if err := rows.Err(); err != nil {
    return nil, err
}
return users, nil
```

```text
[{1 alice alice@example.com} {2 bob bob@example.com}]
```

🕳️ 坑:循环结束后那句 `rows.Err()` 别省。`rows.Next()` 返回 false 有两种可能——"数据取完了"和"半路出错了(比如网络断了)",长得一模一样。不查 `rows.Err()`,你会把"只取到一半的数据"当成完整结果返回。

**一句话总结:多行查询的完整仪式 = defer Close + for Next + Scan + 最后 Err()。**

---

## 8. 🕳️ 坑:Scan 的目标必须和列一一对应

查了 3 列、只给 2 个目标变量:

```go
var id int64
var name string
err := db.QueryRowContext(ctx,
    "SELECT id, name, email FROM users WHERE id = 1",
).Scan(&id, &name) // 查了 3 列,只给了 2 个目标
fmt.Println(err)
```

```text
sql: expected 3 destination arguments in Scan, not 2
```

数量不对当场报错还算友好;更阴的是**数量对但顺序错**——`name` 扫进 email 变量,类型都是 string,一声不响地数据错位。所以:改了 SELECT 的列,必须同步改 Scan 的目标列表,两边像拉链一样对齐。

---

## 9. 顺手别用 SELECT *

```sql
SELECT * FROM users            -- ❌ 列的数量和顺序交给了表结构
SELECT id, name, email FROM users  -- ✅ 列由这行 SQL 自己锁死
```

用 `SELECT *`,哪天表加一列,所有 Scan 立刻集体报"expected N destination arguments"。明确列名,让 SQL 和 Scan 的对应关系写在同一处、看得见。

---

## 10. 错误包装:带上操作上下文

```go
if err != nil {
    return fmt.Errorf("query user %d: %w", id, err)
}
```

03 模块的 `%w` 包装术直接沿用:报错时能看出"哪个操作、哪个 id"。但注意分寸——**不要把密码、token 这类敏感参数写进错误信息**,错误最终可能进日志、进监控、进告警群。

---

## 本篇重点

- [ ] 三个动词按需选:`ExecContext`(改数据)、`QueryRowContext`(一行)、`QueryContext`(多行);`Scan` 用指针逐列接收,没有自动映射。
- [ ] UPDATE/DELETE 动了 0 行不算 err,要自己查 `RowsAffected`;`LastInsertId` 在 PostgreSQL 用 `RETURNING id` 替代。
- [ ] `QueryRow` 的错误延迟到 `Scan` 才返回;没查到是 `sql.ErrNoRows`,用 `errors.Is` 认出来转成业务错误。
- [ ] 多行仪式:defer Close → for Next → Scan → 最后必查 `rows.Err()`。
- [ ] SELECT 列和 Scan 目标像拉链一样一一对应,别用 `SELECT *`。

---

## 练习

1. 用 `ExecContext` 插入一条用户记录。
2. 用 `RowsAffected` 判断更新是否命中。
3. 用 `QueryRowContext` 查询单个用户,并处理 `sql.ErrNoRows`。
4. 用 `QueryContext` 查询用户列表,关闭 rows 并检查 `rows.Err()`。
5. 解释 `QueryRowContext` 为什么通常在 `Scan` 时才返回错误。

提示:练习 1-4 可以共用一个 SQLite 内存库(`sql.Open("sqlite", ":memory:")`)和一张 users 表串起来做;练习 5 回看第 4 节的拆解图,想想"链式调用"和"错误往哪放"的关系。
