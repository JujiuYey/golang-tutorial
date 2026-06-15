# 07: context：取消、超时与请求边界

`context.Context` 用来在 API 边界上传递取消信号、截止时间和请求范围值。它是写可靠并发代码的重要工具。

---

## 1. context 的基本用途

常见用途：

- 调用方取消任务。
- 设置超时或截止时间。
- 在请求链路中传递少量请求范围值。

函数通常把 context 作为第一个参数：

```go
func Fetch(ctx context.Context, url string) ([]byte, error) {
    // ...
}
```

不要把 context 存进 struct 作为长期字段。它应该随请求或操作传递。

---

## 2. Background 和 TODO

```go
ctx := context.Background()
```

`Background` 通常用在 main、测试、顶层初始化。

```go
ctx := context.TODO()
```

`TODO` 用在暂时不知道应该传什么 context 的地方。最终应该替换成更明确的 context。

---

## 3. WithCancel

```go
ctx, cancel := context.WithCancel(context.Background())
defer cancel()
```

`cancel()` 会关闭 `ctx.Done()`。

```go
go func() {
    select {
    case <-ctx.Done():
        fmt.Println("cancelled:", ctx.Err())
    }
}()

cancel()
```

---

## 4. WithTimeout

```go
ctx, cancel := context.WithTimeout(context.Background(), time.Second)
defer cancel()
```

超过时间后，`ctx.Done()` 会关闭，`ctx.Err()` 返回 `context.DeadlineExceeded`。

即使超时会自动发生，也应该调用 `cancel()`，让相关资源及时释放。

---

## 5. 在 goroutine 中监听 ctx.Done

```go
func worker(ctx context.Context, jobs <-chan Job) {
    for {
        select {
        case job, ok := <-jobs:
            if !ok {
                return
            }
            handle(job)
        case <-ctx.Done():
            return
        }
    }
}
```

长期运行的 goroutine 应该能响应取消。

---

## 6. 发送时也要监听取消

```go
func produce(ctx context.Context, out chan<- int) error {
    for i := 0; i < 10; i++ {
        select {
        case out <- i:
        case <-ctx.Done():
            return ctx.Err()
        }
    }
    return nil
}
```

如果下游停止接收，单纯 `out <- i` 可能永久阻塞。发送时监听 `ctx.Done()` 可以让上游退出。

---

## 7. context 值

```go
type traceIDKey struct{}

ctx := context.WithValue(context.Background(), traceIDKey{}, "abc-123")

traceID, _ := ctx.Value(traceIDKey{}).(string)
```

context value 适合传递请求范围的小数据，比如 trace id、认证信息。不要用它传业务参数、配置对象或可选参数。

---

## 8. context 传播

```go
func Handler(ctx context.Context, id int64) error {
    user, err := LoadUser(ctx, id)
    if err != nil {
        return err
    }
    return SendEmail(ctx, user.Email)
}
```

同一个请求链路上的下游调用应该继续传递 context。这样上游取消时，下游也能停止。

---

## 9. context 和 error

```go
if err := ctx.Err(); err != nil {
    return err
}
```

常见错误：

- `context.Canceled`
- `context.DeadlineExceeded`

调用方可以根据错误判断是主动取消还是超时。

---

## 10. context 使用检查

写带 context 的函数时问：

1. context 是否作为第一个参数。
2. 是否在阻塞操作中监听 `ctx.Done()`。
3. 是否调用了 `cancel()`。
4. 是否把 context 继续传给下游。
5. 是否滥用 context value 传业务参数。

---

## 练习

1. 用 `context.WithCancel` 停止一个 worker。
2. 用 `context.WithTimeout` 给任务加 200ms 超时。
3. 写一个 producer，发送时监听 `ctx.Done()`。
4. 写函数链 `A(ctx) -> B(ctx) -> C(ctx)`，让取消信号传到最底层。
5. 用 context value 存取 trace id，并说明为什么不应该用它传普通参数。
