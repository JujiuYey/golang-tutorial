# 07: defer 规则与资源清理

`defer` 会把函数调用延迟到当前函数返回前执行。它最常见的用途是资源清理。

---

## 1. 基本用法

```go
func read() {
    file, err := os.Open("test.txt")
    if err != nil {
        return
    }
    defer file.Close()

    // 使用 file
}
```

注意：只有资源成功创建后，才应该 `defer` 清理。

---

## 2. defer 执行顺序

多个 `defer` 按后进先出顺序执行。

```go
func example() {
    fmt.Println("开始")

    defer fmt.Println("defer 1")
    defer fmt.Println("defer 2")
    defer fmt.Println("defer 3")

    fmt.Println("结束")
}
```

输出：

```text
开始
结束
defer 3
defer 2
defer 1
```

---

## 3. defer 参数立即求值

```go
func main() {
    i := 1
    defer fmt.Println(i)
    i = 2
}
```

输出：

```text
1
```

`defer fmt.Println(i)` 注册时，参数 `i` 已经求值。

---

## 4. defer 与闭包

闭包会在执行时读取外部变量。

```go
func main() {
    i := 1
    defer func() {
        fmt.Println(i)
    }()
    i = 2
}
```

输出：

```text
2
```

---

## 5. defer 修改命名返回值

命名返回值可以在 `defer` 中被修改。

```go
func example() (err error) {
    defer func() {
        if err != nil {
            err = fmt.Errorf("example failed: %w", err)
        }
    }()

    return errors.New("original")
}
```

这种写法要谨慎使用，避免让返回路径变得难读。

---

## 6. Close 错误

很多资源的 `Close` 也会返回错误。

```go
defer file.Close()
```

这会忽略 `Close` 的错误。简单读文件通常可以接受；写文件、网络连接、压缩流等场景可能需要处理。

---

## 练习

实现函数：

```go
func readFile(path string) ([]byte, error)
```

要求：

1. 使用 `os.Open` 打开文件。
2. 打开失败时返回带路径的错误。
3. 打开成功后使用 `defer` 关闭文件。
4. 使用 `io.ReadAll` 读取内容。
