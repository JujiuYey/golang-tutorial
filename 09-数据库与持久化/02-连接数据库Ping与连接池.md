# 02: 连接数据库、Ping 与连接池配置

数据库连接需要在程序启动时创建，并在退出时关闭。`*sql.DB` 内部是连接池，连接池参数会影响稳定性和性能。

---

## 1. 打开数据库

SQLite 示例：

```go
db, err := sql.Open("sqlite", "file:app.db")
if err != nil {
    return nil, fmt.Errorf("open db: %w", err)
}
```

MySQL 示例：

```go
dsn := "user:password@tcp(localhost:3306)/app?parseTime=true"
db, err := sql.Open("mysql", dsn)
```

PostgreSQL 示例：

```go
dsn := "postgres://user:password@localhost:5432/app?sslmode=disable"
db, err := sql.Open("postgres", dsn)
```

DSN 不要硬编码在源码里，实际项目中从配置或环境变量读取。

---

## 2. PingContext

```go
ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
defer cancel()

if err := db.PingContext(ctx); err != nil {
    db.Close()
    return nil, fmt.Errorf("ping db: %w", err)
}
```

`PingContext` 用来确认数据库真的可达。

---

## 3. 关闭数据库

```go
defer db.Close()
```

`Close` 会关闭连接池，通常在程序退出时执行。不要在每个请求结束时关闭全局 db。

---

## 4. 连接池参数

```go
db.SetMaxOpenConns(25)
db.SetMaxIdleConns(25)
db.SetConnMaxLifetime(5 * time.Minute)
db.SetConnMaxIdleTime(1 * time.Minute)
```

含义：

- `MaxOpenConns`: 最大打开连接数。
- `MaxIdleConns`: 最大空闲连接数。
- `ConnMaxLifetime`: 单个连接最多复用多久。
- `ConnMaxIdleTime`: 空闲连接最多保留多久。

---

## 5. 连接池不是越大越好

连接数过大可能压垮数据库。连接数过小可能让请求排队等待连接。

设置连接池时要考虑：

- 数据库允许的最大连接数。
- 服务实例数量。
- 每个请求持有连接多久。
- 慢查询数量。
- 事务是否长时间占用连接。

---

## 6. db.Stats

```go
stats := db.Stats()
fmt.Println(stats.OpenConnections)
fmt.Println(stats.InUse)
fmt.Println(stats.Idle)
fmt.Println(stats.WaitCount)
```

`Stats` 可以帮助观察连接池行为。生产环境通常把这些指标接入监控。

---

## 7. 配置结构

```go
type DBConfig struct {
    Driver          string
    DSN             string
    MaxOpenConns    int
    MaxIdleConns    int
    ConnMaxLifetime time.Duration
}
```

把配置集中起来，便于测试和部署。

---

## 8. 构造函数

```go
func OpenDB(ctx context.Context, cfg DBConfig) (*sql.DB, error) {
    db, err := sql.Open(cfg.Driver, cfg.DSN)
    if err != nil {
        return nil, fmt.Errorf("open db: %w", err)
    }

    db.SetMaxOpenConns(cfg.MaxOpenConns)
    db.SetMaxIdleConns(cfg.MaxIdleConns)
    db.SetConnMaxLifetime(cfg.ConnMaxLifetime)

    if err := db.PingContext(ctx); err != nil {
        db.Close()
        return nil, fmt.Errorf("ping db: %w", err)
    }
    return db, nil
}
```

打开失败时要关闭已经创建的 db 句柄。

---

## 9. 环境变量

```go
dsn := os.Getenv("DATABASE_URL")
if dsn == "" {
    return errors.New("DATABASE_URL is required")
}
```

数据库密码、地址、库名通常来自环境变量或配置文件，不要写进代码仓库。

---

## 10. 启动检查

服务启动时至少要确认：

1. DSN 存在。
2. 驱动名正确。
3. `PingContext` 成功。
4. 连接池参数合理。
5. 退出时会调用 `db.Close()`。

---

## 练习

1. 写 `OpenDB(ctx, cfg)` 函数。
2. 加上 `PingContext` 和 3 秒超时。
3. 配置连接池参数。
4. 打印 `db.Stats()`。
5. 从 `DATABASE_URL` 读取 DSN。
