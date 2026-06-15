# 08: 迁移、schema 与索引

数据库 schema 会随项目演进。迁移用于把数据库结构从一个版本变到另一个版本。

---

## 1. schema 是什么

schema 包括：

- 表。
- 列。
- 类型。
- 主键。
- 外键。
- 索引。
- 唯一约束。
- 默认值。

Go 代码和数据库 schema 必须匹配。

---

## 2. 迁移是什么

迁移通常是一组按顺序执行的 SQL 文件：

```text
001_create_users.up.sql
001_create_users.down.sql
002_add_users_email_index.up.sql
002_add_users_email_index.down.sql
```

up 表示应用迁移，down 表示回滚迁移。

---

## 3. 创建表

```sql
CREATE TABLE users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    email TEXT NOT NULL UNIQUE,
    created_at TIMESTAMP NOT NULL
);
```

字段约束应该尽量放到数据库层：

- `NOT NULL`
- `UNIQUE`
- `CHECK`
- `FOREIGN KEY`

应用层校验提升用户体验，数据库约束保证最终一致性。

---

## 4. 索引

```sql
CREATE INDEX idx_users_created_at ON users(created_at);
```

索引用于加速查询，但也会增加写入成本和存储成本。

为常用过滤条件、排序字段、关联字段建立索引。

---

## 5. 唯一索引

```sql
CREATE UNIQUE INDEX idx_users_email ON users(email);
```

唯一性必须由数据库保证。应用层先查再插入无法避免并发竞态。

---

## 6. 外键

```sql
CREATE TABLE posts (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    author_id INTEGER NOT NULL,
    title TEXT NOT NULL,
    FOREIGN KEY(author_id) REFERENCES users(id)
);
```

外键能保证引用关系，但也会影响写入和删除策略。是否启用要根据项目一致性需求决定。

---

## 7. 迁移不要随意修改历史

已经在共享环境执行过的迁移，不要直接改历史文件。新增迁移修正问题。

原因：

- 不同环境可能已经执行旧迁移。
- 修改历史会导致环境状态不一致。
- 回滚和审计会变困难。

---

## 8. Go 里执行简单迁移

教学项目可以直接执行 SQL：

```go
func Migrate(ctx context.Context, db *sql.DB) error {
    _, err := db.ExecContext(ctx, `
        CREATE TABLE IF NOT EXISTS users (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT NOT NULL,
            email TEXT NOT NULL UNIQUE,
            created_at TIMESTAMP NOT NULL
        );
    `)
    if err != nil {
        return fmt.Errorf("migrate users: %w", err)
    }
    return nil
}
```

正式项目建议使用迁移工具管理版本。

---

## 9. schema 和代码同步

如果 SQL 查询：

```sql
SELECT id, name, email FROM users
```

Go struct 和 Scan 要对应：

```go
type User struct {
    ID    int64
    Name  string
    Email string
}
```

schema 改了，查询、Scan、测试都要跟着改。

---

## 10. 迁移检查

1. 是否有主键。
2. 必填字段是否 `NOT NULL`。
3. 唯一性是否有数据库约束。
4. 常用查询是否有合适索引。
5. 迁移是否可重复、可追踪。

---

## 练习

1. 写 `users` 表迁移。
2. 给 `email` 加唯一索引。
3. 给 `created_at` 加普通索引。
4. 写 `posts` 表，并添加 `author_id` 外键。
5. 写一个 Go 函数执行教学项目里的简单迁移。
