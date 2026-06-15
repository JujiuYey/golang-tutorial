# 07: 类型约束、类型集合与 comparable

泛型函数里的类型参数必须有约束。约束决定这个类型参数允许哪些具体类型，以及函数体内能对它做什么操作。

---

## 1. any 约束

```go
func Identity[T any](v T) T {
    return v
}
```

`any` 允许任何类型。函数体里只能做所有类型都支持的操作，比如赋值、返回、传给接受 `any` 的函数。

---

## 2. comparable 约束

如果要使用 `==` 或把类型作为 map key，需要 `comparable`。

```go
func Contains[T comparable](items []T, target T) bool {
    for _, item := range items {
        if item == target {
            return true
        }
    }
    return false
}
```

调用：

```go
fmt.Println(Contains([]int{1, 2, 3}, 2))
fmt.Println(Contains([]string{"go", "rust"}, "go"))
```

slice、map、function 这类不可比较类型不能作为 `T comparable` 的类型实参。

---

## 3. 自定义约束

```go
type Integer interface {
    int | int8 | int16 | int32 | int64 |
        uint | uint8 | uint16 | uint32 | uint64 | uintptr
}
```

这个接口用在泛型约束位置，表示 T 可以是这些整数类型之一。

```go
func Add[T Integer](a, b T) T {
    return a + b
}
```

因为 `Integer` 的类型集合都支持 `+`，所以函数里可以使用加法。

---

## 4. 类型集合接口不能当普通值类型使用

包含类型项的接口只能作为约束。

```go
type Number interface {
    int | int64 | float64
}

func Add[T Number](a, b T) T {
    return a + b
}
```

不能这样用：

```go
// var n Number // 编译错误：包含类型集合的接口不能作为普通变量类型
```

普通接口描述方法集合，可以作为变量类型。泛型约束接口可以包含类型集合，主要用于类型参数约束。

---

## 5. 近似类型 `~`

```go
type Integer interface {
    ~int | ~int64
}
```

`~int` 表示底层类型是 `int` 的所有类型。

```go
type UserID int

func Max[T interface{ ~int | ~int64 }](a, b T) T {
    if a > b {
        return a
    }
    return b
}

fmt.Println(Max(UserID(1), UserID(2)))
```

如果约束写成 `int | int64`，`UserID` 这种自定义类型不会被包含。写成 `~int`，它就可以使用。

---

## 6. 约束里也可以包含方法

```go
type StringerNumber interface {
    ~int | ~int64
    String() string
}
```

这个约束要求类型的底层类型是 `int` 或 `int64`，并且拥有 `String() string` 方法。

只有同时满足类型集合和方法集合的类型，才能作为类型实参。

---

## 7. ordered 约束

如果要使用 `<`、`>`，需要约束类型都是可排序的。

```go
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
```

这里把所有支持 `<` 的常见类型列出来。

---

## 8. map key 约束

```go
func Keys[K comparable, V any](m map[K]V) []K {
    keys := make([]K, 0, len(m))
    for k := range m {
        keys = append(keys, k)
    }
    return keys
}
```

map 的 key 必须可比较，所以 `K` 需要 `comparable`。

---

## 9. 约束越小越好

如果只需要比较相等，用 `comparable`。

如果只需要打印，用 `any` 就够。

如果需要加法，再定义数值约束。

约束表达的是函数体真正需要的能力。约束越宽，调用方越自由；约束越窄，函数体可用操作越多。

---

## 10. 不要到处定义 Number

数值约束看起来方便，但容易被滥用。很多业务逻辑虽然都用数字，但语义不同。

例如金额、年龄、库存、分数都能相加，但它们不一定应该共用一个泛型算法。

泛型约束应该服务于清晰的通用逻辑，而不是把所有数字都混在一起。

---

## 练习

1. 实现 `Contains[T comparable](items []T, target T) bool`。
2. 实现 `Keys[K comparable, V any](m map[K]V) []K`。
3. 定义 `Ordered` 约束，实现 `Min[T Ordered](a, b T) T`。
4. 定义 `type UserID int`，测试 `~int` 和 `int` 约束的区别。
5. 写一个约束，要求类型底层是 `string` 并且有 `Validate() error` 方法。
