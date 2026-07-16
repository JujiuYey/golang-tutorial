# 06: struct 基础：字段、字面量与零值

这篇学 struct——Go 拿来表达业务对象、配置、请求响应的核心工具。你可以把它理解成"形状焊死的 JS 对象"：字段名、字段类型在编译期全部定死，不能像 JS 那样运行时随手 `obj.newField = 1`。换来的是零值可用、可整体复制、可直接比较这三样 JS 对象给不了的东西。

---

## 1. 定义 struct

```go
type User struct {
    ID   int
    Name string
    Age  int
}
```

`type User struct {...}` 定义了一个新类型（和 02 模块 `type Weekday int` 是同一个 `type` 关键字，只是这次组合了多个字段）。

不初始化直接用，得到的是**零值 struct**——每个字段各自取零值：

```go
var u User
fmt.Printf("%+v\n", u)
```

```text
{ID:0 Name: Age:0}
```

`%+v` 会带上字段名打印，调试 struct 的标配。对比 JS：没有 `undefined` 状态的对象——一个 `User` 从诞生起所有字段就都有值，这是 Go "零值可用"哲学的延续。

---

## 2. 访问和修改字段

点号访问，和 JS 一样：

```go
u := User{}

u.ID = 1
u.Name = "alice"
u.Age = 18

fmt.Println(u.Name)
```

```text
alice
```

---

## 3. 使用字段名初始化

推荐写法——带字段名：

```go
u := User{
    ID:   1,
    Name: "alice",
    Age:  18,
}
```

好处：字段顺序无所谓、以后类型加字段不受影响、可读性好。还可以只给一部分，剩下的自动零值：

```go
u := User{
    Name: "alice",
}

fmt.Println(u.ID)
fmt.Println(u.Age)
```

```text
0
0
```

长得像 JS 对象字面量，但注意两个差别：冒号前**没有引号**（是字段名不是字符串），多行时每行（包括最后一行）**必须尾逗号**。

---

## 4. 按顺序初始化

不写字段名也行，但必须按定义顺序、一个不少：

```go
u := User{1, "alice", 18}
fmt.Println(u)
```

```text
{1 alice 18}
```

字段一多、或者以后类型插一个新字段，这种写法就全线错位。教学示例和两三个字段的小类型可以用，业务代码优先带字段名。

---

## 5. 匿名 struct

不想专门起名字的临时组合，定义和初始化一次写完：

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

```text
localhost
```

拆开这个双花括号结构：

```text
struct {           }{           }
────────┬───────── ──────┬──────
① 类型定义（字段们）  ② 紧跟字面量（具体值）
```

这是最接近 JS "随手造个对象"的写法。但只要同一形状出现第二次，就该提成命名类型。

---

## 6. 嵌套 struct

字段的类型也可以是另一个 struct：

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

```text
Shanghai
```

链式点号访问，和 JS 嵌套对象一样。嵌套表达"一个对象拥有另一个对象"。

---

## 7. 字段可见性

字段名**首字母大小写**决定其他包能不能访问——Go 没有 `public`/`private` 关键字，命名就是权限声明：

```go
type User struct {
    Name string // 大写开头：导出，包外可见
    age  int    // 小写开头：未导出，只有本包能碰
}
```

这个规则不只管字段，类型名、函数名、方法名、常量变量名全都适用（01 模块提过 `fmt.Println` 大写的原因，就是它）。

---

## 8. struct tag

字段后面可以挂一段反引号字符串，叫 tag——字段的"元信息标签"：

```go
type User struct {
    ID   int    `json:"id"`
    Name string `json:"name"`
}
```

tag 本身只是字符串，Go 语言层面不赋予任何含义，**读它的库说了算**。比如 `encoding/json` 会按 `json:"id"` 决定 JSON 字段名：

```go
data, _ := json.Marshal(User{ID: 1, Name: "alice"})
fmt.Println(string(data))
```

```text
{"id":1,"name":"alice"}
```

没有 tag 的话输出会是 `{"ID":1,"Name":"alice"}`——Go 字段大写开头（第 7 节的可见性要求），而 JSON 惯例是小写，tag 就是两边的翻译官。数据库映射、表单校验也都是这个套路，后续模块大量出现。

---

## 9. struct 比较

所有字段都可比较的 struct，本身就能 `==`——**按字段逐个比内容**：

```go
type Point struct {
    X int
    Y int
}

p1 := Point{X: 1, Y: 2}
p2 := Point{X: 1, Y: 2}

fmt.Println(p1 == p2)
```

```text
true
```

对比 JS：`{x:1} === {x:1}` 是 `false`（比引用），Go 的 struct `==` 比的是内容——这也是上一篇 struct 能当 map key 的原因。

但只要字段里混进 slice、map 或 function（不可比较三人组），整个 struct 就失去比较能力：

```go
package main

import "fmt"

type Group struct {
	Names []string
}

func main() {
	fmt.Println(Group{} == Group{})
}
```

```text
# command-line-arguments
./main.go:10:14: invalid operation: Group{} == Group{} (struct containing []string cannot be compared)
```

---

## 10. struct 赋值会复制整个值

struct 和数组一样是**值**——赋值即整体复制：

```go
u1 := User{Name: "alice", Age: 18}
u2 := u1

u2.Age = 20

fmt.Println(u1.Age)
fmt.Println(u2.Age)
```

```text
18
20
```

`u1` 不受影响。JS 里 `const u2 = u1` 之后改 `u2.age` 会连带 `u1`——Go 这里是真复制。

### 🕳️ 坑：复制是"浅"的——slice/map 字段照旧共享

以为会怎样：既然整体复制了，两个 struct 就彻底独立。
实际怎样：字段本身被复制了，但 slice 字段复制的是 slice **头**（第 02 篇），底层数组还是同一个。
为什么：struct 复制 = 逐字段复制，每个字段按自己的语义来。

```go
type Team struct {
    Members []string
}

t1 := Team{Members: []string{"alice", "bob"}}
t2 := t1

t2.Members[0] = "carol"

fmt.Println(t1.Members)
fmt.Println(t2.Members)
```

```text
[carol bob]
[carol bob]
```

改 `t2`，`t1` 跟着变。画出来：

```text
t1.Members ──┐（两个独立的 slice 头）
             ├──▶ 同一个底层数组 [alice→carol, bob]
t2.Members ──┘
```

**一句话总结：struct 赋值复制所有字段，但字段里藏着的指针、slice 头、map 句柄，复制后依然指向原来的共享数据。** 想彻底独立，得对这些字段逐个手动克隆（第 03 篇的 clone）。

---

## 本篇重点

- [ ] struct 是"形状焊死的对象"：字段编译期定死，零值即可用（每个字段各取零值），`%+v` 打印带字段名。
- [ ] 初始化优先带字段名（可部分初始化 + 尾逗号必写）；匿名 struct 应付一次性组合。
- [ ] 首字母大小写 = 可见性：大写导出、小写包内私有，全 Go 通用规则。
- [ ] struct `==` 比的是字段内容（JS 比引用），但含 slice/map/function 字段就不可比较。
- [ ] struct 赋值整体复制、互不影响——但 slice/map 字段复制后仍共享底层数据（浅复制）。

---

## 练习

1. 定义 `Book`，字段包括 `Title`、`Author`、`Pages`。
2. 用字段名初始化一个 `Book`，再只初始化 `Title` 观察其他字段零值。
3. 定义 `User` 和 `Address`，让 `User` 包含 `Address`。
4. 定义一个包含 slice 字段的 struct，赋值复制后修改 slice 元素，观察两个 struct 的变化。
5. 定义一个可比较的 `Point`，把它作为 map 的 key。

提示：第 4 题是第 10 节实验的复刻，动手前先预测输出；第 5 题回顾上一篇第 8 节"struct key 按内容匹配"，试试用两个分别构造的 `Point` 读写同一个 key。
