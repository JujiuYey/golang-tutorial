# 05: map 的内存语义、排序与并发边界

map 变量本身可以被复制，但复制后仍然指向同一份底层哈希表。理解这一点，才能看懂 map 赋值、传参和修改的行为。

---

## 1. map 赋值会共享底层数据

```go
a := map[string]int{"x": 1}
b := a

b["x"] = 100

fmt.Println(a["x"]) // 100
fmt.Println(b["x"]) // 100
```

赋值复制的是 map 变量本身。两个变量都可以访问同一份底层数据。

---

## 2. map 作为参数

```go
func addScore(scores map[string]int, name string, score int) {
    scores[name] = score
}

func main() {
    scores := map[string]int{}
    addScore(scores, "alice", 100)
    fmt.Println(scores["alice"]) // 100
}
```

函数参数仍然是值传递。这里复制的是 map 变量，复制后的变量仍然指向同一份底层数据，所以函数里写入键值会影响调用方看到的 map 内容。

---

## 3. 重新赋值不会改掉调用方的 map 变量

```go
func reset(scores map[string]int) {
    scores = make(map[string]int)
    scores["alice"] = 100
}

func main() {
    scores := map[string]int{"bob": 90}
    reset(scores)
    fmt.Println(scores) // map[bob:90]
}
```

`reset` 里只是让参数变量指向新的 map，没有改变调用方变量 `scores` 本身。

如果确实要让调用方拿到新 map，更清晰的方式是返回新值：

```go
func reset() map[string]int {
    return map[string]int{"alice": 100}
}
```

---

## 4. map 元素不可取地址

不能直接修改 map 中 struct 值的字段。

```go
type User struct {
    Name string
    Age  int
}

users := map[string]User{
    "alice": {Name: "alice", Age: 18},
}

// users["alice"].Age++ // 编译错误
```

原因是 map 扩容和搬迁时，元素位置可能变化。Go 不允许拿到 map 元素的稳定地址。

正确写法是取出、修改、放回：

```go
u := users["alice"]
u.Age++
users["alice"] = u
```

也可以让 map 保存指针：

```go
users := map[string]*User{
    "alice": {Name: "alice", Age: 18},
}

users["alice"].Age++
```

保存指针后要额外注意 nil 指针和共享可变状态。

---

## 5. map 不能直接比较

map 只能和 `nil` 比较。

```go
var m map[string]int
fmt.Println(m == nil)
```

不能比较两个 map 内容是否相等：

```go
// a := map[string]int{"x": 1}
// b := map[string]int{"x": 1}
// fmt.Println(a == b) // 编译错误
```

如果要比较内容，需要遍历键值，或在合适章节学习标准库里的 `maps.Equal`。

---

## 6. 稳定输出需要排序

map 遍历顺序不稳定。如果要稳定打印、测试或生成输出，先取出 key 并排序。

```go
scores := map[string]int{
    "bob":   90,
    "alice": 100,
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

稳定输出是写测试、生成文档、输出 CLI 结果时非常重要的习惯。

---

## 7. 遍历时修改 map

遍历 map 时删除当前或尚未遍历的 key 是允许的，但不要依赖复杂行为。

```go
for key := range scores {
    if scores[key] == 0 {
        delete(scores, key)
    }
}
```

遍历过程中新增 key，新增项是否会在本轮遍历中出现是不确定的。清晰的做法是先收集，再统一修改：

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

普通 map 不支持并发读写。

```go
// 一个 goroutine 写 map，另一个 goroutine 同时读或写 map，可能触发运行时错误或数据竞争。
```

并发场景常见选择：

- 用 `sync.Mutex` 保护 map。
- 用 channel 把 map 操作集中到一个 goroutine。
- 特定读多写少场景考虑 `sync.Map`。

并发会在后续章节详细展开。现在先记住：普通 map 不是并发安全容器。

---

## 9. 删除不一定立刻释放所有内存

`delete` 会删除键值关系，但 map 的底层空间是否马上归还给系统，不应该作为业务假设。

如果一个 map 曾经非常大，之后要长期保留但只剩少量数据，可以考虑创建新 map，把需要的数据复制过去。

```go
small := make(map[string]int, len(old))
for k, v := range old {
    if shouldKeep(k, v) {
        small[k] = v
    }
}
old = small
```

---

## 练习

1. 写函数 `CloneMap(m map[string]int) map[string]int`，确保修改返回值不影响原 map。
2. 写函数 `SortedKeys(m map[string]int) []string`，返回排序后的 key。
3. 创建 `map[string]User`，尝试直接修改字段，观察编译错误，再改成取出、修改、放回。
4. 创建 `map[string]*User`，修改字段，解释为什么这次可以生效。
