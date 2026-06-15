# 06: select、多路复用与超时

`select` 用来同时等待多个 channel 操作。它是 Go 并发中处理超时、取消、多输入源的核心语法。

---

## 1. 基本 select

```go
select {
case v := <-ch1:
    fmt.Println("ch1:", v)
case v := <-ch2:
    fmt.Println("ch2:", v)
}
```

哪个 case 的 channel 先准备好，就执行哪个 case。

如果多个 case 同时准备好，Go 会伪随机选择一个。

---

## 2. select 会阻塞

如果没有 case 准备好，且没有 `default`，`select` 会阻塞。

```go
select {
case v := <-ch:
    fmt.Println(v)
}
```

这和直接 `<-ch` 类似，只是 `select` 可以等待多个操作。

---

## 3. default

```go
select {
case v := <-ch:
    fmt.Println(v)
default:
    fmt.Println("not ready")
}
```

如果没有 channel 操作准备好，会立即执行 `default`。

`default` 会让 select 变成非阻塞检查。不要在循环里无节制地使用空 `default`，否则可能造成 CPU 空转。

---

## 4. 超时

```go
select {
case v := <-ch:
    fmt.Println("value:", v)
case <-time.After(time.Second):
    fmt.Println("timeout")
}
```

`time.After` 返回一个 channel，到时间后会收到当前时间。

如果在高频循环中使用超时，更适合使用 `time.NewTimer` 并适当复用或停止，避免创建太多 timer。

---

## 5. 等待取消

```go
select {
case v := <-work:
    fmt.Println(v)
case <-done:
    return
}
```

`done` 可以是普通 channel，也可以是 `ctx.Done()`。

---

## 6. select 中发送

```go
select {
case out <- value:
    fmt.Println("sent")
case <-ctx.Done():
    return ctx.Err()
}
```

发送也可以作为 case。这样可以避免下游不接收时 goroutine 永久卡住。

---

## 7. 处理关闭的 channel

```go
select {
case v, ok := <-ch:
    if !ok {
        return
    }
    fmt.Println(v)
}
```

从已关闭 channel 接收会立即成功。如果在 select 循环中不处理关闭状态，可能不断收到零值。

---

## 8. nil channel 可以动态禁用 case

nil channel 上的操作永远不准备好。可以把 channel 设为 nil 来禁用某个 case。

```go
for ch1 != nil || ch2 != nil {
    select {
    case v, ok := <-ch1:
        if !ok {
            ch1 = nil
            continue
        }
        fmt.Println("ch1", v)
    case v, ok := <-ch2:
        if !ok {
            ch2 = nil
            continue
        }
        fmt.Println("ch2", v)
    }
}
```

这在合并多个输入 channel 时很有用。

---

## 9. for select

```go
for {
    select {
    case job := <-jobs:
        handle(job)
    case <-ctx.Done():
        return
    }
}
```

`for select` 是长期运行 goroutine 的常见结构。每个循环都应该有明确退出条件。

---

## 10. select 设计检查

写 select 时问：

1. 没有 case 准备好时，应该等待还是立即返回。
2. channel 关闭后是否正确退出。
3. 是否需要处理取消信号。
4. 发送 case 是否可能因为无人接收而卡住。
5. `default` 是否会导致忙等。

---

## 练习

1. 同时等待两个 channel，打印先到的值。
2. 用 `time.After` 给接收操作加 500ms 超时。
3. 写一个 `for select`，同时处理 jobs 和 done。
4. 合并两个输入 channel，关闭后把对应 channel 设为 nil。
5. 写一个发送 case，同时监听 `ctx.Done()`。
