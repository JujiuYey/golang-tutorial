# 09: Repository 边界与测试替身

Repository 用来把业务代码和数据库访问隔开。这里关注的是边界清晰：调用方只依赖自己真正需要的数据库能力。

---

## 1. 具体实现

```go
type UserStore struct {
    db *sql.DB
}

func NewUserStore(db *sql.DB) *UserStore {
    return &UserStore{db: db}
}
```

构造函数返回具体类型，调用方可以使用它的完整能力。

---

## 2. 方法接收 context

```go
func (s *UserStore) FindByID(ctx context.Context, id int64) (User, error) {
    var user User
    err := s.db.QueryRowContext(ctx, `
        SELECT id, name, email
        FROM users
        WHERE id = ?
    `, id).Scan(&user.ID, &user.Name, &user.Email)
    if errors.Is(err, sql.ErrNoRows) {
        return User{}, ErrNotFound
    }
    if err != nil {
        return User{}, fmt.Errorf("find user %d: %w", id, err)
    }
    return user, nil
}
```

数据库访问方法应该接收调用方传入的 context。

---

## 3. 小接口由使用方定义

```go
type UserFinder interface {
    FindByID(ctx context.Context, id int64) (User, error)
}

func BuildProfile(ctx context.Context, finder UserFinder, id int64) (Profile, error) {
    user, err := finder.FindByID(ctx, id)
    if err != nil {
        return Profile{}, err
    }
    return Profile{Name: user.Name}, nil
}
```

`BuildProfile` 只需要查用户，所以定义小接口。

---

## 4. 测试替身

```go
type fakeUserFinder struct {
    user User
    err  error
}

func (f fakeUserFinder) FindByID(ctx context.Context, id int64) (User, error) {
    return f.user, f.err
}
```

业务逻辑测试可以用 fake，不必启动真实数据库。

---

## 5. Repository 不要吞掉错误语义

```go
if errors.Is(err, sql.ErrNoRows) {
    return User{}, ErrNotFound
}
```

把底层数据库错误转换成业务错误是有价值的。调用方不应该到处知道 `sql.ErrNoRows`。

---

## 6. 不要把 *sql.DB 到处传

业务函数如果到处直接接收 `*sql.DB`，SQL 会散落在各处，测试和维护都会变难。

更清晰的做法是把 SQL 放在 store/repository 里，业务层依赖小接口。

---

## 7. 事务和 Repository

事务中需要多个 repository 共享同一个 `*sql.Tx`。可以定义一个小接口抽象共同方法：

```go
type DBTX interface {
    ExecContext(context.Context, string, ...any) (sql.Result, error)
    QueryContext(context.Context, string, ...any) (*sql.Rows, error)
    QueryRowContext(context.Context, string, ...any) *sql.Row
}
```

`*sql.DB` 和 `*sql.Tx` 都能满足类似接口。

```go
type UserStore struct {
    db DBTX
}
```

这样同一套查询方法可以在普通连接和事务中使用。

---

## 8. Repository 不等于贫血转发层

如果方法只是把 SQL 原样暴露出去，边界价值不大。

有价值的 repository 应该封装：

- SQL 细节。
- 错误转换。
- Scan 映射。
- 查询约束。
- 事务参与方式。

---

## 9. 命名

常见命名：

- `UserStore`
- `UserRepository`
- `UserDAO`

Go 里没有强制术语。选择一个项目内统一使用即可。

---

## 10. 边界检查

1. 调用方是否依赖小接口。
2. SQL 是否集中在数据访问层。
3. 错误是否转换成业务语义。
4. 是否支持 context。
5. 事务中是否能复用方法。

---

## 练习

1. 实现 `UserStore.FindByID`。
2. 定义 `UserFinder` 小接口。
3. 用 fake 测试 `BuildProfile`。
4. 定义 `DBTX` 接口，让 store 支持 `*sql.DB` 和 `*sql.Tx`。
5. 说明哪些错误应该在 repository 中转换。
