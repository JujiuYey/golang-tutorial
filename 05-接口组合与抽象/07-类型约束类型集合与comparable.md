# 07: 类型约束、类型集合与 comparable

上一篇留了个问题:`any` 约束下连 `+` 都用不了。这篇解决"怎么给类型参数放权"——约束写得越具体,函数体能做的操作越多。学完你能用 `comparable` 写查找函数、自己定义数值约束写 `Min`/`Max`,并搞懂那个神秘的 `~` 波浪号。

核心心智模型一句话:**约束是一份双向合同——圈定调用方能传哪些类型,同时圈定函数体能对 T 做哪些操作。**

---

## 1. any 约束：什么都能进,什么都不能干

```go
func Identity[T any](v T) T {
    return v
}
```

`any` 是最宽的约束:任何类型都能当实参。代价是函数体里只能做**所有类型都支持**的操作——赋值、返回、传给收 `any` 的函数(比如 `fmt.Println`),仅此而已。宽进则严出。

---

## 2. comparable 约束：解锁 ==

想用 `==` 比较,或把 T 当 map 的 key,就用内置约束 `comparable`：

```go
package main

import "fmt"

func Contains[T comparable](items []T, target T) bool {
    for _, item := range items {
        if item == target {     // 有 comparable 才能写这行
            return true
        }
    }
    return false
}

func main() {
    fmt.Println(Contains([]int{1, 2, 3}, 2))
    fmt.Println(Contains([]string{"go", "rust"}, "go"))
}
```

```text
true
true
```

回忆 04 模块:slice、map、函数是**不可比较**类型。它们自然也进不了 `comparable` 的圈——实跑验证：

```go
func main() {
    _ = Contains([][]int{{1}, {2}}, []int{1})
}
```

```text
# mod05/a07_contains_bad
./main.go:13:14: []int does not satisfy comparable
```

`does not satisfy comparable`——不满足约束,门口就被拦下。

**一句话总结：要用 `==` 或当 map key,约束写 `comparable`。**

---

## 3. 自定义约束：接口的新用法

`+`、`<` 这些操作没有内置约束,要自己圈类型。语法有点意外——用 `interface`,但里面列的不是方法,是**类型**：

```go
type Integer interface {
    int | int8 | int16 | int32 | int64 |
        uint | uint8 | uint16 | uint32 | uint64 | uintptr
}
```

拆开看：

```text
type Integer interface {
    int | int8 | int16 | ...
//  ─┬─ ┬
//   │  └ ②竖线:「或」,和 TS 的联合类型 int | int8 一个意思
//   └ ①列的是类型名,不是方法签名 → 这叫「类型集合」
}
```

用它当约束,`+` 就解锁了——因为圈里每个类型都支持加法：

```go
func Add[T Integer](a, b T) T {
    return a + b
}

func main() {
    fmt.Println(Add(1, 2))
    fmt.Println(Add(int64(10), int64(20)))
}
```

```text
3
30
```

对比上一篇 `Add[T any]` 的编译错误——同一段函数体,约束从 `any` 收窄到 `Integer`,`+` 就合法了。合同换了,权限就换了。

---

## 4. 🕳️ 坑：类型集合接口不能当普通变量类型

以为会怎样：`Number` 是 interface,那 `var n Number` 声明个变量总行吧。
实际怎样：编译错误。

```go
type Number interface {
    int | int64 | float64
}

func main() {
    var n Number
    _ = n
}
```

```text
# mod05/a07_var_bad
./main.go:8:8: cannot use type Number outside a type constraint: interface contains type constraints
```

为什么：`interface` 关键字现在身兼两职,含类型集合的那种只能站在约束的位置上：

| | 普通接口 | 约束接口 |
|---|---|---|
| 里面写的 | 方法签名 | 类型集合(可含方法) |
| 例子 | `Reader`、`error` | `Integer`、`comparable` |
| 能当变量/参数类型 | ✅ | ❌ |
| 能当泛型约束 | ✅(要求有这些方法) | ✅ |

报错原文 `cannot use type ... outside a type constraint` 就是在说:这种接口出不了方括号。

---

## 5. 近似类型 `~`：把"亲戚类型"也圈进来

回忆 02 模块:你可以基于 int 定义新类型 `type UserID int`。问题来了——`UserID` 算不算约束里的 `int`?

**不写 `~` 就不算。** 实跑对比,先看不带 `~` 的：

```go
type UserID int

func Max[T interface{ int | int64 }](a, b T) T {
    if a > b {
        return a
    }
    return b
}

func main() {
    fmt.Println(Max(UserID(1), UserID(2)))
}
```

```text
# mod05/a07_tilde_bad
./main.go:15:17: UserID does not satisfy interface{int | int64} (possibly missing ~ for int in interface{int | int64})
```

编译器连修法都提示了:`possibly missing ~`。加上波浪号：

```go
func Max[T interface{ ~int | ~int64 }](a, b T) T {
    if a > b {
        return a
    }
    return b
}

func main() {
    fmt.Println(Max(UserID(1), UserID(2)))
}
```

```text
2
```

```text
 int   → 只有 int 本人
~int   → int 本人 + 所有「底层类型是 int」的自定义类型(UserID、Age…)
```

自定义类型在 Go 业务代码里到处都是,所以**自己写约束时,类型前基本都该带 `~`**。

顺带一提:`interface{ ~int | ~int64 }` 这种直接写在方括号里的匿名约束是合法的,约束复用时才需要起名字。

---

## 6. 约束里也可以包含方法

类型集合和方法可以混着写,两条都要满足：

```go
type StringerNumber interface {
    ~int | ~int64       // 条件①:底层类型是 int 或 int64
    String() string     // 条件②:还得有 String() string 方法
}
```

这种约束读作"是这类数字,**并且**会自我描述"。用得不多,知道语法上允许即可。

---

## 7. ordered 约束：解锁 < 和 >

`comparable` 只管 `==`,不管大小比较。想用 `<`、`>`,要把所有支持排序的类型圈出来：

```go
package main

import "fmt"

type Ordered interface {
    ~int | ~int8 | ~int16 | ~int32 | ~int64 |
        ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 | ~uintptr |
        ~float32 | ~float64 |
        ~string
}

func Min[T Ordered](a, b T) T {
    if a < b {
        return a
    }
    return b
}

func main() {
    fmt.Println(Min(3, 1))
    fmt.Println(Min("go", "rust"))
    fmt.Println(Min(2.5, 1.5))
}
```

```text
1
go
1.5
```

注意 string 也在圈里——字符串按字典序比大小,所以 `Min("go", "rust")` 是 `"go"`。这份清单不用背,标准库 `cmp.Ordered` 就是它,实战直接 `import "cmp"` 拿来用。

| 想做的操作 | 需要的约束 |
|---|---|
| 赋值、返回、打印 | `any` 够了 |
| `==`、当 map key | `comparable` |
| `<`、`>` 排序 | `cmp.Ordered`(或自定义) |
| `+` 等算术 | 自定义数值约束 |

---

## 8. map key 约束

写泛型 map 工具函数时,K 的约束几乎总是 `comparable`——因为 map 的 key 本来就必须可比较(map 靠 `==` 判断 key 撞没撞)：

```go
package main

import "fmt"

func Keys[K comparable, V any](m map[K]V) []K {
    keys := make([]K, 0, len(m))
    for k := range m {
        keys = append(keys, k)
    }
    return keys
}

func main() {
    m := map[string]int{"a": 1, "b": 2}
    keys := Keys(m)
    fmt.Println(len(keys))
    fmt.Printf("%T\n", keys)
}
```

```text
2
[]string
```

`[K comparable, V any]` 这个组合以后你会写到吐:key 要能比较,value 随意。(打印 `len` 而不是 keys 本身,是因为 map 遍历顺序随机——04 模块的老知识。)

---

## 9. 约束越小越好

选约束和上一篇选接口一个逻辑:**函数体真正用到什么能力,就要什么**。

- 只需要判等 → `comparable`,别上 Ordered。
- 只需要打印 → `any` 就够。
- 需要加法 → 再定义数值约束。

约束越宽,调用方越自由;约束越窄,函数体权限越大——按需取平衡,不多要。

---

## 10. 不要到处定义 Number

最后一个设计提醒:数值约束容易被滥用。金额、年龄、库存、分数底层都是数字、都能相加,但"订单金额加库存数量"是一句业务胡话。

泛型约束该服务于**类型无关的通用算法**(求最小值、求和一个同类型切片);别为了"它们都是数字"就把语义不同的业务量塞进同一个泛型函数——编译器拦不住的错误,才是最贵的错误。

---

## 本篇重点

- [ ] 约束是双向合同:圈定谁能进,也圈定函数体能干什么;`any` 宽进严出。
- [ ] `==`/map key 用 `comparable`;slice、map、函数进不来(`does not satisfy comparable`)。
- [ ] 自定义约束 = interface 里列类型集合(`int | int64`);这种接口只能当约束,不能声明变量。
- [ ] `~int` 圈进所有底层是 int 的自定义类型;自己写约束基本都该带 `~`(编译器报错会提示 `possibly missing ~`)。
- [ ] `<`/`>` 用 `cmp.Ordered`;约束按函数体的真实需要选,越小越好。

---

## 练习

1. 实现 `Contains[T comparable](items []T, target T) bool`。
2. 实现 `Keys[K comparable, V any](m map[K]V) []K`。
3. 定义 `Ordered` 约束，实现 `Min[T Ordered](a, b T) T`。
4. 定义 `type UserID int`，测试 `~int` 和 `int` 约束的区别。
5. 写一个约束，要求类型底层是 `string` 并且有 `Validate() error` 方法。

提示：第 4 题把两种约束的版本都编译一遍,对照第 5 节的报错原文;第 5 题参考第 6 节的混合写法,记得给你的自定义 string 类型实现 `Validate` 方法再试着调用。
