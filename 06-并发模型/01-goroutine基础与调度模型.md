# 01: goroutine 基础与调度模型

goroutine 是 Go 运行时管理的并发执行单元。使用 `go` 关键字可以启动一个新的 goroutine。

---

## 1. 启动 goroutine

```go
func printMessage(msg string) {
    fmt.Println(msg)
}

func main() {
    go printMessage("hello")
}
```

`go printMessage("hello")` 会让 `printMessage` 在新的 goroutine 中执行。

注意：`main` 函数返回时，整个程序会退出。其他 goroutine 不会自动阻止程序结束。

---

## 2. 不要用 Sleep 等待 goroutine

很多入门示例会这样写：

```go
go doWork()
time.Sleep(time.Second)
```

这种写法不可靠。任务可能提前完成，也可能超过 1 秒还没完成。更好的方式是使用 `sync.WaitGroup`、channel 或 context 表达明确的同步关系。

---

## 3. 匿名函数 goroutine

```go
go func() {
    fmt.Println("run in goroutine")
}()
```

匿名函数后面的 `()` 表示立即调用，只是调用发生在新的 goroutine 中。

可以传参数：

```go
name := "worker-1"

go func(n string) {
    fmt.Println(n)
}(name)
```

把需要的值作为参数传进去，通常比直接捕获外部变量更清晰。

---

## 4. goroutine 和函数调用的区别

普通函数调用会等待函数返回：

```go
doWork()
fmt.Println("done")
```

goroutine 调用会立即继续执行后面的代码：

```go
go doWork()
fmt.Println("started")
```

这意味着启动 goroutine 后，你必须明确设计结果如何返回、错误如何处理、什么时候退出。

---

## 5. Go 调度模型的直觉

Go 运行时会把大量 goroutine 调度到较少的操作系统线程上运行。

你不需要直接管理线程。你需要管理的是：

- goroutine 数量是否失控。
- goroutine 是否能退出。
- goroutine 之间如何通信。
- 共享状态是否安全。

---

## 6. GOMAXPROCS

`GOMAXPROCS` 控制同时执行 Go 代码的操作系统线程数量上限。

```go
runtime.GOMAXPROCS(4)
```

现代 Go 默认会根据 CPU 核数设置。大多数业务代码不需要手动调整。

并发不等于并行：

- 并发：多个任务在时间上交错推进。
- 并行：多个任务在同一时刻真正同时执行。

goroutine 支持并发，是否并行还取决于 CPU、调度和 `GOMAXPROCS`。

---

## 7. goroutine 的参数求值

```go
for i := 0; i < 3; i++ {
    go fmt.Println(i)
}
```

调用 `go fmt.Println(i)` 时，参数会先求值，再启动 goroutine。

如果使用闭包，要小心捕获变量：

```go
for i := 0; i < 3; i++ {
    i := i
    go func() {
        fmt.Println(i)
    }()
}
```

把循环变量重新绑定或作为参数传入，可以让每个 goroutine 拿到自己的值：

```go
for i := 0; i < 3; i++ {
    go func(v int) {
        fmt.Println(v)
    }(i)
}
```

---

## 8. goroutine 没有返回值接收方

不能直接从 goroutine 拿返回值：

```go
// result := go compute() // 语法错误
```

常见做法：

- 用 channel 返回结果。
- 写入受保护的共享变量。
- 使用回调或任务队列。
- 在更高层使用 errgroup 这类工具。

本章先使用标准库讲基础模式。

---

## 9. panic 的影响

某个 goroutine 发生 panic 且没有 recover，会导致整个程序崩溃。

```go
go func() {
    panic("boom")
}()
```

如果 goroutine 边界需要恢复 panic，应在 goroutine 内部 defer recover。

```go
go func() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("panic:", r)
        }
    }()
    doWork()
}()
```

不要用 recover 掩盖普通错误。普通失败应该返回 error 或通过 channel 传递。

---

## 10. goroutine 的设计检查

启动 goroutine 前先问：

1. 它什么时候退出。
2. 如果发生错误，错误往哪里传。
3. 如果调用方取消，goroutine 怎么知道。
4. 它是否访问共享数据。
5. 同时启动很多个时，数量是否受控。

这些问题比 `go` 关键字本身更重要。

---

## 练习

1. 启动 3 个 goroutine 打印不同名称，用 `WaitGroup` 等待它们完成。
2. 把循环变量作为参数传给 goroutine，观察输出。
3. 故意在 goroutine 中 panic，观察程序行为。
4. 给 goroutine 加上 defer recover，观察程序是否继续运行。
5. 写一个函数，说明它启动的 goroutine 如何退出。
