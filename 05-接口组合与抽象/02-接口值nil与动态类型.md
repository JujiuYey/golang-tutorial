# 02: 接口值、nil 与动态类型

接口值里包含两部分：动态类型和动态值。很多 nil 相关 bug 都来自没有分清这两层。

---

## 1. 接口值的概念模型

```go
var r io.Reader
```

此时 `r` 是 nil 接口值。它没有动态类型，也没有动态值。

当赋值：

```go
var b *bytes.Buffer = bytes.NewBufferString("hello")
var r io.Reader = b
```

接口值 `r` 可以理解为：

- 动态类型：`*bytes.Buffer`
- 动态值：`b` 指向的那个 buffer

---

## 2. nil 接口

```go
var r io.Reader
fmt.Println(r == nil) // true
```

没有动态类型、没有动态值的接口，才等于 nil。

---

## 3. typed nil 放入接口

```go
var b *bytes.Buffer = nil
var r io.Reader = b

fmt.Println(b == nil) // true
fmt.Println(r == nil) // false
```

`r` 的动态类型是 `*bytes.Buffer`，动态值是 nil 指针。接口值本身已经不是 nil。

这类值经常被称为 typed nil。

---

## 4. typed nil 的实际风险

```go
type MyError struct {
    Message string
}

func (e *MyError) Error() string {
    return e.Message
}

func returnsError() error {
    var err *MyError = nil
    return err
}

func main() {
    err := returnsError()
    fmt.Println(err == nil) // false
}
```

`returnsError` 返回的是 `error` 接口。虽然里面的指针值是 nil，但接口有动态类型 `*MyError`，所以 `err != nil`。

更好的写法是在没有错误时直接返回 nil：

```go
func returnsError() error {
    var err *MyError = nil
    if err != nil {
        return err
    }
    return nil
}
```

---

## 5. nil 接收者方法

有些类型允许 nil 指针调用方法。

```go
type Node struct {
    Value int
}

func (n *Node) IsNil() bool {
    return n == nil
}
```

这种方法内部先判断 nil，所以安全。

不安全的写法：

```go
func (n *Node) ValueOrZero() int {
    return n.Value // n 为 nil 时 panic
}
```

接口调用方法时，是否 panic 取决于具体方法如何处理 nil 接收者。

---

## 6. 接口之间赋值

接口值可以赋给另一个接口，只要动态类型满足目标接口。

```go
var r io.Reader = strings.NewReader("hello")
var anyValue any = r

fmt.Println(anyValue)
```

`any` 可以保存任意类型的值，包括接口值。

---

## 7. 打印动态类型

```go
var r io.Reader = strings.NewReader("hello")

fmt.Printf("%T\n", r) // *strings.Reader
```

`%T` 打印接口中保存的动态类型。

---

## 8. 接口调用是动态分派

```go
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
```

`Say` 调用 `s.Speak()` 时，运行时会根据接口值里的动态类型调用具体方法。

---

## 9. 接口值会复制动态值

```go
type Counter struct {
    N int
}

func (c Counter) Value() int {
    return c.N
}

c := Counter{N: 1}
var v interface{ Value() int } = c

c.N = 10
fmt.Println(v.Value()) // 1
```

把 `c` 放入接口时，接口里保存的是当时的 `Counter` 值副本。

如果放入指针：

```go
c := Counter{N: 1}
var v interface{ Value() int } = &c

c.N = 10
fmt.Println(v.Value()) // 10
```

接口保存的是指针值的副本，指针仍指向同一个 `c`。

---

## 10. 判断接口 nil 的习惯

函数返回接口类型时：

- 没有值就直接返回 `nil`。
- 避免把 typed nil 指针直接作为接口返回。
- 接收接口参数时，不要只靠 `x == nil` 判断内部动态值是否可用。

尤其是返回 `error` 时，要非常小心 typed nil。

---

## 练习

1. 写代码验证 nil 接口 `var r io.Reader` 等于 nil。
2. 写代码验证 `var b *bytes.Buffer = nil; var r io.Reader = b` 后 `r != nil`。
3. 自定义一个指针错误类型，复现 typed nil error。
4. 把结构体值放入接口，修改原结构体，观察接口里的值是否变化。
5. 把结构体指针放入接口，再观察修改结果。
