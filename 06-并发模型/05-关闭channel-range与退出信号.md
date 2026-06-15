# 05: 关闭 channel、range 与退出信号

关闭 channel 表示不会再有新值发送。关闭不是清空 channel，也不是让 goroutine 停止的魔法。

---

## 1. close 的含义

```go
close(ch)
```

关闭 channel 后：

- 不能再发送值。
- 已经缓冲的值仍然可以被接收。
- 缓冲区取空后，继续接收会得到元素类型零值和 `ok=false`。

---

## 2. 接收关闭状态

```go
v, ok := <-ch
if !ok {
    fmt.Println("channel closed")
    return
}
fmt.Println(v)
```

`ok=false` 表示 channel 已关闭且没有剩余值。

---

## 3. range channel

```go
for v := range ch {
    fmt.Println(v)
}
```

`range ch` 会一直接收，直到 channel 被关闭并且值被取完。

如果 channel 永远不关闭，`range` 会一直等待。

---

## 4. 发送方负责关闭

通常由发送方关闭 channel，因为发送方知道什么时候不会再发送。

```go
func producer(out chan<- int) {
    defer close(out)
    for i := 0; i < 3; i++ {
        out <- i
    }
}
```

接收方不要随便关闭 channel。接收方不知道是否还有其他发送方正在发送。

---

## 5. 关闭后发送会 panic

```go
ch := make(chan int)
close(ch)

// ch <- 1 // panic: send on closed channel
```

因此多发送方场景要特别小心关闭时机。

---

## 6. 关闭 nil channel 会 panic

```go
var ch chan int
// close(ch) // panic
```

关闭前要确保 channel 已经初始化，并且当前 goroutine 拥有关闭责任。

---

## 7. 多发送方关闭模式

多个 goroutine 都向同一个 channel 发送时，不应该让任意发送方直接关闭 channel。

常见做法是用 `WaitGroup` 等所有发送方结束，再由一个独立 goroutine 关闭。

```go
out := make(chan int)
var wg sync.WaitGroup

for i := 0; i < 3; i++ {
    wg.Add(1)
    go func(i int) {
        defer wg.Done()
        out <- i
    }(i)
}

go func() {
    wg.Wait()
    close(out)
}()

for v := range out {
    fmt.Println(v)
}
```

---

## 8. done channel

可以用关闭 channel 广播退出信号。

```go
done := make(chan struct{})

go func() {
    <-done
    fmt.Println("stop")
}()

close(done)
```

关闭后的 channel 可以被所有接收方立即接收到零值，所以适合广播。

更现代的取消传播通常使用 `context.Context`，后面会讲。

---

## 9. 不要用 close 传递错误

关闭只能表达“没有更多值”。如果要传递错误，应该使用单独的 error channel、结果结构体或 context 取消原因。

```go
type Result struct {
    Value int
    Err   error
}
```

把值和错误放在同一个结果类型里，语义更清楚。

---

## 10. channel 关闭检查

写 channel 代码时问：

1. 谁是唯一关闭者。
2. 关闭发生在所有发送之后吗。
3. 接收方是否能在关闭后退出。
4. 是否有发送方可能向已关闭 channel 发送。
5. 是否需要传递错误或取消原因。

---

## 练习

1. 写一个 producer，发送 1 到 5 后关闭 channel。
2. 用 `for range` 接收 producer 输出。
3. 写代码验证关闭后继续接收会得到零值和 `ok=false`。
4. 写多发送方示例，用 `WaitGroup` 统一关闭结果 channel。
5. 用关闭 `done` channel 的方式通知多个 goroutine 退出。
