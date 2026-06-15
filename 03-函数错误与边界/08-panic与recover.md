# 08: panic、recover 与边界

`panic` 表示程序进入了无法正常继续的状态。普通业务错误应该返回 `error`，不要用 `panic`。

---

## 1. panic

```go
func panicExample() {
    fmt.Println("开始")
    panic("发生严重错误")
    fmt.Println("不会执行")
}
```

`panic` 会中断当前函数的正常执行，并开始沿调用栈向上传播。

---

## 2. panic 会执行 defer

```go
func main() {
    defer fmt.Println("defer")
    panic("boom")
}
```

发生 panic 时，已经注册的 `defer` 仍会执行。

---

## 3. recover

`recover` 可以捕获 panic，防止程序崩溃。它必须在 defer 函数中调用才有效。

```go
func safeCall() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("捕获到 panic:", r)
        }
    }()

    panic("故意的 panic")
}
```

---

## 4. recover 的边界

`recover` 只能恢复当前 goroutine 中的 panic。

```go
func main() {
    go func() {
        panic("goroutine panic")
    }()
}
```

外层 goroutine 的 `recover` 捕获不到另一个 goroutine 的 panic。

---

## 5. 什么时候使用 panic

适合使用 panic：

- 程序启动时发现不可恢复配置错误。
- 违反程序内部不变量。
- 库内部用 panic 简化控制流，但在包边界 recover 后转成 error。

不适合使用 panic：

- 用户输入错误。
- 文件不存在。
- 网络请求失败。
- 数据库查询没有结果。

这些应该返回 `error`。

---

## 6. 包边界恢复

如果内部可能 panic，可以在边界处恢复并转成 error。

```go
func runSafely(fn func()) (err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("panic: %v", r)
        }
    }()

    fn()
    return nil
}
```

---

## 练习

实现函数：

```go
func safeRun(fn func()) (err error)
```

要求：

1. 正常执行时返回 `nil`。
2. 如果 `fn` panic，捕获 panic 并返回 error。
3. 不要在普通业务逻辑里主动 panic。
