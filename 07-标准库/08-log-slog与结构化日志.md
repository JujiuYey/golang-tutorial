# 08: log、slog 与结构化日志

这篇解决"程序跑起来之后,怎么知道它在干什么":简单场景用 `log`,正经服务用 `log/slog` 写**结构化日志**(每条日志是一组 key=value,机器能检索,不是一坨拼接字符串)。学完你能躲开两个坑:`log.Fatal` 会跳过所有 defer;slog 的 Debug 日志默认根本不输出。

对 JS 用户:`log` ≈ 加了时间戳的 `console.log`,`slog` ≈ winston/pino 的标准库版。

注:本篇所有输出里的时间戳是运行时刻,**你的输出时间会不同**。

---

## 1. fmt 和 log 的分工

```go
func main() {
    fmt.Println("hello")
    log.Println("server started")
}
```

```text
hello
2026/07/16 10:24:31 server started
```

区别一眼可见:`log` 自动带时间戳。分工原则:

- `fmt.Println` → stdout:程序的**正常结果**(用户要的数据)。
- `log.Println` → **stderr**:运行过程的**诊断信息**。

这正好接上上一篇第 9 节:结果和诊断分流,重定向和管道才干净。

---

## 2. log.Fatal 和 log.Panic

```go
log.Fatal(err) // 打印日志,然后 os.Exit(1)
log.Panic(err) // 打印日志,然后 panic
```

### 🕳️ 坑:Fatal 会跳过所有 defer

以为会怎样:程序退出前,defer 的清理照常执行(03-07 说好的"无论哪条路退出")。
实际怎样:

```go
func main() {
    defer fmt.Println("清理工作") // 你以为它会执行
    log.Println("before")
    log.Fatal("boom")
    fmt.Println("after") // 永远到不了
}
```

```text
2026/07/16 10:24:36 before
2026/07/16 10:24:36 boom
```

"清理工作"没打出来(进程退出码 1)。为什么:`Fatal` 内部是 `os.Exit(1)`——直接掐断进程,**defer 没有机会运行**。文件没 Flush、连接没还,全都来不及。

所以纪律是:**库代码永远返回 error,不许 `log.Fatal`**;`Fatal` 只允许出现在 `main` 的最外层,那里已经没有需要清理的东西。

---

## 3. 自己造 logger:log.New

```go
func main() {
    logger := log.New(os.Stdout, "app: ", log.LstdFlags)
    logger.Println("started")
}
```

```text
app: 2026/07/16 10:24:31 started
```

三个参数:写到哪(`io.Writer`——又是通用插座,可以是文件、buffer)、每行前缀、格式标志。测试时把第一个参数换成 `bytes.Buffer`,日志内容就能断言了。

---

## 4. slog:结构化日志

`log` 输出的是自由文本;`slog` 让每条日志变成"消息 + 成对的 key/value":

```go
func main() {
    slog.Info("server started", "addr", ":8080")
    slog.Warn("retrying", "attempt", 2)
    slog.Error("request failed", "err", "connection refused")
}
```

```text
2026/07/16 10:24:32 INFO server started addr=:8080
2026/07/16 10:24:32 WARN retrying attempt=2
2026/07/16 10:24:32 ERROR request failed err="connection refused"
```

拆一下调用:

```text
slog.Info("server started", "addr", ":8080")
//   ─┬── ───────┬────────  ──┬───  ───┬───
//  ①级别   ②消息(说人话)    ③key    ④value(key/value 必须成对)
```

🕳️ 坑:key/value 落单(奇数个参数)不报错,slog 会输出一个 `!BADKEY` 字段。参数多的时候数一眼对数。

为什么要结构化?`"user 42 login failed"` 这种拼接文本,日志系统只能全文搜索;`"msg=login_failed user_id=42"` 可以按字段过滤、聚合、告警——和 pino 对 `console.log` 的改进是同一个理由。

---

## 5. 两种输出格式:Text 和 JSON

默认输出(上一节)其实是兼容旧 log 的过渡格式。正经用法是显式选 handler:

```go
func main() {
    text := slog.New(slog.NewTextHandler(os.Stdout, nil))
    text.Info("server started", "addr", ":8080")

    json := slog.New(slog.NewJSONHandler(os.Stdout, nil))
    json.Info("server started", "addr", ":8080")
}
```

```text
time=2026-07-16T10:24:32.754-07:00 level=INFO msg="server started" addr=:8080
{"time":"2026-07-16T10:24:32.754795-07:00","level":"INFO","msg":"server started","addr":":8080"}
```

| Handler | 输出 | 用在哪 |
|---------|------|--------|
| `NewTextHandler` | `key=value` 文本 | 本地开发,人眼看 |
| `NewJSONHandler` | 一行一个 JSON | 生产环境,日志系统采集 |

同一行业务代码,换个 handler 就换了格式——**打日志的代码和日志长什么样,解耦了**。

---

## 6. 日志级别:Debug 默认是关的

四个级别按严重程度递增:Debug < Info < Warn < Error。

### 🕳️ 坑:slog.Debug 打了个寂寞

```go
func main() {
    slog.Debug("这条你看不到")
    slog.Info("这条能看到")
}
```

```text
2026/07/16 10:24:33 INFO 这条能看到
```

Debug 那条消失了。为什么:**默认级别是 Info**,低于它的直接丢弃。想看 Debug,要在 handler 上调低门槛:

```go
h := slog.NewTextHandler(os.Stdout, &slog.HandlerOptions{
    Level: slog.LevelDebug,
})
logger := slog.New(h)

logger.Debug("现在能看到 debug 了")
logger.Info("info 照常")
```

```text
time=2026-07-16T10:24:33.845-07:00 level=DEBUG msg="现在能看到 debug 了"
time=2026-07-16T10:24:33.845-07:00 level=INFO msg="info 照常"
```

级别不是装饰,选择标准:Debug=开发排查细节,Info=正常里程碑,Warn=不对劲但还能跑,Error=这次操作失败了。

---

## 7. With:给一批日志挂公共字段

```go
func main() {
    logger := slog.New(slog.NewTextHandler(os.Stdout, nil)).
        With("service", "api", "version", "1.0")

    logger.Info("started")

    reqLogger := logger.With("request_id", "abc123")
    reqLogger.Info("handling request")
}
```

```text
time=2026-07-16T10:24:34.403-07:00 level=INFO msg=started service=api version=1.0
time=2026-07-16T10:24:34.404-07:00 level=INFO msg="handling request" service=api version=1.0 request_id=abc123
```

`With` 返回一个**新** logger(原 logger 不变),之后它打的每条日志都自动带上这些字段。典型用法:服务启动时 `With(service/version)`,每个请求进来再 `With(request_id)` 造一个请求级 logger——同一请求的所有日志靠 request_id 串起来,排查问题时一搜全出。

---

## 8. 不要记录敏感信息

这几样永远别写进日志:

- 密码、token、私钥
- 完整身份证号、银行卡号
- 用户隐私数据

日志会被集中收集、长期保存、很多人可见——写进去等于广播。上一篇 JSON 的 `json:"-"` 挡的是序列化,这里要挡的是你自己的 `logger.Info("login", "password", pwd)`。**能定位问题即可,多一个字段都别给。**

---

## 9. 底层返回 error,边界统一记日志

同一个错误,如果每层都打一遍日志,排查时会看到四五条重复记录。惯例是:

底层只包装、只返回(03 章的 `%w`):

```go
func save(id int) error {
    if err := errDB; err != nil { // 模拟底层存储失败
        return fmt.Errorf("save user %d: %w", id, err)
    }
    return nil
}
```

边界层(HTTP handler、main、任务入口)统一记录:

```go
if err := save(42); err != nil {
    logger.Error("request failed", "err", err)
}
```

```text
time=2026-07-16T10:24:34.930-07:00 level=ERROR msg="request failed" err="save user 42: connection refused"
```

注意输出:`%w` 一路攒下来的上下文(`save user 42: connection refused`)在最终这一条日志里完整呈现——**每层加料、一处出锅**。

---

## 10. 字段名要稳定

```go
logger.Info("request completed",
    "method", "GET",
    "path", "/users",
    "status", 200,
    "duration_ms", 12,
)
```

```text
{"time":"2026-07-16T10:24:35.537626-07:00","level":"INFO","msg":"request completed","method":"GET","path":"/users","status":200,"duration_ms":12}
```

今天叫 `duration_ms`、明天叫 `elapsed`、后天叫 `time_taken`,日志系统里的查询和告警规则就全废了。**把字段名当 API 对待**:定了就不轻易改,单位放进名字里(`duration_ms` 优于 `duration`)。

---

## 本篇重点

- [ ] 结果走 stdout 用 fmt,诊断走 stderr 用 log/slog;`log.Fatal` = 打日志 + `os.Exit(1)`,**defer 不执行**,库代码禁用。
- [ ] slog 的字段是成对的 key/value,落单会出 `!BADKEY`;结构化的意义是让日志可检索、可聚合。
- [ ] 开发用 TextHandler,生产用 JSONHandler,业务代码不用改;Debug 级别默认被丢弃,要用 `HandlerOptions{Level: slog.LevelDebug}` 打开。
- [ ] `With` 挂公共字段(service、request_id),同一请求的日志串成一条线。
- [ ] 底层 `%w` 包装返回、边界统一 `logger.Error`,一个错误只记一次;字段名当 API 对待,敏感信息永不入日志。

---

## 练习

1. 用 `log.New` 创建一个带前缀的 logger,先写到 stdout,再改成写到 `bytes.Buffer` 并打印 buffer 内容(提示:第一个参数是 `io.Writer`)。
2. 用 `slog.NewTextHandler` 输出文本日志,再打开 Debug 级别输出一条 Debug(提示:第 6 节的 HandlerOptions)。
3. 用 `slog.NewJSONHandler` 输出 JSON 日志,观察和文本格式的字段对应关系。
4. 写一个函数返回包装过的 error,在调用边界用 slog 记录,确认日志里能看到完整错误链(提示:第 9 节;`%w` 是 03 章的)。
5. 给日志增加 `request_id` 公共字段,模拟两个不同请求各打两条日志,确认能靠字段区分(提示:`With` 返回新 logger,互不影响)。
