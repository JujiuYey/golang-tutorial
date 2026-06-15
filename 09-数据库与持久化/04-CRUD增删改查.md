# 04: CRUD：增删改查的基本写法

CRUD 是数据库代码的基本功。写 CRUD 时要关注 SQL 参数、错误处理、影响行数和空结果。

---

## 1. User 模型

```go
type User struct {
    ID        int64
    Name      string
    Email     string
    CreatedAt time.Time
}
```

这是普通 Go struct。`database/sql` 不依赖 struct tag 自动映射，需要手动 Scan。

---

## 2. 创建表

SQLite 示例：

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

生产项目通常使用迁移工具管理 schema。

---

## 3. Create

SQLite 或 MySQL 常见写法：

```go
func CreateUser(ctx context.Context, db *sql.DB, name, email string) (int64, error) {
    result, err := db.ExecContext(ctx,
        "INSERT INTO users(name, email, created_at) VALUES(?, ?, ?)",
        name,
        email,
        time.Now(),
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

PostgreSQL 常用 `RETURNING id`。

---

## 4. Get

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

`ErrNotFound` 可以是你自己定义的业务错误：

```go
var ErrNotFound = errors.New("not found")
```

---

## 5. List

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

分页参数要做范围校验。

---

## 6. Update

```go
func UpdateUserEmail(ctx context.Context, db *sql.DB, id int64, email string) error {
    result, err := db.ExecContext(ctx,
        "UPDATE users SET email = ? WHERE id = ?",
        email,
        id,
    )
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

更新时检查影响行数，避免悄悄更新 0 行。

---

## 7. Delete

```go
func DeleteUser(ctx context.Context, db *sql.DB, id int64) error {
    result, err := db.ExecContext(ctx,
        "DELETE FROM users WHERE id = ?",
        id,
    )
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

---

## 8. Count

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

`COUNT(*)` 查询一行，用 `QueryRowContext`。

---

## 9. 唯一约束错误

插入重复 email 时，数据库会返回驱动相关错误。不同驱动错误类型不同。

处理方式通常有两层：

1. 数据库层设置唯一约束。
2. 应用层把驱动错误转换成业务错误，比如 `ErrEmailExists`。

不要只靠应用层先查再插入来保证唯一性，因为并发下会有竞态。

---

## 10. CRUD 检查

1. 是否使用 context。
2. SQL 是否参数化。
3. 是否处理 `ErrNoRows`。
4. 多行查询是否关闭 rows。
5. update/delete 是否检查影响行数。
6. 唯一性是否由数据库约束保证。

---

## 练习

1. 实现 `CreateUser`。
2. 实现 `GetUser`，没查到时返回 `ErrNotFound`。
3. 实现 `ListUsers`，支持 limit/offset。
4. 实现 `UpdateUserEmail`，检查影响行数。
5. 实现 `DeleteUser`，检查影响行数。
