# 04: 缓冲 channel 与单向 channel

缓冲 channel 在发送方和接收方之间提供一段容量。单向 channel 用来限制函数只能发送或只能接收。

---

## 1. 创建缓冲 channel

```go
ch := make(chan int, 3)
```

容量是 3。可以查看长度和容量：

```go
fmt.Println(len(ch))
fmt.Println(cap(ch))
```

`len(ch)` 表示当前缓冲区里有多少个值，`cap(ch)` 表示容量。

---

## 2. 发送到缓冲 channel

```go
ch := make(chan int, 2)

ch <- 1
ch <- 2
```

前两次发送不会阻塞，因为缓冲区还有空间。

第三次发送会阻塞，直到有接收方取走一个值：

```go
// ch <- 3 // 阻塞
```

---

## 3. 从缓冲 channel 接收

```go
fmt.Println(<-ch) // 1
fmt.Println(<-ch) // 2
```

channel 保持先进先出的接收顺序。

如果缓冲区为空，并且没有发送方，接收会阻塞。

---

## 4. 缓冲不是无限队列

缓冲 channel 可以吸收短时间的生产消费速度差异，但不能当成无限队列。

如果生产速度长期大于消费速度：

- 缓冲区最终会满。
- 发送方会阻塞。
- goroutine 可能堆积。

容量应该来自明确的设计，比如 worker 数量、批次大小、外部系统并发限制。

---

## 5. 容量影响同步语义

无缓冲 channel：

```go
ch := make(chan int)
```

发送和接收必须配对，强调同步。

缓冲 channel：

```go
ch := make(chan int, 10)
```

发送方可以先放入一部分数据，强调解耦。

容量大小会影响程序行为，不能随意改。

---

## 6. 单向发送 channel

```go
func producer(out chan<- int) {
    for i := 0; i < 3; i++ {
        out <- i
    }
    close(out)
}
```

`chan<- int` 表示只发送。函数内部不能从 `out` 接收。

---

## 7. 单向接收 channel

```go
func consumer(in <-chan int) {
    for v := range in {
        fmt.Println(v)
    }
}
```

`<-chan int` 表示只接收。函数内部不能向 `in` 发送，也不能关闭它。

---

## 8. 双向 channel 可以传给单向参数

```go
ch := make(chan int)

go producer(ch)
consumer(ch)
```

`ch` 是双向 channel。传入 `producer` 后，在函数内部只允许发送。传入 `consumer` 后，在函数内部只允许接收。

单向 channel 是 API 约束，不是运行时新建了一个 channel。

---

## 9. 用缓冲 channel 限制并发

```go
sem := make(chan struct{}, 3)

for _, task := range tasks {
    sem <- struct{}{}
    go func(task Task) {
        defer func() { <-sem }()
        handle(task)
    }(task)
}
```

`sem` 的容量是 3，表示最多同时运行 3 个任务。

实际项目中还要配合 `WaitGroup` 等待所有 goroutine 完成。

---

## 10. 空 struct 信号

```go
done := make(chan struct{})
```

如果 channel 只用于发送信号，不关心值本身，常用 `struct{}`。

```go
done <- struct{}{}
```

关闭 channel 也可以广播信号，后面会专门讲。

---

## 练习

1. 创建容量为 2 的 channel，发送 2 个值，再接收它们。
2. 写代码验证第三次发送会阻塞。
3. 写 `producer(out chan<- int)` 和 `consumer(in <-chan int)`。
4. 用容量为 3 的 channel 限制同时运行的任务数量。
5. 用 `chan struct{}` 表示完成信号。
