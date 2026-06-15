# 10: 数据竞争、死锁与 goroutine 泄漏

并发代码的高风险问题主要是竞争、死锁和 goroutine 泄漏。它们都可能在小示例里看不出来，在真实负载下暴露。

---

## 1. 数据竞争

当两个 goroutine 同时访问同一变量，并且至少一个是写操作，且没有同步保护时，就可能发生数据竞争。

```go
var n int

go func() {
    n++
}()

go func() {
    n++
}()
```

`n++` 会读旧值、加一、写回。两个 goroutine 交错执行时，结果不可靠。

---

## 2. race detector

运行测试时加 `-race`：

```bash
go test -race ./...
```

运行程序时也可以：

```bash
go run -race .
```

race detector 会在运行到存在竞争的路径时报告问题。它只能发现实际执行到的竞争，所以测试覆盖仍然重要。

---

## 3. 普通 map 并发读写

普通 map 不能并发读写。

```go
m := map[string]int{}

go func() {
    m["x"] = 1
}()

go func() {
    fmt.Println(m["x"])
}()
```

这可能触发运行时错误，也可能被 race detector 报告。使用 `Mutex`、`sync.Map` 或单 goroutine 所有权模型。

---

## 4. 死锁

死锁表示 goroutine 互相等待，程序无法继续推进。

```go
ch := make(chan int)
ch <- 1
```

没有接收方时，发送会阻塞。main goroutine 卡住后，运行时会报告死锁。

---

## 5. 锁顺序死锁

```go
var a, b sync.Mutex

go func() {
    a.Lock()
    defer a.Unlock()
    b.Lock()
    defer b.Unlock()
}()

go func() {
    b.Lock()
    defer b.Unlock()
    a.Lock()
    defer a.Unlock()
}()
```

两个 goroutine 以相反顺序拿锁，可能互相等待。多把锁时要固定加锁顺序。

---

## 6. goroutine 泄漏

goroutine 泄漏是指 goroutine 永远阻塞或无法退出。

```go
func gen() <-chan int {
    out := make(chan int)
    go func() {
        for i := 0; ; i++ {
            out <- i
        }
    }()
    return out
}

func main() {
    out := gen()
    fmt.Println(<-out)
}
```

main 只接收一个值，`gen` 里的 goroutine 还会继续尝试发送，最终卡住。

---

## 7. 用 context 避免泄漏

```go
func gen(ctx context.Context) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for i := 0; ; i++ {
            select {
            case out <- i:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}
```

调用方取消 context 后，生成器 goroutine 可以退出。

---

## 8. channel 泄漏

如果下游不再接收，上游发送可能永久阻塞。

```go
func send(out chan<- int) {
    out <- 1
}
```

在可能取消的代码里，发送也要监听取消：

```go
select {
case out <- 1:
case <-ctx.Done():
    return
}
```

---

## 9. 关闭 channel 的 panic

常见 panic：

- 向已关闭 channel 发送。
- 关闭已经关闭的 channel。
- 关闭 nil channel。

关闭责任要明确，最好由唯一发送方或协调者关闭。

---

## 10. 排查并发问题

常用手段：

- `go test -race ./...`
- 给 goroutine 增加明确退出日志。
- 用 context 控制生命周期。
- 缩小复现代码。
- 检查 channel 关闭责任。
- 检查锁顺序。
- 检查是否持锁做慢操作。

并发问题不要靠猜。先复现，再加同步，再验证。

---

## 练习

1. 写一个数据竞争测试，用 `go test -race` 观察报告。
2. 用 Mutex 修复这个数据竞争。
3. 写一个无接收方发送导致死锁的例子。
4. 写一个 goroutine 泄漏的生成器，再用 context 修复。
5. 写一个向已关闭 channel 发送的例子，观察 panic。
