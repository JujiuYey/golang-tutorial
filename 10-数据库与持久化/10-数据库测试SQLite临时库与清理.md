# 10: 数据库测试：SQLite、临时库与清理

数据库代码需要测试。测试可以使用临时 SQLite 数据库，也可以在集成测试中使用真实数据库。课程阶段先用 SQLite 建立方法。

---

## 1. 使用 t.TempDir

```go
func openTestDB(t *testing.T) *sql.DB {
    t.Helper()

    path := filepath.Join(t.TempDir(), "test.db")
    db, err := sql.Open("sqlite", "file:"+path)
    if err != nil {
        t.Fatal(err)
    }
    t.Cleanup(func() {
        db.Close()
    })
    return db
}
```

每个测试使用独立临时目录，互不影响。

---

## 2. 内存数据库

SQLite 可以使用内存数据库：

```go
db, err := sql.Open("sqlite", ":memory:")
```

注意：SQLite 内存库和连接有关。`database/sql` 连接池可能打开多个连接，导致不同连接看到的内存库不是同一份。测试时可以限制：

```go
db.SetMaxOpenConns(1)
```

---

## 3. 测试前迁移

```go
db := openTestDB(t)
if err := Migrate(context.Background(), db); err != nil {
    t.Fatal(err)
}
```

测试应该自己准备 schema，不依赖开发机已有数据库状态。

---

## 4. 插入测试数据

```go
_, err := db.ExecContext(ctx,
    "INSERT INTO users(name, email, created_at) VALUES(?, ?, ?)",
    "alice",
    "alice@example.com",
    time.Now(),
)
if err != nil {
    t.Fatal(err)
}
```

测试数据要尽量少而明确。

---

## 5. 测试 ErrNotFound

```go
_, err := store.FindByID(ctx, 999)
if !errors.Is(err, ErrNotFound) {
    t.Fatalf("err = %v, want ErrNotFound", err)
}
```

错误路径和正常路径一样重要。

---

## 6. 测试唯一约束

```go
_, err := db.ExecContext(ctx,
    "INSERT INTO users(name, email) VALUES(?, ?)",
    "alice",
    "same@example.com",
)
```

重复插入同一个 email，验证数据库约束是否生效。

---

## 7. 测试事务回滚

```go
err := RunInTx(ctx, db, func(tx *sql.Tx) error {
    if _, err := tx.ExecContext(ctx, "INSERT INTO users(name) VALUES(?)", "alice"); err != nil {
        return err
    }
    return errors.New("force rollback")
})
if err == nil {
    t.Fatal("expected error")
}
```

再查询 users，确认插入没有保留。

---

## 8. 并发测试和 race

数据库本身能处理并发连接，但你的 Go 代码仍可能有数据竞争。

运行：

```bash
go test -race ./...
```

如果 store 内部有共享缓存、计数器或 map，要特别注意同步。

---

## 9. 测试真实数据库

真实 MySQL/PostgreSQL 测试更接近生产，但成本更高：

- 需要启动服务。
- 需要清理数据。
- 需要管理连接配置。
- CI 配置更复杂。

可以把真实数据库测试标成集成测试，在需要时运行。

---

## 10. 数据库测试检查

1. 每个测试是否隔离。
2. schema 是否由测试准备。
3. 测试数据是否明确。
4. 正常路径和错误路径是否覆盖。
5. 是否运行 race detector。

---

## 练习

1. 写 `openTestDB(t)`。
2. 写 `Migrate(ctx, db)` 并在测试中调用。
3. 测试 `CreateUser` 和 `GetUser`。
4. 测试不存在用户返回 `ErrNotFound`。
5. 测试事务回滚。
