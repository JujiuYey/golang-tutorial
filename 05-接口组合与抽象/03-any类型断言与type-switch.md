# 03: 空接口、any、类型断言与 type switch

`any` 是 `interface{}` 的别名。它表示任意类型的值都可以放进来。

---

## 1. interface{} 和 any

```go
var a interface{} = 100
var b any = "hello"
```

从 Go 1.18 开始，`any` 是 `interface{}` 的别名：

```go
type any = interface{}
```

现代 Go 代码里通常优先写 `any`，更短，也更能表达“任意值”。

---

## 2. any 的能力很弱

```go
func Print(v any) {
    fmt.Println(v)
}
```

`Print` 可以接收任意值，但它不能直接调用具体方法：

```go
func Use(v any) {
    // v.Len() // 编译错误
}
```

`any` 的含义是“我不知道它是什么”。不知道具体类型，就不能直接使用具体能力。

---

## 3. 类型断言

```go
var v any = "hello"

s, ok := v.(string)
if ok {
    fmt.Println(strings.ToUpper(s))
}
```

`v.(string)` 尝试把接口中的动态值取成 `string`。

推荐使用 comma ok 写法：

```go
s, ok := v.(string)
if !ok {
    return
}
fmt.Println(s)
```

---

## 4. 直接断言会 panic

```go
var v any = 100

// s := v.(string) // panic
```

只有在你非常确定动态类型时，才使用直接断言。大多数业务代码里，comma ok 更稳。

---

## 5. 断言到接口

类型断言不只能断言到具体类型，也可以断言到接口。

```go
type Flusher interface {
    Flush() error
}

func FlushIfPossible(v any) error {
    f, ok := v.(Flusher)
    if !ok {
        return nil
    }
    return f.Flush()
}
```

这种写法适合“如果传入值支持某个额外能力，就使用它”。

---

## 6. type switch

```go
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
```

`x := v.(type)` 只能写在 type switch 里。每个 case 中，`x` 会拥有对应的具体类型。

---

## 7. any 不是泛型

```go
func First(items []any) any {
    if len(items) == 0 {
        return nil
    }
    return items[0]
}
```

这个函数接收的是 `[]any`，不是“任意类型的 slice”。`[]int` 不能直接传给 `[]any` 参数。

```go
// nums := []int{1, 2, 3}
// First(nums) // 编译错误
```

如果想保留元素类型，需要用泛型：

```go
func First[T any](items []T) (T, bool) {
    var zero T
    if len(items) == 0 {
        return zero, false
    }
    return items[0], true
}
```

---

## 8. any 的适用场景

适合使用 `any`：

- 日志字段、调试输出。
- JSON 解码到未知结构。
- 框架边界暂时无法知道具体类型。
- type switch 确实需要分支处理多个类型。

不适合使用 `any`：

- 为了省掉类型设计。
- 明明只接受一种类型，却写成 `any`。
- 想保留输入输出类型关系的函数。
- 可以用小接口表达行为的地方。

---

## 9. any 与 map

```go
var data map[string]any = map[string]any{
    "name": "alice",
    "age":  18,
}
```

读取后仍然要断言：

```go
name, ok := data["name"].(string)
if !ok {
    return
}
fmt.Println(name)
```

过度使用 `map[string]any` 会让类型信息丢失，后续代码必须不断断言。

---

## 10. 类型断言失败时的零值

```go
var v any = 100
s, ok := v.(string)

fmt.Println(s)  // ""
fmt.Println(ok) // false
```

断言失败时，结果值是目标类型的零值。

这也是为什么必须检查 `ok`。只看 `s == ""`，分不清动态值真的就是空字符串，还是断言失败。

---

## 练习

1. 写函数 `Describe(v any) string`，用 type switch 区分 `nil`、`int`、`string`、`[]string`。
2. 写函数 `AsString(v any) (string, bool)`。
3. 写函数 `FlushIfPossible(v any) error`，断言到 `interface{ Flush() error }`。
4. 验证 `[]int` 不能直接传给 `[]any` 参数。
5. 把一个 `map[string]any` 中的字段安全取成 `string` 和 `int`。
