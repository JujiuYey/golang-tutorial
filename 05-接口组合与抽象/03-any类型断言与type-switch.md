# 03: 空接口、any、类型断言与 type switch

这篇解决"值进了 `any` 之后怎么安全取出来"的问题。学完你能用类型断言和 type switch 处理 JSON 解码、日志字段这类"类型未知"的数据,并且知道 `any` 什么时候是工具、什么时候是偷懒。

先校准坐标：`any` 让 Go 变量获得一点 JS 变量的味道——什么都能装。但 JS 变量装完就能随便用（`v.length`、`v()`,错了运行时再说）；Go 的 `any` 装进去容易,**用之前必须先把类型验明正身**,否则编译器直接拦住你。

---

## 1. interface{} 和 any

一个方法都不要求的接口,就是"零门槛能力清单"——任何类型都满足它,所以任何值都能装：

```go
var a interface{} = 100
var b any = "hello"
```

从 Go 1.18 开始,`any` 就是 `interface{}` 的官方别名：

```go
type any = interface{}
```

两个写法完全等价。现代 Go 代码优先写 `any`——短,而且一眼读出"任意值"的意思。旧代码和旧文章里的 `interface{}` 见到别慌,是同一个东西。

---

## 2. any 的能力很弱

装得进不等于用得了。`any` 的含义是"我不知道它是什么",而不知道类型,就没有任何具体能力可调：

```go
func Print(v any) {
    fmt.Println(v)      // OK:Println 本来就收 any
}

func Use(v any) {
    // v.Len() // 编译错误:any 没有 Len 方法
}
```

对比 JS：`v.length` 在 JS 里编译不拦你,大不了运行时 undefined;Go 里 `v.Len()` 直接编译不过。**编译器只认接口清单上写了的方法,`any` 的清单是空的。**

想用具体能力,就得先把类型取出来——往下看。

---

## 3. 类型断言：把值从盒子里取出来

上一篇说接口是"双层盒子"。类型断言就是**开盒验货**：猜盒子里是某个类型,猜对了取出来用。

```go
package main

import (
    "fmt"
    "strings"
)

func main() {
    var v any = "hello"

    s, ok := v.(string)
    if ok {
        fmt.Println(strings.ToUpper(s))
    }
}
```

```text
HELLO
```

拆开这个新语法：

```text
s, ok := v.(string)
//┬  ┬     ─┬──────
//│  │      └ ③断言:「我猜 v 里装的是 string」
//│  └ ②猜没猜中(bool)
//└ ①猜中时:取出来的 string 值;没猜中:零值
```

`v.(string)` 只能用在接口值上,念作"断言 v 是 string"。带 `ok` 的写法(comma ok,和 map 取值 `m[k]` 的双返回值同款)猜错不炸,是业务代码的标准姿势。

---

## 4. 🕳️ 坑：不带 ok 直接断言,猜错就 panic

断言还有个单返回值形式 `s := v.(string)`——猜错直接程序崩溃：

```go
func main() {
    var v any = 100
    s := v.(string)
    fmt.Println(s)
}
```

```text
panic: interface conversion: interface {} is int, not string

goroutine 1 [running]:
main.main()
	/.../main.go:7 +0x34
exit status 2
```

panic 信息倒是很诚实：`is int, not string`——盒子里是 int,你非说是 string。

只有在你**百分百确定**动态类型时才用单返回值形式。大多数业务代码里,comma ok 更稳。

**一句话总结：断言默认带 ok;不带 ok 等于签了"猜错就崩"的免责声明。**

---

## 5. 断言到接口：探测"额外能力"

断言的目标不只能是具体类型,还可以是**另一个接口**——用来问"你除了基本功能,还会不会这一手":

```go
type Flusher interface {
    Flush() error
}

func FlushIfPossible(v any) error {
    f, ok := v.(Flusher)   // 问:你会 Flush 吗?
    if !ok {
        return nil          // 不会,跳过
    }
    return f.Flush()        // 会,那就调
}
```

标准库常玩这一手：比如 HTTP 框架拿到一个 `io.Writer`,断言它是不是也实现了 `Flusher`,是就顺手刷缓冲。**可选能力,探测着用。**

---

## 6. type switch：一次开盒,多路分发

要按类型分好几种情况处理,连着写断言太啰嗦——Go 给了专用语法 type switch：

```go
package main

import "fmt"

func PrintType(v any) {
    switch x := v.(type) {
    case nil:
        fmt.Println("nil")
    case int:
        fmt.Println("int:", x)
    case string:
        fmt.Println("string:", x)
    case []int:
        fmt.Println("[]int:", x)
    default:
        fmt.Printf("unknown: %T\n", x)
    }
}

func main() {
    PrintType(nil)
    PrintType(42)
    PrintType("go")
    PrintType([]int{1, 2})
    PrintType(3.14)
}
```

```text
nil
int: 42
string: go
[]int: [1 2]
unknown: float64
```

拆开这个新语法：

```text
switch x := v.(type) {
//     ┬      ──┬───
//     │        └ ②固定写法,字面上就写 type,只能出现在 switch 里
//     └ ①每个 case 分支里,x 自动变成该 case 的具体类型
case int:
    // 这里 x 是 int,能直接做算术
case string:
    // 这里 x 是 string,能直接调字符串函数
```

最妙的是 `x` 在每个分支里**自动带上对应类型**,不用再手动断言。还记得 02 模块说 Go 的 switch 不贯穿吗?这里同样,每个 case 独立,匹配到就走、走完就出。

---

## 7. 🕳️ 坑：any 不是泛型

以为会怎样：参数是 `[]any`,那 `[]int` 应该能传进去(JS 里数组就是数组)。
实际怎样：编译错误。

```go
func First(items []any) any {
    if len(items) == 0 {
        return nil
    }
    return items[0]
}

func main() {
    nums := []int{1, 2, 3}
    fmt.Println(First(nums))
}
```

```text
# mod05/a03_anyslice
./main.go:14:20: cannot use nums (variable of type []int) as []any value in argument to First
```

为什么：`[]int` 和 `[]any` 是两种**内存布局完全不同**的切片——`[]any` 的每个元素是一个"双层盒子",`[]int` 的每个元素就是裸的 int。转换需要逐个装盒,Go 不肯偷偷做这种 O(n) 的事。

想写"任意元素类型的切片都能收"的函数,正确工具是泛型(第 06 篇)：

```go
func First[T any](items []T) (T, bool) {
    var zero T
    if len(items) == 0 {
        return zero, false
    }
    return items[0], true
}
```

先混个眼熟：`[T any]` 里的 `any` 是约束,函数收 `[]int` 时 T 就是 int,类型信息不丢。

---

## 8. any 的适用场景

| 适合 any | 不适合 any |
|---|---|
| 日志字段、调试输出 | 为了省掉类型设计 |
| JSON 解码到未知结构 | 明明只接受一种类型 |
| 框架边界暂时无法知道类型 | 想保留输入输出类型关系(用泛型) |
| 确实需要 type switch 分支处理 | 可以用小接口表达行为的地方 |

**一句话总结：any 是"类型真的未知"时的逃生口,不是"懒得想类型"的默认值。**

---

## 9. any 与 map：每读一次,断言一次

`map[string]any` 是 JSON 场景的常客,长得很像 JS 对象：

```go
data := map[string]any{
    "name": "alice",
    "age":  18,
}
```

但和 JS 对象不同,读出来的值是 `any`,用之前还得断言：

```go
name, ok := data["name"].(string)
if !ok {
    fmt.Println("name 不是 string")
    return
}
fmt.Println(name)
```

```text
alice
```

这也是为什么不要滥用 `map[string]any`：类型信息在进 map 那一刻就丢了,后续每一次使用都要付一次"断言税"。结构已知时,老老实实定义 struct。

---

## 10. 🕳️ 坑：断言失败给的是零值,分不清"失败"和"真的是零值"

```go
func main() {
    var v any = 100
    s, ok := v.(string)

    fmt.Println(s)
    fmt.Println(ok)
}
```

```text

false
```

第一行是空的——断言失败时 `s` 是 string 的零值 `""`(所以打印出一个空行)。

问题来了：如果盒子里装的**本来就是** `""`,断言成功,`s` 也是 `""`。只看 `s` 你分不清这两种情况——**所以判断成败只能看 `ok`,不能看值**。和 map 的 comma ok 是同一个道理(零值歧义)。

---

## 本篇重点

- [ ] `any` = `interface{}`:什么都能装,但装进去就"失忆",不断言不能用任何具体能力。
- [ ] 类型断言 `v.(T)` 开盒验货:带 comma ok 猜错不炸;单返回值形式猜错 panic。
- [ ] 多类型分发用 `switch x := v.(type)`,每个 case 里 x 自动是具体类型。
- [ ] `[]int` 传不进 `[]any`——any 不是泛型;要保留元素类型,用 `[T any]`。
- [ ] 断言失败给零值,判断成败只看 `ok`;`map[string]any` 每次读取都要交"断言税",能定义 struct 就别用它。

---

## 练习

1. 写函数 `Describe(v any) string`，用 type switch 区分 `nil`、`int`、`string`、`[]string`。
2. 写函数 `AsString(v any) (string, bool)`。
3. 写函数 `FlushIfPossible(v any) error`，断言到 `interface{ Flush() error }`。
4. 验证 `[]int` 不能直接传给 `[]any` 参数。
5. 把一个 `map[string]any` 中的字段安全取成 `string` 和 `int`。

提示：第 1 题注意 type switch 里 `case nil` 要放着;第 4 题把编译错误原文抄下来读一遍;第 5 题每个字段都走 comma ok,想想 `age` 断言成 `int` 失败时你怎么区分"没这个键"和"类型不对"。
