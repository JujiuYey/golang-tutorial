# 06: struct 基础：字段、字面量与零值

struct 用来把多个字段组合成一个新的数据类型。它是 Go 表达业务对象、配置、请求响应和内部状态的核心工具。

---

## 1. 定义 struct

```go
type User struct {
    ID   int
    Name string
    Age  int
}
```

`User` 是一个新类型，里面有三个字段。

创建零值：

```go
var u User
fmt.Printf("%+v\n", u) // {ID:0 Name: Age:0}
```

struct 的零值是所有字段各自的零值。

---

## 2. 访问和修改字段

```go
u := User{}

u.ID = 1
u.Name = "alice"
u.Age = 18

fmt.Println(u.Name)
```

字段访问使用点号。

---

## 3. 使用字段名初始化

推荐使用字段名初始化：

```go
u := User{
    ID:   1,
    Name: "alice",
    Age:  18,
}
```

优点：

- 字段顺序变化时不容易出错。
- 可以只初始化部分字段。
- 可读性更好。

只初始化部分字段：

```go
u := User{
    Name: "alice",
}

fmt.Println(u.ID)  // 0
fmt.Println(u.Age) // 0
```

---

## 4. 按顺序初始化

```go
u := User{1, "alice", 18}
```

这种写法要求字段数量和顺序完全匹配。字段一多，或者类型以后新增字段，就很容易出错。

教学、测试里的短结构体可以用；业务代码里通常优先使用字段名。

---

## 5. 匿名 struct

匿名 struct 适合临时组合数据。

```go
config := struct {
    Host string
    Port int
}{
    Host: "localhost",
    Port: 8080,
}

fmt.Println(config.Host)
```

如果这个结构会在多个地方出现，应该定义成命名类型。

---

## 6. 嵌套 struct

```go
type Address struct {
    City   string
    Street string
}

type User struct {
    Name    string
    Address Address
}

u := User{
    Name: "alice",
    Address: Address{
        City:   "Shanghai",
        Street: "Century Ave",
    },
}

fmt.Println(u.Address.City)
```

嵌套 struct 适合表达“一个对象拥有另一个对象”的关系。

---

## 7. 字段可见性

字段名首字母大写，表示导出字段，可以被其他包访问。

```go
type User struct {
    Name string // 导出
    age  int    // 未导出
}
```

在同一个包内可以访问 `age`。其他包只能访问 `Name`。

这个规则也适用于类型名、函数名、方法名和常量变量名。

---

## 8. struct tag

struct tag 是字段上的元信息，常用于 JSON、数据库、表单绑定等场景。

```go
type User struct {
    ID   int    `json:"id"`
    Name string `json:"name"`
}
```

tag 本身只是字符串。具体含义由读取它的库决定，比如 `encoding/json` 会根据 `json:"name"` 决定 JSON 字段名。

---

## 9. struct 比较

如果所有字段都可比较，那么 struct 也可比较。

```go
type Point struct {
    X int
    Y int
}

p1 := Point{X: 1, Y: 2}
p2 := Point{X: 1, Y: 2}

fmt.Println(p1 == p2) // true
```

如果字段里有 slice、map、function，这个 struct 就不能直接比较：

```go
type Group struct {
    Names []string
}

// fmt.Println(Group{} == Group{}) // 编译错误
```

---

## 10. struct 赋值会复制整个值

```go
u1 := User{Name: "alice", Age: 18}
u2 := u1

u2.Age = 20

fmt.Println(u1.Age) // 18
fmt.Println(u2.Age) // 20
```

struct 是值。赋值会复制字段。如果字段本身是 slice、map、指针等引用语义的值，那么复制后的字段仍可能共享底层数据。

```go
type Team struct {
    Members []string
}

t1 := Team{Members: []string{"alice", "bob"}}
t2 := t1

t2.Members[0] = "carol"

fmt.Println(t1.Members) // [carol bob]
```

这里复制了 `Team`，也复制了 slice 头，但两个 slice 头仍指向同一个底层数组。

---

## 练习

1. 定义 `Book`，字段包括 `Title`、`Author`、`Pages`。
2. 用字段名初始化一个 `Book`，再只初始化 `Title` 观察其他字段零值。
3. 定义 `User` 和 `Address`，让 `User` 包含 `Address`。
4. 定义一个包含 slice 字段的 struct，赋值复制后修改 slice 元素，观察两个 struct 的变化。
5. 定义一个可比较的 `Point`，把它作为 map 的 key。
