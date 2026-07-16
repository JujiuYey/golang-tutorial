# 07: struct 方法、接收者与嵌入

这篇给 struct 装上行为（方法），核心是一个 JS 里不存在的选择题：**接收者用值还是用指针**——选错了，方法改半天字段，调用方那边纹丝不动。顺便学嵌入：Go 没有 class 继承，代码复用靠"组合 + 提升"。

---

## 1. 定义方法

方法就是"绑定了类型的函数"——在 `func` 和函数名之间多写一对括号：

```go
type User struct {
    Name string
    Age  int
}

func (u User) Greet() string {
    return "hello, " + u.Name
}
```

拆开这个新语法：

```text
func (u User) Greet() string
//   ───┬────
//   接收者：u 相当于 JS 方法里的 this，
//   只不过要显式命名、显式写类型
```

调用和 JS 一样用点号：

```go
u := User{Name: "alice"}
fmt.Println(u.Greet())
```

```text
hello, alice
```

对比 JS：class 方法里 `this` 是隐式的、行为还随调用方式漂移；Go 把它拍在明面上——接收者就是方法的第一个参数，规规矩矩走传参那套规则。这句话是下一节的伏笔。

---

## 2. 值接收者

**接收者遵守传参规则**意味着：值接收者 = 复制一份 struct 进方法。

```go
func (u User) Rename(name string) {
    u.Name = name    // 改的是复制进来的那一份
}

func main() {
    u := User{Name: "alice"}
    u.Rename("bob")
    fmt.Println(u.Name)
}
```

```text
alice
```

改名失败——`Rename` 拿到的 `u` 是副本，方法结束副本就丢了。这在 JS 里根本不可能发生（`this` 永远指向对象本身），所以值得专门记：**值接收者的方法，改字段改了个寂寞。**

值接收者适合：方法只读不写、类型小复制便宜、希望类型表现得像普通值（比如 `Point`、`time.Time` 这类）。

---

## 3. 指针接收者

想让方法真的改到调用方的数据，接收者写成指针：

```go
func (u *User) Rename(name string) {
    u.Name = name    // 通过地址找到原 struct 去改
}

func main() {
    u := User{Name: "alice"}
    u.Rename("bob")
    fmt.Println(u.Name)
}
```

```text
bob
```

生效了。指针接收者适合：

- 方法需要修改接收者（最常见的理由）。
- struct 较大，不想每次调用都复制。
- 类型里有锁、连接、缓冲区这类**不该被复制**的字段。
- 同一类型的方法风格要统一——只要有一个方法用指针接收者，其余的通常跟着用。

| 接收者 | 方法里拿到的 | 能改到原值吗 | JS 对应 |
|------|------|------|------|
| `(u User)` | 副本 | 不能 | 无对应，JS 做不到 |
| `(u *User)` | 地址 | 能 | 就是 `this` 的日常行为 |

---

## 4. Go 会自动取地址和解引用

按上一节的逻辑，`u` 是值、`Rename` 要指针，似乎得写 `(&u).Rename("bob")`？不用——编译器帮你：

```go
u := User{Name: "alice"}
u.Rename("bob")        // 编译器自动处理成 (&u).Rename("bob")
```

反过来也一样，指针调用值接收者方法会自动解引用：

```go
p := &User{Name: "alice"}
fmt.Println(p.Greet()) // 自动处理成 (*p).Greet()
```

```text
hello, alice
```

所以日常调用时几乎感觉不到接收者类型的存在——**语法糖抹平了调用侧，但"改不改得到原值"的语义差异一点没变**，这正是它偶尔坑人的原因。

---

## 5. nil 指针接收者

指针接收者方法可能被 nil 指针调上来。方法内部主动处理 nil，就是安全的：

```go
func (u *User) IsZero() bool {
    return u == nil || (u.Name == "" && u.Age == 0)
}

var np *User
fmt.Println(np.IsZero())
```

```text
true
```

没处理 nil 就解引用，程序会因为 `nil pointer dereference` 发生 panic。第 06 章会系统解释 panic，这里先把它理解为"运行时发现无法继续，立即停止当前流程"：

```go
package main

import "fmt"

type User struct {
	Name string
	Age  int
}

func (u *User) NameLength() int {
	return len(u.Name)
}

func main() {
	var u *User
	fmt.Println(u.NameLength())
}
```

```text
panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x2 addr=0x8 pc=0x10228d46c]

goroutine 1 [running]:
main.(*User).NameLength(...)
	./main.go:11
main.main()
	./main.go:16 +0x1c
exit status 2
```

注意一个反 JS 直觉的细节：`np.NameLength()` 里**调用本身不炸**（不是 JS 的 "cannot read property of null"），炸的是方法体里第一次真正解引用 `u.Name` 的那行。方法调用只是把 nil 当接收者参数传了进去。

---

## 6. 接收者命名

接收者名字惯例是类型名的首字母缩写，短小统一：

```go
func (u User) Greet() string  { ... }
func (s Store) Save() error   { ... }
func (c Client) Get() error   { ... }
```

不要写 `this` 或 `self`——不是语法禁止，是社区约定强烈反对。从 JS 迁移过来最容易手滑的就是这个。

---

## 7. struct 嵌入

在 struct 里只写类型不写字段名，就是**嵌入**：

```go
type Address struct {
    City string
}

type User struct {
    Name string
    Address        // ← 只有类型，没有字段名：嵌入
}
```

嵌入后，内层字段被**提升**到外层，两条路径都能访问：

```go
u := User{
    Name: "alice",
    Address: Address{
        City: "Shanghai",
    },
}

fmt.Println(u.Address.City)  // 完整路径
fmt.Println(u.City)          // 提升后的短路径
```

```text
Shanghai
Shanghai
```

注意初始化时字段名是类型名 `Address:`——嵌入字段其实有个隐式的名字，就是它的类型名。

---

## 8. 嵌入方法提升

字段能提升，方法也能——这是 Go 复用行为的主要方式：

```go
type Logger struct{}

func (Logger) Print(msg string) {
    fmt.Println(msg)
}

type Service struct {
    Logger    // 嵌入之后，Service"自动获得"Print 方法
}

func main() {
    s := Service{}
    s.Print("started")
}
```

```text
started
```

看着像 JS 的 `class Service extends Logger`，但**嵌入不是继承**：没有父子类型关系（`Service` 不"是一个"`Logger`，不能当 `Logger` 用），没有方法重写多态，只是"把内层的东西摆到外层够得着的地方"。心智模型是**组合 + 转发的语法糖**。

---

## 9. 字段冲突

外层和嵌入层字段重名时，短路径优先**外层**：

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

fmt.Println(b.Name)
fmt.Println(b.A.Name)
```

```text
outer
inner
```

规则：**路径越浅越优先**。有冲突时别依赖优先级，直接写完整路径，读代码的人少猜一步。

---

## 10. 方法值

方法可以像函数一样存进变量（03 模块"函数是一等公民"的延伸），这叫方法值——**存的那一刻接收者就被绑定**：

```go
func (u User) Greet() string  { return "hello, " + u.Name }
func (u *User) GreetP() string { return "hello, " + u.Name }

u := User{Name: "alice"}
greet := u.Greet    // 值接收者：绑定时复制了一份 u
greetP := u.GreetP  // 指针接收者：绑定的是 u 的地址

u.Name = "bob"

fmt.Println(greet())
fmt.Println(greetP())
```

```text
hello, alice
hello, bob
```

同一时刻绑定、同一时刻调用，结果却不同：值接收者在绑定瞬间**快照**了 `u`（改名与它无关），指针接收者拿的是地址（现场读到最新值）。这和第 03 章的"参数复制 vs 闭包读取外层变量"是同一个问题：关键都在于复制发生在哪一刻。

这类细节不用死记，遇到方法值 + 闭包的代码，主动问一句"接收者在绑定时被复制了吗"就够。

---

## 本篇重点

- [ ] 方法 = 绑定接收者的函数；接收者是显式版的 `this`，走普通传参规则。
- [ ] 值接收者拿副本，改字段无效；指针接收者拿地址，才能修改原值——JS 的 `this` 永远是后者，前者是 Go 特有的陷阱。
- [ ] 调用侧 `&`/`*` 由编译器自动补全，语法无感但语义有别；有一个方法用指针接收者，整个类型统一用。
- [ ] nil 指针能调方法，炸点在方法体内第一次解引用；防御式方法要先判 `u == nil`。
- [ ] 嵌入是组合不是继承：字段、方法被"提升"到外层，重名时外层优先；方法值在绑定时按接收者类型决定快照还是引用。

---

## 练习

1. 给 `Book` 定义 `Summary() string` 方法。
2. 给 `Counter` 定义 `Inc()` 方法，先用值接收者观察结果，再改成指针接收者。
3. 定义 `User` 嵌入 `Address`，访问提升字段。
4. 定义 `Service` 嵌入 `Logger`，调用提升方法。
5. 写一个 nil 指针接收者方法，并说明它为什么安全或不安全。

提示：第 2 题是第 2、3 节实验的复刻，连续调三次 `Inc` 再打印，两个版本的差距会非常直观；第 5 题对照第 5 节，找出你方法里"第一次解引用"发生在哪一行。
