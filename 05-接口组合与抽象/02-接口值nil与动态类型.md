# 02: 接口值、nil 与动态类型

这篇解决 Go 面试和实战里最著名的一个坑：**函数明明返回了 nil 指针,`err != nil` 却是 true**。看懂"接口值是一个双层盒子"这件事,这类 bug 你不但能躲开,还能给别人讲明白。

---

## 1. 接口值的概念模型：双层盒子

一个接口变量在内部装两样东西：

```text
接口值
┌──────────────────┐
│ 动态类型  type    │ ← 里面装的是什么类型
│ 动态值    value   │ ← 具体的那个值
└──────────────────┘
```

刚声明、没赋值时,两格都是空的：

```go
var r io.Reader   // type: 无, value: 无 —— 这才是 nil 接口
```

赋一个值进去,两格同时被填上：

```go
var b *bytes.Buffer = bytes.NewBufferString("hello")
var r io.Reader = b
// r 的 type:  *bytes.Buffer
// r 的 value: b 指向的那个 buffer
```

对比 JS：JS 的变量就是"一个值",没有这层包装。Go 的接口变量更像 `{ type, value }` 的元组——后面所有的坑都出在"有人只填了 type,没填 value"。

**一句话总结：接口值 = 动态类型 + 动态值,两格。**

---

## 2. nil 接口：两格都空才是 nil

```go
var r io.Reader
fmt.Println(r == nil)
```

```text
true
```

判空规则只有一条：**动态类型和动态值都为空,接口才等于 nil。** 只要 type 格被填过,哪怕 value 格装的是 nil 指针,整个接口就不再是 nil——这就是下一节的坑。

---

## 3. 🕳️ 坑：typed nil —— 装着 nil 指针的接口不是 nil

以为会怎样：`b` 是 nil，把它赋给接口 `r`，`r` 当然也是 nil。
实际怎样：`b == nil` 是 true，`r == nil` 却是 false。

```go
package main

import (
    "bytes"
    "fmt"
    "io"
)

func main() {
    var b *bytes.Buffer = nil
    var r io.Reader = b

    fmt.Println(b == nil)
    fmt.Println(r == nil)
}
```

```text
true
false
```

为什么：赋值 `r = b` 时，Go 把 `b` 的**类型**也一起装进了盒子：

```text
r 的盒子
┌────────────────────────┐
│ type:  *bytes.Buffer   │ ← 填上了!
│ value: nil             │
└────────────────────────┘
     type 格非空 → r != nil
```

这种"type 有、value 是 nil 指针"的接口值,行话叫 **typed nil**。它和真正的 nil 接口(两格全空)用 `==` 一比就露馅。

| | 动态类型 | 动态值 | `== nil` |
|---|---|---|:---:|
| `var r io.Reader` | 无 | 无 | true |
| `var r io.Reader = (*bytes.Buffer)(nil)` | `*bytes.Buffer` | nil | **false** |

---

## 4. typed nil 的实际风险：error 永远"非空"

这个坑 95% 的出场方式是 `error`（`error` 本身就是接口）。函数用具体指针类型存错误、再当接口返回,就中招了：

```go
package main

import "fmt"

type MyError struct {
    Message string
}

func (e *MyError) Error() string {
    return e.Message
}

func returnsError() error {
    var err *MyError = nil
    return err            // 把 *MyError 类型的 nil 装进 error 接口
}

func main() {
    err := returnsError()
    fmt.Println(err == nil)
    if err != nil {
        fmt.Println("进入了错误处理分支!")
    }
}
```

```text
false
进入了错误处理分支!
```

明明没有错误，调用方却走进了错误分支——因为 `return err` 时接口的 type 格被填成了 `*MyError`。

修法很简单：**没有错误就返回字面量 nil**，别让具体类型的 nil 指针"过一遍手"：

```go
func returnsError() error {
    var err *MyError = nil
    if err != nil {
        return err
    }
    return nil   // 字面量 nil:两格全空的干净 nil 接口
}
```

**口诀：返回接口时,要 nil 就返回写死的 nil,别返回可能为 nil 的具体指针变量。**

---

## 5. nil 接收者方法：能调用,但看方法自己

接上一个自然的问题：接口里装着 nil 指针,调用方法会不会 panic？答案是**看方法内部动不动 `n` 的字段**。

Go 允许在 nil 指针上调用方法（方法只是"接收者作为第一个参数的函数",nil 也是合法参数）：

```go
type Node struct {
    Value int
}

func (n *Node) IsNil() bool {
    return n == nil        // 只比较指针本身,安全
}

func main() {
    var n *Node
    fmt.Println(n.IsNil())
}
```

```text
true
```

但方法一旦解引用字段,就炸：

```go
func (n *Node) ValueOrZero() int {
    return n.Value         // n 为 nil 时解引用
}

func main() {
    var n *Node
    fmt.Println(n.ValueOrZero())
}
```

```text
panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x2 addr=0x0 pc=0x10229546c]

goroutine 1 [running]:
main.(*Node).ValueOrZero(...)
	/.../main.go:10
main.main()
	/.../main.go:15 +0x1c
exit status 2
```

**一句话总结：nil 指针调方法合法,方法碰字段才 panic——typed nil 接口调方法同理。**

---

## 6. 接口之间赋值

接口值可以转手装进另一个接口,只要动态类型满足目标接口。`any`（能装任意类型,下一篇细讲）是最宽的目标：

```go
package main

import (
    "fmt"
    "io"
    "strings"
)

func main() {
    var r io.Reader = strings.NewReader("hello")
    var v any = r

    fmt.Printf("%T\n", v)
}
```

```text
*strings.Reader
```

注意打印的是 `*strings.Reader` 不是 `io.Reader`——转手时搬运的是盒子里的**内容**（动态类型+动态值），不是盒子本身。

---

## 7. 打印动态类型：%T

调试接口值的第一工具是 `%T`,它掀开盒子看 type 格：

```go
var r io.Reader = strings.NewReader("hello")

fmt.Printf("%T\n", r)
```

```text
*strings.Reader
```

排查第 3 节那种 typed nil 时特别好用：`%T` 打出具体类型而不是 `<nil>`,说明 type 格被填过。

---

## 8. 接口调用是动态分派

调用接口方法时,运行时看盒子里的动态类型,找到对应实现再调——和 JS 里"对象上找方法"一个感觉：

```go
package main

import "fmt"

type Speaker interface {
    Speak() string
}

type Dog struct{}
func (Dog) Speak() string { return "woof" }

type Cat struct{}
func (Cat) Speak() string { return "meow" }

func Say(s Speaker) {
    fmt.Println(s.Speak())
}

func main() {
    Say(Dog{})
    Say(Cat{})
}
```

```text
woof
meow
```

同一行 `s.Speak()`，装 Dog 时汪、装 Cat 时喵——**代码在编译期只知道"会 Speak"，具体调谁运行时才定**（动态分派）。

---

## 9. 接口值会复制动态值

把值装进接口时装的是**副本**——这是上一模块"Go 一切赋值皆拷贝"在接口上的延续：

```go
package main

import "fmt"

type Counter struct {
    N int
}

func (c Counter) Value() int {
    return c.N
}

func main() {
    c := Counter{N: 1}
    var v interface{ Value() int } = c   // 装进去的是 c 的副本
    c.N = 10
    fmt.Println(v.Value())

    c2 := Counter{N: 1}
    var v2 interface{ Value() int } = &c2   // 装的是指针(的副本)
    c2.N = 10
    fmt.Println(v2.Value())
}
```

```text
1
10
```

| 装进接口的是 | 盒子里存的 | 改原变量后接口看到 |
|---|---|---|
| 值 `c` | 装入那一刻的副本 | 旧值(1) |
| 指针 `&c2` | 指针的副本,仍指向原件 | 新值(10) |

和 slice/函数传参的规律完全一致：**想让接口看到后续修改,装指针。**

---

## 10. 判断接口 nil 的习惯

把本篇的坑收拢成三条守则：

- 函数返回接口类型(尤其 `error`)时,没有值就返回字面量 `nil`。
- 别把可能为 nil 的具体指针变量直接 return 成接口——先判空。
- 接收接口参数时,`x == nil` 只能说明"盒子全空",不能说明"里面的指针可用";必要时用 `%T` 或下一篇的类型断言检查。

---

## 本篇重点

- [ ] 接口值是双层盒子：动态类型 + 动态值;两格全空才 `== nil`。
- [ ] typed nil：把 nil 具体指针赋给接口,type 格被填上,接口 `!= nil`——`error` 返回值是重灾区。
- [ ] 修法：没有错误就 `return nil`(字面量),别让具体指针类型过手。
- [ ] nil 指针可以调方法,方法内部解引用字段才 panic。
- [ ] 接口装值是拷贝副本,装指针才能看到后续修改;`%T` 打印动态类型是调试利器。

---

## 练习

1. 写代码验证 nil 接口 `var r io.Reader` 等于 nil。
2. 写代码验证 `var b *bytes.Buffer = nil; var r io.Reader = b` 后 `r != nil`。
3. 自定义一个指针错误类型，复现 typed nil error。
4. 把结构体值放入接口，修改原结构体，观察接口里的值是否变化。
5. 把结构体指针放入接口，再观察修改结果。

提示：第 3 题照第 4 节的骨架写,重点是让 `if err != nil` 误判;做完再改成正确版本对比;第 4、5 题用 `interface{ Value() int }` 这种匿名接口最省事。
