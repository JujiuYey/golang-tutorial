# 07: struct 方法、接收者与嵌入

方法让类型拥有行为。struct 配合方法，可以把数据和操作数据的逻辑组织在一起。

---

## 1. 定义方法

```go
type User struct {
    Name string
    Age  int
}

func (u User) Greet() string {
    return "hello, " + u.Name
}
```

`(u User)` 是接收者。它表示 `Greet` 是 `User` 类型的方法。

调用：

```go
u := User{Name: "alice"}
fmt.Println(u.Greet())
```

---

## 2. 值接收者

值接收者会复制接收者。

```go
func (u User) Rename(name string) {
    u.Name = name
}

func main() {
    u := User{Name: "alice"}
    u.Rename("bob")
    fmt.Println(u.Name) // alice
}
```

`Rename` 修改的是副本，所以调用方的 `u` 没变。

值接收者适合：

- 方法不需要修改接收者。
- 类型较小，复制成本低。
- 你希望这个类型表现得像普通值。

---

## 3. 指针接收者

指针接收者可以修改原值。

```go
func (u *User) Rename(name string) {
    u.Name = name
}

func main() {
    u := User{Name: "alice"}
    u.Rename("bob")
    fmt.Println(u.Name) // bob
}
```

指针接收者适合：

- 方法需要修改接收者。
- 类型较大，复制成本不值得。
- 类型中包含锁、缓冲区、连接等不应该被复制的字段。
- 你希望所有方法都使用一致的接收者风格。

---

## 4. Go 会自动取地址和解引用

```go
u := User{Name: "alice"}
u.Rename("bob")
```

如果 `Rename` 是 `*User` 方法，`u.Rename("bob")` 会被编译器处理成类似 `(&u).Rename("bob")`。

指针变量调用值接收者方法也可以：

```go
p := &User{Name: "alice"}
fmt.Println(p.Greet())
```

编译器会自动解引用。

---

## 5. nil 指针接收者

指针接收者方法有可能被 nil 指针调用。

```go
func (u *User) IsZero() bool {
    return u == nil || (u.Name == "" && u.Age == 0)
}
```

只有在方法内部显式处理 nil 时，这种写法才安全。否则解引用 nil 指针会 panic。

```go
func (u *User) NameLength() int {
    return len(u.Name) // 如果 u 为 nil，会 panic
}
```

---

## 6. 接收者命名

接收者名字通常短小，并和类型名有关：

```go
func (u User) Greet() string {
    return "hello, " + u.Name
}

func (s Store) Save() error {
    return nil
}

func (c Client) Get() error {
    return nil
}
```

不要把接收者都叫 `this` 或 `self`。Go 里没有固定的接收者关键字。

---

## 7. struct 嵌入

嵌入是一种组合方式。

```go
type Address struct {
    City string
}

type User struct {
    Name string
    Address
}

u := User{
    Name: "alice",
    Address: Address{
        City: "Shanghai",
    },
}

fmt.Println(u.Address.City)
fmt.Println(u.City)
```

`u.City` 是提升字段访问。它让外层类型可以直接访问嵌入字段的字段。

---

## 8. 嵌入方法提升

```go
type Logger struct{}

func (Logger) Print(msg string) {
    fmt.Println(msg)
}

type Service struct {
    Logger
}

func main() {
    s := Service{}
    s.Print("started")
}
```

`Service` 嵌入了 `Logger`，因此可以直接调用提升后的 `Print` 方法。

嵌入不是继承。它不会建立父子类体系，核心仍然是组合。

---

## 9. 字段冲突

如果外层和嵌入字段有同名字段，直接访问时优先外层字段。

```go
type A struct {
    Name string
}

type B struct {
    Name string
    A
}

b := B{
    Name: "outer",
    A:    A{Name: "inner"},
}

fmt.Println(b.Name)   // outer
fmt.Println(b.A.Name) // inner
```

有冲突时，显式写完整路径更清晰。

---

## 10. 方法值

方法可以像函数一样保存到变量里。

```go
u := User{Name: "alice"}
greet := u.Greet

fmt.Println(greet())
```

方法值会绑定接收者。接收者是值还是指针，会影响后续是否看到修改。

```go
u := User{Name: "alice"}
greet := u.Greet

u.Name = "bob"
fmt.Println(greet()) // 取决于 Greet 的接收者和值绑定时的复制
```

这类细节不需要死记。遇到方法值和闭包结合的代码，要主动判断接收者是否被复制。

---

## 练习

1. 给 `Book` 定义 `Summary() string` 方法。
2. 给 `Counter` 定义 `Inc()` 方法，先用值接收者观察结果，再改成指针接收者。
3. 定义 `User` 嵌入 `Address`，访问提升字段。
4. 定义 `Service` 嵌入 `Logger`，调用提升方法。
5. 写一个 nil 指针接收者方法，并说明它为什么安全或不安全。
