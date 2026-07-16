# 03: 常量与 iota

这篇解决两个问题：Go 的 `const` 和 JS 的 `const` 差在哪；以及 Go 没有 `enum`，那一排状态码、一排权限位该怎么写（答案是 `iota`）。

先对齐概念：JS 的 `const` 是"这个绑定不许再赋值"，值是运行时算出来的都行。Go 的 `const` 更严格——**值必须在编译期就能确定**，运行时算出来的东西（函数返回值、用户输入）都当不了常量。

---

## 1. 基本常量

```go
const pi = 3.14159
const appName = "demo"
const debug = true
```

和 JS 一样，常量不能被重新赋值：

```go
func main() {
    const limit = 10
    limit = 20
}
```

```text
编译错误：cannot assign to limit (neither addressable nor a map index expression)
```

区别在于时机：JS 是运行时抛 `TypeError`，Go 是编译期直接不让过。

---

## 2. 批量声明

一组相关常量用 `const (...)` 块，和 `var (...)` 同款语法：

```go
const (
    statusOK       = 200
    statusNotFound = 404
    statusError    = 500
)
```

命名规则：局部常量可以小写；导出的包级常量首字母大写（Go 用大小写控制可见性，不用 `export` 关键字，这个后面模块篇会展开）。

---

## 3. 无类型常量

Go 的常量有个变量没有的特权：可以是**无类型**的。同一个常量能"变形"塞进不同类型的变量里：

```go
const n = 10

func main() {
    var a int = n
    var b int64 = n
    var c float64 = n

    fmt.Println(a, b, c)
}
```

```text
10 10 10
```

同样的事换成变量就不行——上一篇说过，变量的类型是焊死的：

```go
func main() {
    x := 10           // x 是 int
    var y int64 = x   // int 塞不进 int64
    _ = y
}
```

```text
编译错误：cannot use x (variable of type int) as int64 value in variable declaration
```

需要显式转换（02-06 展开讲）：

```go
var y int64 = int64(x)
```

**一句话总结：常量是"还没定型的数字"，可以适配目标类型；变量是"已定型的值"，跨类型必须显式转换。**

---

## 4. iota

Go 没有 `enum` 关键字。想要 TS 里 `enum Weekday { Monday, Tuesday, ... }` 那种自动编号，用 `iota`——一个只在 `const` 块里生效的计数器，从 `0` 开始，每换一行加 `1`：

```go
const (
    Monday = iota // 0，之后每行自动 +1
    Tuesday
    Wednesday
    Thursday
    Friday
    Saturday
    Sunday
)

func main() {
    fmt.Println(Monday, Tuesday, Wednesday, Thursday, Friday, Saturday, Sunday)
}
```

```text
0 1 2 3 4 5 6
```

拆开看这个块在干什么：

```text
Monday = iota   // 第 0 行：iota = 0
Tuesday         // 第 1 行：省略了 "= iota"，自动沿用上一行的表达式，iota = 1
Wednesday       // 第 2 行：iota = 2
...
```

两条规则：**① 每个 `const` 块里 iota 从 0 重新数；② 后面的行不写 `=`，就自动复用上一行的表达式。**

---

## 5. 跳过 iota

不想要某个值（最常见是不想要 0），用 `_` 把它扔掉：

```go
const (
    _ = iota // 0 被丢弃
    Low
    Medium
    High
)

func main() {
    fmt.Println(Low, Medium, High)
}
```

```text
1 2 3
```

为什么常常跳过 0？还记得上一篇的零值吗——`int` 变量没赋值就是 `0`。如果 `0` 也是一个合法状态，你就分不清"这个变量被赋成了状态 0"还是"这个变量根本没人赋值"。让合法状态从 1 开始，`0` 就天然成了"未设置"的信号。

---

## 6. iota 位标志

`iota` 配合左移 `<<`，一行生成一组互不重叠的权限位（类似 Linux 文件权限的 rwx）：

```go
const (
    Read = 1 << iota // 1 << 0
    Write            // 1 << 1
    Execute          // 1 << 2
)

func main() {
    fmt.Println(Read, Write, Execute)
    fmt.Printf("%03b %03b %03b\n", Read, Write, Execute)
}
```

```text
1 2 4
001 010 100
```

每个常量占一个独立的二进制位，用 `|` 就能组合权限（`Read|Write` = `011` = 3）、用 `&` 判断是否含某权限。位运算细节在 02-04 讲。

---

## 本篇重点

- [ ] Go 的 `const` 值必须编译期可确定，比 JS 的 `const`（只是不许重新赋值）严格得多。
- [ ] 无类型常量可以直接赋给 `int`/`int64`/`float64` 等多种类型；变量不行，必须显式转换。
- [ ] `iota` 是 `const` 块专属计数器：从 0 开始逐行 +1，省略表达式的行自动复用上一行。
- [ ] 状态枚举常从 1 开始（`_ = iota` 跳过 0），把零值 0 留作"未设置"的信号。
- [ ] `1 << iota` 生成位标志：1、2、4、8……每个常量独占一个二进制位。

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

提示：第 2 问有两种写法——用 `_` 跳过，或者改第一行的表达式（想想 `iota + 1`）。第 3 问回看第 5 节和零值。
