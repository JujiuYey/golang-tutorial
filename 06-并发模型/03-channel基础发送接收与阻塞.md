# 03: channel 基础：发送、接收与阻塞

channel 是 goroutine 之间通信和同步的工具。它可以传递值，也可以让两个 goroutine 在某个点汇合。

---

## 1. 创建 channel

```go
ch := make(chan int)
```

这是一个无缓冲 channel。它传递 `int` 类型的值。

channel 的零值是 nil：

```go
var ch chan int
fmt.Println(ch == nil) // true
```

nil channel 上的发送和接收会永久阻塞。

---

## 2. 发送和接收

发送：

```go
ch <- 100
```

接收：

```go
v := <-ch
```

无缓冲 channel 要求发送方和接收方同时准备好。

```go
ch := make(chan int)

go func() {
    ch <- 100
}()

v := <-ch
fmt.Println(v)
```

---

## 3. 无缓冲 channel 是同步点

```go
func main() {
    ch := make(chan string)

    go func() {
        ch <- "done"
    }()

    msg := <-ch
    fmt.Println(msg)
}
```

发送方执行到 `ch <- "done"` 会等待接收方。接收方执行到 `<-ch` 会等待发送方。双方配对后，值完成传递。

---

## 4. 发送会阻塞

```go
ch := make(chan int)
ch <- 1 // 没有接收方，阻塞
```

这段代码在 main goroutine 中会导致死锁：

```text
fatal error: all goroutines are asleep - deadlock!
```

---

## 5. 接收会阻塞

```go
ch := make(chan int)
v := <-ch // 没有发送方，阻塞
fmt.Println(v)
```

channel 操作阻塞不是错误，它是同步机制。问题在于有没有其他 goroutine 能继续推进程序。

---

## 6. channel 传递的是值

```go
ch := make(chan int)

go func() {
    x := 10
    ch <- x
    x = 20
}()

fmt.Println(<-ch) // 10
```

发送时会把值复制进 channel。对于 slice、map、指针这类值，复制的是它们的描述符或地址，底层数据仍可能共享。

---

## 7. 用 channel 返回结果

```go
func square(n int, out chan<- int) {
    out <- n * n
}

func main() {
    out := make(chan int)
    go square(4, out)

    result := <-out
    fmt.Println(result)
}
```

这里 channel 既传递了结果，也让 main goroutine 等待计算完成。

---

## 8. 多个发送方

```go
ch := make(chan int)

for i := 0; i < 3; i++ {
    go func(i int) {
        ch <- i
    }(i)
}

for i := 0; i < 3; i++ {
    fmt.Println(<-ch)
}
```

接收顺序取决于 goroutine 调度，不保证等于启动顺序。

---

## 9. 多个接收方

```go
jobs := make(chan int)

for worker := 0; worker < 3; worker++ {
    go func(id int) {
        for job := range jobs {
            fmt.Println("worker", id, "job", job)
        }
    }(worker)
}
```

多个接收方从同一个 channel 接收时，每个值只会被其中一个接收方拿到。

---

## 10. channel 不是队列万能药

channel 很适合表达 goroutine 之间的通信，但不要为了“并发感”把所有数据流都改成 channel。

如果只是普通函数调用、普通 slice 遍历、普通 map 查询，直接写同步代码通常更清晰。

---

## 练习

1. 用无缓冲 channel 从 goroutine 返回一个字符串。
2. 写代码验证无接收方发送会阻塞。
3. 启动 3 个 goroutine 向同一个 channel 发送结果，主 goroutine 接收 3 次。
4. 启动 2 个 worker 从同一个 jobs channel 读取任务。
5. 解释 channel 传递 slice 时，哪些部分被复制，哪些数据可能共享。
