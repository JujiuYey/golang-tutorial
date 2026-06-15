# 03: 常量与 iota

常量用 `const` 声明。常量的值在编译期确定，运行时不能修改。

---

## 1. 基本常量

```go
const pi = 3.14159
const appName = "demo"
const debug = true
```

常量不能被重新赋值：

```go
func main() {
    const limit = 10
    // limit = 20 // 编译错误
}
```

---

## 2. 批量声明

```go
const (
    statusOK       = 200
    statusNotFound = 404
    statusError    = 500
)
```

常量命名通常按用途决定。局部常量可以小写，导出的包级常量首字母大写。

---

## 3. 无类型常量

Go 的常量可以是无类型的。

```go
const n = 10

func main() {
    var a int = n
    var b int64 = n
    var c float64 = n

    fmt.Println(a, b, c)
}
```

这里 `n` 可以赋给 `int`、`int64`、`float64`，因为它是无类型常量。变量不行：

```go
func main() {
    x := 10
    var y int64 = x // 编译错误
    _ = y
}
```

需要显式转换：

```go
var y int64 = int64(x)
```

---

## 4. iota

`iota` 是常量生成器，在每个 `const` 块中从 `0` 开始递增。

```go
const (
    Monday = iota
    Tuesday
    Wednesday
    Thursday
    Friday
    Saturday
    Sunday
)
```

等价于：

```go
const (
    Monday    = 0
    Tuesday   = 1
    Wednesday = 2
    Thursday  = 3
    Friday    = 4
    Saturday  = 5
    Sunday    = 6
)
```

---

## 5. 跳过 iota

可以用 `_` 跳过某个值：

```go
const (
    _ = iota
    Low
    Medium
    High
)
```

这里 `Low` 是 `1`，`Medium` 是 `2`，`High` 是 `3`。

---

## 6. iota 位标志

`iota` 常用于位标志：

```go
const (
    Read = 1 << iota
    Write
    Execute
)
```

等价于：

```go
const (
    Read    = 1 // 001
    Write   = 2 // 010
    Execute = 4 // 100
)
```

---

## 练习

定义一个订单状态常量：

```go
const (
    StatusPending = iota
    StatusPaid
    StatusShipped
    StatusCanceled
)
```

要求：

1. 打印每个状态对应的数字。
2. 再改成从 `1` 开始。
3. 解释为什么有时不希望状态从 `0` 开始。
