# 08: log、slog 与结构化日志

日志用于记录程序运行过程。简单程序可以用 `log`，现代服务更常用 `log/slog` 写结构化日志。

---

## 1. fmt 和 log

`fmt.Println` 适合普通输出。

```go
fmt.Println("hello")
```

`log.Println` 会附带时间等日志信息。

```go
log.Println("server started")
```

命令行工具的正常结果通常用 stdout；诊断信息和错误可以用日志或 stderr。

---

## 2. log.Fatal 和 log.Panic

```go
log.Fatal(err)
```

`Fatal` 会打印日志后调用 `os.Exit(1)`。defer 不会执行。

```go
log.Panic(err)
```

`Panic` 会打印日志后 panic。

库代码不要随便 `log.Fatal`。库应该返回 error，让调用方决定如何处理。

---

## 3. 创建 logger

```go
logger := log.New(os.Stdout, "app: ", log.LstdFlags)
logger.Println("started")
```

这样可以控制输出位置、前缀和格式。

---

## 4. slog 基本用法

```go
slog.Info("server started", "addr", ":8080")
slog.Error("request failed", "err", err)
```

`slog` 支持结构化字段。字段是成对出现的 key/value。

---

## 5. JSON 日志

```go
logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
logger.Info("server started", "addr", ":8080")
```

JSON 日志适合被日志系统采集和查询。

文本日志：

```go
logger := slog.New(slog.NewTextHandler(os.Stdout, nil))
```

本地开发时文本日志更易读。

---

## 6. 日志级别

常见级别：

- Debug
- Info
- Warn
- Error

```go
logger.Info("user created", "id", user.ID)
logger.Warn("retrying", "attempt", 2)
logger.Error("save failed", "err", err)
```

日志级别不是装饰，要表达事件严重程度。

---

## 7. With

```go
logger := slog.Default().With("service", "api", "version", "1.0")
logger.Info("started")
```

`With` 会给后续日志附加公共字段。

请求级日志：

```go
reqLogger := logger.With("request_id", requestID)
reqLogger.Info("handling request")
```

---

## 8. 不要记录敏感信息

避免写入日志：

- 密码
- token
- 私钥
- 完整身份证号
- 用户隐私数据

日志经常会被集中收集和长期保存。能定位问题即可，不要把敏感数据写进去。

---

## 9. 错误日志和返回 error

在底层函数里通常返回 error，不立即打日志。

```go
func Save(user User) error {
    if err := store.Save(user); err != nil {
        return fmt.Errorf("save user %d: %w", user.ID, err)
    }
    return nil
}
```

在边界层统一记录：

```go
if err := Save(user); err != nil {
    logger.Error("request failed", "err", err)
}
```

这样可以避免同一个错误被重复记录很多次。

---

## 10. 日志字段命名

字段名要稳定：

```go
logger.Info("request completed",
    "method", "GET",
    "path", "/users",
    "status", 200,
    "duration_ms", 12,
)
```

稳定字段方便检索、聚合和告警。

---

## 练习

1. 用 `log.New` 创建一个带前缀的 logger。
2. 用 `slog.NewTextHandler` 输出文本日志。
3. 用 `slog.NewJSONHandler` 输出 JSON 日志。
4. 写一个函数返回 error，在调用边界用 slog 记录。
5. 给日志增加 `request_id` 公共字段。
