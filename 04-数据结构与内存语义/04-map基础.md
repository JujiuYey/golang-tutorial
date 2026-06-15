# 04: map 基础：键值、读取与遍历

map 用来表示键到值的映射。它适合做索引、计数、缓存、集合和按 ID 查找对象。

---

## 1. 声明 map

```go
var scores map[string]int

fmt.Println(scores == nil) // true
fmt.Println(len(scores))   // 0
```

nil map 可以读取、求长度、删除键，但不能写入。

```go
fmt.Println(scores["alice"]) // 0
delete(scores, "alice")      // 可以执行

// scores["alice"] = 100     // panic: assignment to entry in nil map
```

---

## 2. 使用 make 创建 map

```go
scores := make(map[string]int)
scores["alice"] = 100
scores["bob"] = 90
```

可以提供容量提示：

```go
scores := make(map[string]int, 100)
```

容量提示不是长度，也不是上限，只是给运行时一个预估规模，方便减少扩容成本。

---

## 3. map 字面量

```go
scores := map[string]int{
    "alice": 100,
    "bob":   90,
}
```

多行字面量最后一项也要保留逗号：

```go
scores := map[string]int{
    "alice": 100,
    "bob":   90,
}
```

---

## 4. 读取值

```go
scores := map[string]int{
    "alice": 100,
}

fmt.Println(scores["alice"]) // 100
fmt.Println(scores["bob"])   // 0
```

键不存在时，读取结果是值类型的零值。对于 `map[string]int`，零值是 `0`。

如果要判断键是否存在，使用 comma ok：

```go
score, ok := scores["bob"]
if !ok {
    fmt.Println("bob does not exist")
    return
}

fmt.Println(score)
```

---

## 5. 写入与更新

```go
scores := make(map[string]int)

scores["alice"] = 100 // 添加
scores["alice"] = 95  // 更新
```

计数场景可以直接利用零值：

```go
counts := make(map[string]int)

for _, word := range []string{"go", "go", "rust"} {
    counts[word]++
}

fmt.Println(counts) // map[go:2 rust:1]
```

---

## 6. 删除键

```go
delete(scores, "alice")
```

删除不存在的键不会报错：

```go
delete(scores, "not-exist")
```

---

## 7. 遍历 map

```go
scores := map[string]int{
    "alice": 100,
    "bob":   90,
}

for name, score := range scores {
    fmt.Println(name, score)
}
```

只遍历键：

```go
for name := range scores {
    fmt.Println(name)
}
```

只遍历值：

```go
for _, score := range scores {
    fmt.Println(score)
}
```

map 的遍历顺序不保证稳定。不要写依赖 map 遍历顺序的业务逻辑。

---

## 8. map 的键必须可比较

可以作为 map key 的类型必须支持 `==` 比较。

常见可用 key：

- `string`
- 整数、浮点数、布尔值
- 指针
- 数组，前提是数组元素也可比较
- struct，前提是所有字段都可比较

不能作为 key 的常见类型：

- slice
- map
- function

例如：

```go
type Point struct {
    X int
    Y int
}

visited := map[Point]bool{}
visited[Point{X: 1, Y: 2}] = true
```

---

## 9. 用 map 表示集合

Go 没有内置 set 类型，常用 map 表示集合。

```go
seen := make(map[string]bool)
seen["go"] = true

if seen["go"] {
    fmt.Println("seen")
}
```

如果只关心键是否存在，也可以用 `struct{}` 节省一点空间：

```go
seen := make(map[string]struct{})
seen["go"] = struct{}{}

if _, ok := seen["go"]; ok {
    fmt.Println("seen")
}
```

---

## 10. 嵌套 map

```go
users := make(map[string]map[string]int)

if users["alice"] == nil {
    users["alice"] = make(map[string]int)
}

users["alice"]["score"] = 100
```

嵌套 map 写入前，要确保内层 map 已经初始化。

---

## 练习

1. 用 `map[string]int` 统计一组单词出现次数。
2. 写函数 `Has(seen map[string]struct{}, key string) bool`。
3. 创建 `map[string][]string`，表示城市到街道列表的映射。
4. 读取一个不存在的 key，分别观察 `map[string]int`、`map[string]string`、`map[string][]int` 的结果。
