# 07: 事务：BeginTx、Commit、Rollback

事务用于保证一组数据库操作要么全部成功，要么全部失败。转账、订单创建、库存扣减等场景都需要事务。

---

## 1. 开启事务

```go
tx, err := db.BeginTx(ctx, nil)
if err != nil {
    return err
}
```

`BeginTx` 会从连接池中拿出一个连接，并在这个连接上执行事务。

---

## 2. Rollback 兜底

```go
tx, err := db.BeginTx(ctx, nil)
if err != nil {
    return err
}
defer tx.Rollback()
```

即使后面 commit 成功，defer 的 rollback 也会返回错误，通常可以忽略。这个兜底可以保证中途 return 时事务不会悬挂。

---

## 3. Commit

```go
if err := tx.Commit(); err != nil {
    return fmt.Errorf("commit: %w", err)
}
```

`Commit` 也可能失败。不要忽略它。

---

## 4. 事务中的操作

事务里使用 `tx.ExecContext`、`tx.QueryRowContext`、`tx.QueryContext`：

```go
_, err = tx.ExecContext(ctx,
    "UPDATE accounts SET balance = balance - ? WHERE id = ?",
    amount,
    fromID,
)
```

不要在事务中混用 `db.ExecContext`，否则操作可能跑到事务外。

---

## 5. 转账示例

```go
func Transfer(ctx context.Context, db *sql.DB, fromID, toID int64, amount int64) error {
    tx, err := db.BeginTx(ctx, nil)
    if err != nil {
        return err
    }
    defer tx.Rollback()

    if _, err := tx.ExecContext(ctx,
        "UPDATE accounts SET balance = balance - ? WHERE id = ?",
        amount, fromID,
    ); err != nil {
        return fmt.Errorf("debit account: %w", err)
    }

    if _, err := tx.ExecContext(ctx,
        "UPDATE accounts SET balance = balance + ? WHERE id = ?",
        amount, toID,
    ); err != nil {
        return fmt.Errorf("credit account: %w", err)
    }

    if err := tx.Commit(); err != nil {
        return fmt.Errorf("commit transfer: %w", err)
    }
    return nil
}
```

真实转账还要检查余额、账户存在性、并发一致性。

---

## 6. 事务选项

```go
tx, err := db.BeginTx(ctx, &sql.TxOptions{
    Isolation: sql.LevelSerializable,
    ReadOnly:  false,
})
```

隔离级别支持情况取决于数据库。不要随便调高隔离级别，可能影响并发性能。

---

## 7. 事务不要太长

事务会占用连接和数据库锁。事务里不要做：

- 网络请求。
- 用户交互等待。
- 大量慢计算。
- 不必要的文件 IO。

把事务范围控制在必要的数据库操作内。

---

## 8. 封装事务函数

```go
func RunInTx(ctx context.Context, db *sql.DB, fn func(*sql.Tx) error) error {
    tx, err := db.BeginTx(ctx, nil)
    if err != nil {
        return err
    }
    defer tx.Rollback()

    if err := fn(tx); err != nil {
        return err
    }
    if err := tx.Commit(); err != nil {
        return fmt.Errorf("commit tx: %w", err)
    }
    return nil
}
```

这种封装可以减少重复的 begin/rollback/commit 样板。

---

## 9. Commit 后不要继续使用 tx

事务 commit 或 rollback 后，`tx` 就结束了。继续使用会报错。

如果后续还要查询，用 `db` 开新的操作。

---

## 10. 事务检查

1. 是否使用 `BeginTx`。
2. 是否 defer rollback 兜底。
3. 是否检查 commit 错误。
4. 事务内是否全部使用 tx。
5. 事务范围是否足够短。

---

## 练习

1. 实现 `RunInTx`。
2. 实现账户转账。
3. 故意让第二个更新失败，验证第一个更新会回滚。
4. 给事务设置只读选项。
5. 解释为什么事务里不应该做网络请求。
