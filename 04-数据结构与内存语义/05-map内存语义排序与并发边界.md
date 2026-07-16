# 05: map 的内存语义、排序与并发边界

这篇解决"map 传来传去之后，改动到底算谁的"——赋值、传参之后两个变量指向**同一份哈希表**（这回 Go 终于和你的 JS 对象直觉一致了），顺便处理两个工程必踩的点：怎么让 map 稳定输出、为什么并发读写会直接把程序打死。

---

## 1. map 赋值会共享底层数据

```go
a := map[string]int{"x": 1}
b := a

b["x"] = 100

fmt.Println(a["x"])
fmt.Println(b["x"])
```

```text
100
100
```

改 `b`，`a` 跟着变——和 JS 的 `const b = a`（对象）行为一致。原理：map 变量本身是个很小的**句柄**（内部就是指向底层哈希表的指针），`b := a` 复制的是句柄，两个句柄指向同一张表：

```text
a ──┐
    ├──▶ 同一份底层哈希表 {x: 1}
b ──┘
```

对照前两篇排个座次：**数组赋值 = 复制全部元素；slice 赋值 = 复制窗口（共享底层数组）；map 赋值 = 复制句柄（共享整张表）。** 共享程度一个比一个彻底。

顺带拆一个常见误会：有人管 map 叫"引用类型"并推出"所以它分配在堆上"。前半句凑合，后半句不成立——**值/引用说的是"复制时共享不共享"，栈/堆说的是"内存分配在哪"，是两个维度的问题**。第 08 篇会用编译器的逃逸分析输出正面拆这件事。

---

## 2. map 作为参数

```go
func addScore(scores map[string]int, name string, score int) {
    scores[name] = score
}

func main() {
    scores := map[string]int{}
    addScore(scores, "alice", 100)
    fmt.Println(scores["alice"])
}
```

```text
100
```

注意措辞：Go 的传参**永远是值传递**，这里复制的是 map 句柄这个"值"。副本句柄仍指向同一张表，所以函数里写 key，调用方看得见。和 JS 函数收到对象参数的感觉完全一样。

---

## 3. 重新赋值不会改掉调用方的 map 变量

共享的边界在哪？让参数**指向别处**时：

```go
func reset(scores map[string]int) {
    scores = make(map[string]int)   // 只是让"参数这个句柄"指向新表
    scores["alice"] = 100
}

func main() {
    scores := map[string]int{"bob": 90}
    reset(scores)
    fmt.Println(scores)
}
```

```text
map[bob:90]
```

调用方毫发无损。JS 里同样如此（函数里 `obj = {}` 改不了外面的变量）——这个直觉可以放心迁移。**改表里的内容，外面看得见；给句柄换一张表，外面看不见。** 真要给调用方一张新表，返回它：

```go
func reset() map[string]int {
    return map[string]int{"alice": 100}
}
```

---

## 4. map 元素不可取地址

### 🕳️ 坑：不能直接改 map 里 struct 值的字段

以为会怎样：`users["alice"].Age++`，和 JS 改嵌套对象一样顺手。
实际怎样：编译错误。
为什么：map 扩容搬迁时元素会挪窝，Go 拒绝给你一个"可能随时失效的地址"，干脆禁止对 map 元素做原地修改。

```go
package main

import "fmt"

type User struct {
	Name string
	Age  int
}

func main() {
	users := map[string]User{
		"alice": {Name: "alice", Age: 18},
	}

	users["alice"].Age++

	fmt.Println(users)
}
```

```text
# command-line-arguments
./main.go:15:2: cannot assign to struct field users["alice"].Age in map
```

正确姿势是**取出、修改、放回**三步：

```go
u := users["alice"]
u.Age++
users["alice"] = u

fmt.Println(users["alice"])
```

```text
{alice 19}
```

或者让 map 存**指针**，指针指向的对象不归 map 管、地址稳定，就能直接改：

```go
users := map[string]*User{
    "alice": {Name: "alice", Age: 18},
}

users["alice"].Age++
fmt.Println(users["alice"].Age)
```

```text
19
```

代价是要开始操心 nil 指针和"多处共享同一个可变对象"（第 08 篇的主题）。

| 写法 | 修改字段 | 风险 |
|------|------|------|
| `map[string]User` | 取出、改、放回 | 啰嗦但数据独立 |
| `map[string]*User` | 直接 `.Age++` | nil 指针、共享可变状态 |

---

## 5. map 不能直接比较

和 slice 一样，map 只能和 `nil` 比：

```go
package main

import "fmt"

func main() {
	a := map[string]int{"x": 1}
	b := map[string]int{"x": 1}
	fmt.Println(a == b)
}
```

```text
# command-line-arguments
./main.go:8:14: invalid operation: a == b (map can only be compared to nil)
```

要比内容，自己遍历，或用标准库的 `maps.Equal`（标准库模块再展开）。

---

## 6. 稳定输出需要排序

上一篇看过 map 遍历顺序故意随机。要稳定打印（写测试、生成文档、CLI 输出都需要），套路是**取出 key → 排序 → 按序访问**：

```go
scores := map[string]int{
    "bob":   90,
    "alice": 100,
    "carol": 95,
}

names := make([]string, 0, len(scores))
for name := range scores {
    names = append(names, name)
}

sort.Strings(names)

for _, name := range names {
    fmt.Println(name, scores[name])
}
```

```text
alice 100
bob 90
carol 95
```

三步各自都是学过的东西：range 收集 key（预分配容量，第 03 篇）、`sort.Strings` 排序、按序取值。这个组合值得肌肉记忆。

---

## 7. 遍历时修改 map

遍历中**删除** key 是明确允许的：

```go
m := map[string]int{"a": 0, "b": 2, "c": 0, "d": 4}
for key := range m {
    if m[key] == 0 {
        delete(m, key)
    }
}
fmt.Println(m)
```

```text
map[b:2 d:4]
```

但遍历中**新增** key 就进灰色地带了——新项本轮遍历可能出现也可能不出现，规范就是这么写的。别赌。复杂改动用"先收集，再统一动手"：

```go
var toDelete []string
for key, score := range scores {
    if score == 0 {
        toDelete = append(toDelete, key)
    }
}

for _, key := range toDelete {
    delete(scores, key)
}
```

---

## 8. 并发边界

普通 map **不允许并发读写**。这不是"结果可能不对"级别的问题——运行时检测到就会直接终止进程。并发和 recover 都还没学，这里先记住结论：不要一边写 map，一边让其他 goroutine 读写它。

```go
m := make(map[int]int)

go func() {                 // 一个 goroutine 在写（go 关键字并发模块细讲）
    for i := 0; ; i++ {
        m[i] = i
    }
}()

for i := 0; ; i++ {         // main 也在写同一个 map
    m[i] = i
}
```

```text
fatal error: concurrent map writes

goroutine 6 [running]:
internal/runtime/maps.fatal({0x1047c02da?, 0x0?})
	runtime/panic.go:1046 +0x20
main.main.func1()
	./main.go:8 +0x38
created by main.main in goroutine 1
	./main.go:6 +0x60
（后面还有各 goroutine 的堆栈，此处截断）
```

注意开头是 `fatal error` 不是 `panic:`——**fatal error 无法 recover**。并发场景的常见选择（并发模块细讲，先记名字）：

- 用 `sync.Mutex` 给 map 加锁。
- 用 channel 把所有 map 操作集中到一个 goroutine。
- 读多写少的特定场景用 `sync.Map`。

**现阶段记住一句：普通 map 不是并发安全容器，多 goroutine 碰同一个 map = 进程暴毙。**

---

## 9. 删除不一定立刻释放所有内存

`delete` 删的是键值关系，底层哈希表占的空间未必马上归还系统——别把"删完内存就降"当成业务假设。如果一个 map 曾经巨大、之后长期只剩少量数据，可以新建小表搬家：

```go
small := make(map[string]int, len(old))
for k, v := range old {
    if shouldKeep(k, v) {
        small[k] = v
    }
}
old = small   // 旧的大表没人引用了，交给 GC
```

和第 03 篇"切小片拽住大数组"是同一类问题：**持有大结构的一小部分，可能让整个大结构无法回收。**

---

## 本篇重点

- [ ] map 赋值/传参复制的是句柄，指向同一张哈希表——改内容外面可见，换表（重新赋值）外面不可见，和 JS 对象直觉一致。
- [ ] "引用语义"≠"分配在堆上"：共享与否和栈/堆是两个维度，第 08 篇用逃逸分析实证。
- [ ] map 元素不可取地址：改 struct 值字段要"取出、改、放回"，或改存指针（接受 nil 与共享的风险）。
- [ ] 稳定输出三步走：收集 key → `sort.Strings` → 按序访问；map 之间不能 `==`。
- [ ] 并发读写普通 map 是 `fatal error`，recover 都救不了；边遍历边新增 key 行为不确定，先收集再修改。

---

## 练习

1. 写函数 `CloneMap(m map[string]int) map[string]int`，确保修改返回值不影响原 map。
2. 写函数 `SortedKeys(m map[string]int) []string`，返回排序后的 key。
3. 创建 `map[string]User`，尝试直接修改字段，观察编译错误，再改成取出、修改、放回。
4. 创建 `map[string]*User`，修改字段，解释为什么这次可以生效。

提示：第 1 题想想为什么不能 `b := a` 完事（第 1 节刚做过实验）；第 4 题的答案就藏在第 4 节的表格里，重点是"map 管的是指针这个值，不管指针指向的东西"。
