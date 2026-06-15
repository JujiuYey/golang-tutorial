# 05: error 基础与错误返回

Go 使用 `error` 表示错误。错误是普通返回值，不是异常。

---

## 1. error 接口

`error` 是内置接口：

```go
type error interface {
    Error() string
}
```

任何实现了 `Error() string` 方法的类型，都可以作为错误。

---

## 2. 创建错误

使用 `errors.New` 创建简单错误：

```go
err := errors.New("文件不存在")
```

使用 `fmt.Errorf` 创建带格式的错误：

```go
err := fmt.Errorf("无效的参数: %s", arg)
```

---

## 3. 返回错误

```go
func safeDivide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("除数不能为 0")
    }
    return a / b, nil
}
```

成功时错误返回 `nil`。

---

## 4. 处理错误

```go
result, err := safeDivide(10, 0)
if err != nil {
    fmt.Println("error:", err)
    return
}
fmt.Println(result)
```

Go 的错误处理通常紧跟在函数调用之后。

---

## 5. 不要随便忽略错误

```go
result, _ := safeDivide(10, 0)
fmt.Println(result)
```

这段代码会把失败当成正常结果继续执行。忽略错误前必须明确知道为什么可以忽略。

---

## 6. 错误信息要能定位问题

错误信息应该说明发生了什么。

```go
return errors.New("invalid input")
```

如果有上下文，带上上下文：

```go
return fmt.Errorf("read config %s failed", path)
```

---

## 练习

实现函数：

```go
func factorial(n int) (int, error)
```

要求：

1. `n < 0` 时返回错误。
2. `0` 和 `1` 的阶乘返回 `1`。
3. 正常返回结果和 `nil`。
