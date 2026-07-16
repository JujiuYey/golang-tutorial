# 04: CRUD:增删改查的基本写法

这篇把上一篇的三个动词组装成一套完整的 CRUD 函数——相当于你在 Node 里用 Prisma 时 `user.create/findUnique/findMany/update/delete` 自动获得的那五件套,在 Go 里手写一遍。学完你有一份可以直接抄进项目的模板,并且知道每个函数里哪几行是"不能省的保命检查"。

> 本篇代码用 SQLite(`modernc.org/sqlite`)完整实跑,文末输出全部来自真实运行。换 MySQL 只需换驱动、DSN 和建表 SQL 方言。

---

## 1. User 模型:就是普通 struct

```go
type User struct {
    ID        int64
    Name      string
    Email     string
    CreatedAt time.Time
}
```

注意没有任何 tag、注解、装饰器。`database/sql` 不做自动映射——列到字段的连线全靠你在 `Scan` 里手焊(上一篇的拉链规则)。Prisma 用户可能觉得原始,但好处是**每一列去了哪里,代码里明明白白**。

---

## 2. 建表

```go
const createUsersTable = `
CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    email TEXT NOT NULL UNIQUE,
    created_at TIMESTAMP NOT NULL
);`

if _, err := db.ExecContext(ctx, createUsersTable); err != nil {
    return fmt.Errorf("create users table: %w", err)
}
```

建表 SQL 用反引号字符串(02 模块:原样多行用反引号)。`email` 上的 `UNIQUE` 不是装饰,第 9 节它要出场干活。生产项目的 schema 用迁移工具管(第 08 篇),教学阶段直接 Exec 够用。

---

## 3. Create:插入并拿回 id

```go
func CreateUser(ctx context.Context, db *sql.DB, name, email string) (int64, error) {
    result, err := db.ExecContext(ctx,
        "INSERT INTO users(name, email, created_at) VALUES(?, ?, ?)",
        name, email, time.Now(),
    )
    if err != nil {
        return 0, fmt.Errorf("insert user: %w", err)
    }

    id, err := result.LastInsertId()
    if err != nil {
        return 0, fmt.Errorf("last insert id: %w", err)
    }
    return id, nil
}
```

```text
Create: id = 1 err = <nil>
```

套路:参数化 INSERT → `LastInsertId` 拿自增 id。PostgreSQL 走 `INSERT ... RETURNING id` + `QueryRowContext`(上一篇说过)。

---

## 4. Get:查一行,区分"没有"和"坏了"

```go
func GetUser(ctx context.Context, db *sql.DB, id int64) (User, error) {
    var user User
    err := db.QueryRowContext(ctx, `
        SELECT id, name, email, created_at
        FROM users
        WHERE id = ?
    `, id).Scan(&user.ID, &user.Name, &user.Email, &user.CreatedAt)
    if errors.Is(err, sql.ErrNoRows) {
        return User{}, ErrNotFound
    }
    if err != nil {
        return User{}, fmt.Errorf("get user %d: %w", id, err)
    }
    return user, nil
}
```

```text
Get: alice alice@example.com err = <nil>
Get 999: err = not found
```

`ErrNotFound` 是你自己定义的业务哨兵错误:

```go
var ErrNotFound = errors.New("not found")
```

在这里把 `sql.ErrNoRows` 翻译成 `ErrNotFound`,调用方(HTTP handler、业务层)就不需要 import `database/sql` 也能判断"用户不存在"——数据库的方言止步于此层(第 09 篇会把这个思路展开成 Repository)。

---

## 5. List:多行 + 分页

```go
func ListUsers(ctx context.Context, db *sql.DB, limit, offset int) ([]User, error) {
    rows, err := db.QueryContext(ctx, `
        SELECT id, name, email, created_at
        FROM users
        ORDER BY id
        LIMIT ? OFFSET ?
    `, limit, offset)
    if err != nil {
        return nil, fmt.Errorf("list users: %w", err)
    }
    defer rows.Close()

    var users []User
    for rows.Next() {
        var user User
        if err := rows.Scan(&user.ID, &user.Name, &user.Email, &user.CreatedAt); err != nil {
            return nil, fmt.Errorf("scan user: %w", err)
        }
        users = append(users, user)
    }
    if err := rows.Err(); err != nil {
        return nil, fmt.Errorf("iterate users: %w", err)
    }
    return users, nil
}
```

```text
List: 2 个用户
```

上一篇的多行仪式原样上岗:defer Close → for Next → Scan → 最后 `rows.Err()`。两个补充:

- `ORDER BY id`——分页没有稳定排序等于每页随机发牌。
- `limit`/`offset` 来自用户输入时要做范围校验(上限、非负),别让人一把 `limit=1000000` 拖垮库。

---

## 6. Update:必须查影响行数

```go
func UpdateUserEmail(ctx context.Context, db *sql.DB, id int64, email string) error {
    result, err := db.ExecContext(ctx,
        "UPDATE users SET email = ? WHERE id = ?", email, id)
    if err != nil {
        return fmt.Errorf("update user email: %w", err)
    }

    affected, err := result.RowsAffected()
    if err != nil {
        return fmt.Errorf("rows affected: %w", err)
    }
    if affected == 0 {
        return ErrNotFound
    }
    return nil
}
```

```text
Update: <nil>
Update 999: not found
```

上一篇的坑在这里落地:更新不存在的 id,`Exec` 本身**不报错**。不查 `RowsAffected`,接口会对着一个不存在的用户返回 200 OK。

---

## 7. Delete:同款套路

```go
func DeleteUser(ctx context.Context, db *sql.DB, id int64) error {
    result, err := db.ExecContext(ctx,
        "DELETE FROM users WHERE id = ?", id)
    if err != nil {
        return fmt.Errorf("delete user: %w", err)
    }

    affected, err := result.RowsAffected()
    if err != nil {
        return fmt.Errorf("rows affected: %w", err)
    }
    if affected == 0 {
        return ErrNotFound
    }
    return nil
}
```

```text
Delete: <nil>
```

和 Update 一个模子:Exec → RowsAffected → 0 行转 `ErrNotFound`。

---

## 8. Count:聚合也是"查一行"

```go
func CountUsers(ctx context.Context, db *sql.DB) (int64, error) {
    var count int64
    err := db.QueryRowContext(ctx, "SELECT COUNT(*) FROM users").Scan(&count)
    if err != nil {
        return 0, fmt.Errorf("count users: %w", err)
    }
    return count, nil
}
```

```text
Count: 1
```

(alice 已被上一节删掉,只剩 bob。)`COUNT(*)` 永远返回恰好一行,所以走 `QueryRowContext`——按"返回几行"选动词,而不是按"是不是 SELECT"。

---

## 9. 唯一约束:让数据库当守门员

给第 3 节的 Create 喂一个重复 email:

```go
_, err = CreateUser(ctx, db, "alice2", "alice@example.com")
fmt.Println("重复 email:", err)
```

```text
重复 email: insert user: constraint failed: UNIQUE constraint failed: users.email (2067)
```

错误原文是**驱动相关**的(MySQL 报 `Error 1062: Duplicate entry`,长相完全不同)。处理分两层:

1. 数据库层:`UNIQUE` 约束兜底(第 2 节建表时埋的)。
2. 应用层:把驱动错误翻译成业务错误,比如 `ErrEmailExists`,别让驱动方言漏到上层。

### 🕳️ 坑:"先查再插"防不住重复

以为会怎样:插入前先 SELECT 一下,没有同名 email 再 INSERT,万无一失。
实际怎样:两个并发请求同时通过了 SELECT 检查,然后双双 INSERT。
为什么:查和插是两步,中间有时间窗(07 模块的竞态老朋友)。**唯一性必须由数据库约束保证**,应用层检查只是提前给用户友好提示。

---

## 10. CRUD 检查清单

写完一套 CRUD,过一遍:

1. 是否使用 context。
2. SQL 是否参数化。
3. 是否处理 `ErrNoRows`(转成业务错误)。
4. 多行查询是否关闭 rows、检查 `rows.Err()`。
5. update/delete 是否检查影响行数。
6. 唯一性是否由数据库约束保证。

---

## 本篇重点

- [ ] `database/sql` 没有自动映射:struct 是普通 struct,列和字段的对应关系全在 `Scan` 里。
- [ ] Get 类函数把 `sql.ErrNoRows` 翻译成自己的 `ErrNotFound`,别让 `database/sql` 的细节漏到业务层。
- [ ] Update/Delete 必查 `RowsAffected`,0 行 = 目标不存在,而不是成功。
- [ ] List 要稳定排序 + 校验分页参数;Count 是"恰好一行",走 `QueryRowContext`。
- [ ] 唯一性靠数据库 `UNIQUE` 约束,"先查再插"在并发下必翻车。

---

## 练习

1. 实现 `CreateUser`。
2. 实现 `GetUser`,没查到时返回 `ErrNotFound`。
3. 实现 `ListUsers`,支持 limit/offset。
4. 实现 `UpdateUserEmail`,检查影响行数。
5. 实现 `DeleteUser`,检查影响行数。

提示:五个函数放进同一个文件,`main` 里按"建表 → Create → Get → List → Update → Delete"串一遍并打印每步结果;顺手试一次重复 email 和不存在的 id,看错误路径是否符合预期。
