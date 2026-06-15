# 02: WaitGroup：等待一组任务完成

`sync.WaitGroup` 用来等待一组 goroutine 完成。它解决的是“主 goroutine 如何知道其他 goroutine 已经结束”的问题。

---

## 1. 基本用法

```go
func worker(id int, wg *sync.WaitGroup) {
    defer wg.Done()
    fmt.Println("worker", id)
}

func main() {
    var wg sync.WaitGroup

    for i := 0; i < 3; i++ {
        wg.Add(1)
        go worker(i, &wg)
    }

    wg.Wait()
    fmt.Println("all done")
}
```

三个核心方法：

- `Add(n)`: 增加等待计数。
- `Done()`: 完成一个任务，等价于 `Add(-1)`。
- `Wait()`: 阻塞直到计数归零。

---

## 2. Add 要在启动 goroutine 前调用

推荐写法：

```go
wg.Add(1)
go func() {
    defer wg.Done()
    doWork()
}()
```

不要把 `Add(1)` 放进 goroutine 里：

```go
go func() {
    wg.Add(1) // 错误习惯
    defer wg.Done()
    doWork()
}()
```

主 goroutine 可能先执行到 `Wait()`，这时计数还没加上，等待关系就错了。

---

## 3. 用 defer Done

```go
go func() {
    defer wg.Done()

    if err := doWork(); err != nil {
        return
    }
}()
```

`defer wg.Done()` 可以保证函数无论从哪个分支返回，都会减少计数。

---

## 4. WaitGroup 必须传指针

```go
func worker(wg *sync.WaitGroup) {
    defer wg.Done()
}
```

不要复制 `WaitGroup`。复制后不同 goroutine 操作的可能不是同一个计数器。

```go
func worker(wg sync.WaitGroup) { // 错误
    defer wg.Done()
}
```

`sync` 包里的很多类型都不应该复制，包括 `Mutex`、`RWMutex`、`Once`。

---

## 5. WaitGroup 不收集错误

`WaitGroup` 只负责等待，不负责返回错误。

错误可以通过 channel 收集：

```go
errCh := make(chan error, 3)

for i := 0; i < 3; i++ {
    wg.Add(1)
    go func(i int) {
        defer wg.Done()
        if err := doWork(i); err != nil {
            errCh <- err
        }
    }(i)
}

wg.Wait()
close(errCh)

for err := range errCh {
    fmt.Println(err)
}
```

如果需要“有一个任务失败就取消其他任务”，后面会结合 context 讲。

---

## 6. 等待并关闭结果 channel

常见模式：

```go
results := make(chan int)
var wg sync.WaitGroup

for i := 0; i < 3; i++ {
    wg.Add(1)
    go func(i int) {
        defer wg.Done()
        results <- i * i
    }(i)
}

go func() {
    wg.Wait()
    close(results)
}()

for v := range results {
    fmt.Println(v)
}
```

关闭 `results` 的 goroutine 等所有生产者退出后再关闭 channel。这样接收方可以用 `range` 安全退出。

---

## 7. 计数不能为负

如果 `Done` 调用次数多于 `Add`，会 panic。

```go
var wg sync.WaitGroup
wg.Done() // panic
```

每个 `Add(1)` 应该对应一个任务完成时的 `Done()`。

---

## 8. WaitGroup 可以复用，但要小心

上一轮 `Wait()` 返回后，可以再次使用同一个 `WaitGroup`。

```go
wg.Add(1)
go func() {
    defer wg.Done()
}()
wg.Wait()

wg.Add(1)
go func() {
    defer wg.Done()
}()
wg.Wait()
```

不要在一轮 `Wait` 尚未结束时混乱地开始下一轮任务。清晰的代码通常每一组任务使用一个局部 `WaitGroup`。

---

## 9. WaitGroup 和共享变量

```go
var total int

for i := 0; i < 100; i++ {
    wg.Add(1)
    go func() {
        defer wg.Done()
        total++
    }()
}

wg.Wait()
```

`WaitGroup` 只保证等待，不保证 `total++` 安全。这里有数据竞争。保护共享变量需要 `Mutex`、atomic，或改用 channel 汇总。

---

## 练习

1. 用 `WaitGroup` 等待 5 个 worker 完成。
2. 故意漏掉一个 `Done`，观察程序是否卡住。
3. 故意多调用一次 `Done`，观察 panic。
4. 用 `WaitGroup` 等待多个生产者结束后关闭结果 channel。
5. 写一段代码说明 `WaitGroup` 不能解决 `total++` 的数据竞争。
