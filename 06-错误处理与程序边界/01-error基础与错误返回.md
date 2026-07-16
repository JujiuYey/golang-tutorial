# 01: error 基础与错误返回

这篇解决 JS 转 Go 最大的思维切换：**Go 没有 try/catch**。错误不往外抛，而是作为普通返回值递到你手上。学完这篇，你能写出地道的"返回错误 + 检查错误"代码，不再下意识找 catch 在哪。

---

## 1. error 是什么

先扔掉"异常"的概念。Go 的错误就是一个**普通的值**——能赋给变量、能打印、能作为参数和返回值传递，不需要另一套 `throw/catch` 通道。

它的官方定义是一个内置接口：

```go
type error interface {
    Error() string
}
```

第 05 章已经学过接口：接口只规定行为，不关心具体类型。这里的 `error` 就是一个只要求一种行为的小接口：**只要某个类型实现了 `Error() string` 方法，它的值就可以赋给 `error`。**

所以 `error` 不是特殊异常对象，也不是一套藏起来的控制流。它就是一个接口值；里面可以装标准库的错误，也可以装你自己定义并实现了 `Error() string` 的类型。

对比 JS：JS 的错误走"另一条通道"——`throw` 把它抛出正常流程，`catch` 在别处接住；Go 的错误就在**主流程里**，跟着返回值一起回来。

---

## 2. 创建错误

两个造错误的工厂，都在标准库里：

固定的一句话，用 `errors.New`：

```go
package main

import (
    "errors"
    "fmt"
)

func main() {
    err := errors.New("文件不存在")
    fmt.Println(err)
    fmt.Printf("%T\n", err)   // 看看它的真实类型
}
```

```text
文件不存在
*errors.errorString
```

看到没，`err` 打印出来就是那句话，类型也不神秘——就是标准库里一个装着字符串的小盒子。**错误真的只是个值。**

需要往错误里塞变量，用 `fmt.Errorf`（和 `fmt.Printf` 同一套格式化占位符）：

```go
arg := "-1"
err := fmt.Errorf("无效的参数: %s", arg)
fmt.Println(err)
```

```text
无效的参数: -1
```

---

## 3. 返回错误

套路固定：错误放在**最后一个返回值**，成功时这个位置给 `nil`：

```go
func safeDivide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("除数不能为 0")   // 失败:结果给零值,错误给内容
    }
    return a / b, nil                          // 成功:错误位给 nil
}
```

`nil` 在这里的含义就是"无事发生"。调用方看到 `err == nil`，就知道结果可以放心用。

---

## 4. 处理错误

调用方的固定动作：**先查错，再用值**：

```go
result, err := safeDivide(10, 0)
if err != nil {
    fmt.Println("error:", err)
    return                        // 处理完就撤,别带着坏结果往下走
}
fmt.Println(result)
```

```text
error: 除数不能为 0
```

除数正常时走另一条路：

```go
result, err := safeDivide(10, 4)
if err != nil {
    fmt.Println("error:", err)
    return
}
fmt.Println(result)
```

```text
2.5
```

`if err != nil` 这四个词你会在 Go 代码里见到几百遍——这不是啰嗦，是 Go 的设计哲学：**每个可能出错的地方，错误处理就写在旁边**，一眼能看到哪步会失败、失败了怎么办。JS 的 `try` 块一包十几行，反而看不出具体哪一行会炸。

---

## 5. 不要随便忽略错误

🕳️ 坑：用 `_` 把错误扔掉，程序不会报错，但会**把失败当成功继续跑**：

```go
result, _ := safeDivide(10, 0)
fmt.Println(result)
```

```text
0
```

除法明明失败了，程序却拿着 `0` 继续算——这个 `0` 不是答案，是错误分支里随手填的零值。这种 bug 往往在很远的下游才炸出来，极难排查。

以为会怎样：除个零，大不了得到个怪值。实际怎样：得到一个**看起来完全正常的 0**，悄悄污染后续所有计算。为什么：错误信息被 `_` 吞了，程序失去了发现问题的唯一线索。

**规则：忽略错误之前，必须能说清"为什么这里可以忽略"。说不清，就老老实实 `if err != nil`。**

---

## 6. 错误信息要能定位问题

错误信息是写给"半夜看日志的自己"的。对比：

```go
return errors.New("invalid input")               // 什么 input?哪里 invalid?
return fmt.Errorf("read config %s failed", path) // 哪个动作、哪个文件,清清楚楚
```

有上下文就带上上下文（文件路径、参数值、用户 ID）。另外 Go 的惯例：错误信息**小写开头、结尾不加标点**，因为它经常被拼接到更长的错误链里（下一篇讲错误包装时你会看到拼接的样子）。

---

## 本篇重点

- Go 没有 try/catch，错误是普通返回值，随主流程返回，类型是内置的 `error`。
- 造错误：固定文案用 `errors.New`，带变量用 `fmt.Errorf`。
- 返回约定：错误放最后一个返回值，成功给 `nil`。
- 处理约定：调用后立刻 `if err != nil`，处理完尽早 `return`。
- 用 `_` 吞掉错误 = 把失败伪装成成功，是最难查的 bug 来源之一。

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

提示：错误信息里带上 `n` 的实际值（`fmt.Errorf` + `%d`），方便调用方定位；在 `main` 里分别用 `-1` 和 `5` 调用验证两条路径。
