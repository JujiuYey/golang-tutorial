# 04: map 基础：键值、读取与遍历

这篇学 Go 的键值容器 map——对应你在 JS 里的对象字面量和 `Map`，用来做索引、计数、缓存、集合、按 ID 查对象。但三个地方和 JS 直觉正面冲突：**读不存在的 key 不给 `undefined` 给零值、nil map 一写就 panic、遍历顺序故意随机**。这篇把三个雷都排了。

---

## 1. 声明 map

```go
var scores map[string]int

fmt.Println(scores == nil)
fmt.Println(len(scores))
```

```text
true
0
```

`map[string]int` 读作"key 是 string、value 是 int 的 map"。只声明不初始化，得到 **nil map**。

nil map 读、求长度、删除都没事：

```go
fmt.Println(scores["alice"])
delete(scores, "alice")
```

```text
0
```

### 四种写法分两阵营

`var` 给的是 nil；`make` 和 `{}` 都给**非 nil 的空 map**——`len()` 全是 0，但 `m == nil` 答案不同：

| 写法 | `m == nil` | `len(m)` | 能写吗 |
|---|---|---|---|
| `var m map[string]int` | true | 0 | 不能（panic，见下） |
| `m := map[string]int{}` | false | 0 | 能 |
| `m := make(map[string]int)` | false | 0 | 能 |
| `m := make(map[string]int, 100)` | false | 0 | 能（多一个容量提示） |

对比 slice：`var s []int` 是 nil slice，`s := []int{}` 和 `s := make([]int, 0)` 都是非 nil 空 slice——但 slice 那边 nil 和空行为几乎一样（都能 append），map 这边不一样：nil map 能读/删/`len` 但**不能写**，空 map 读写删都行。这正是为什么 map 用之前必须 make 或字面量，slice 用之前可以光 var。

### 🕳️ 坑：nil map 不能写

以为会怎样：像 nil slice 一样，append/写入自动就能用。
实际怎样：一写就 panic。
为什么：slice 的 append 会返回新 slice 头，nil 能"无中生有"；map 的写入是原地操作，没有底层哈希表就没地方放。

```go
package main

func main() {
	var scores map[string]int
	scores["alice"] = 100
}
```

```text
panic: assignment to entry in nil map

goroutine 1 [running]:
main.main()
	./main.go:5 +0x34
exit status 2
```

**口诀：nil slice 能 append，nil map 不能写——map 用之前必须 make 或字面量初始化。**

---

## 2. 使用 make 创建 map

```go
scores := make(map[string]int)
scores["alice"] = 100
scores["bob"] = 90
```

可以给个容量提示：

```go
scores := make(map[string]int, 100)
```

这个 100 不是长度也不是上限，只是告诉运行时"大概会装这么多"，让它少做几次扩容。类比 slice 的 `make([]int, 0, n)` 预分配。

---

## 3. map 字面量

```go
scores := map[string]int{
    "alice": 100,
    "bob":   90,
}
```

注意最后一项 `"bob": 90,` 结尾的逗号**不能省**——Go 的多行字面量强制尾逗号（编译器靠它自动插分号）。JS 里尾逗号是可选风格，Go 里是硬规定。

---

## 4. 读取值

```go
scores := map[string]int{
    "alice": 100,
}

fmt.Println(scores["alice"])
fmt.Println(scores["bob"])
```

```text
100
0
```

### 🕳️ 坑：不存在的 key 返回零值，不是 undefined

以为会怎样：`scores["bob"]` 像 JS 一样给个 `undefined`，一眼识破"没这人"。
实际怎样：安静地返回 value 类型的零值——`int` 就是 `0`。
为什么：Go 没有 undefined。于是"bob 考了 0 分"和"根本没有 bob"读出来一模一样。

要区分这两种情况，用 **comma ok**。它和第 03 章的多返回值是同一种接收方式：第一个值是查询结果，第二个 `bool` 表示键是否存在：

```go
score, ok := scores["bob"]
fmt.Println(score, ok)
```

```text
0 false
```

拆开看：

```text
score, ok := scores["bob"]
//  ┬    ┬
//  值   bool：key 存在为 true，不存在为 false（此时 score 是零值）
```

**一句话总结：读 map 拿一个值是"零值兜底"，拿两个值才知道 key 到底在不在。**

---

## 5. 写入与更新

添加和更新是同一个语法，key 在就覆盖、不在就新增：

```go
scores := make(map[string]int)

scores["alice"] = 100 // 添加
scores["alice"] = 95  // 更新
```

"零值兜底"反过来也是福利——计数器不用先判断 key 存不存在：

```go
counts := make(map[string]int)

for _, word := range []string{"go", "go", "rust"} {
    counts[word]++    // 第一次遇到时 counts[word] 读出 0，加 1 后写回
}

fmt.Println(counts)
```

```text
map[go:2 rust:1]
```

JS 里得写 `counts[word] = (counts[word] ?? 0) + 1`，Go 的零值直接把 `?? 0` 省了。

---

## 6. 删除键

```go
delete(scores, "alice")
```

删除不存在的 key 也不报错，静默无事发生：

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

只要 key 或只要 value 的写法，和 slice 的 range 一致：

```go
for name := range scores { ... }     // 只遍历键
for _, score := range scores { ... } // 只遍历值
```

### 🕳️ 坑：遍历顺序是故意打乱的

同一个 map 连续遍历三次：

```go
big := map[string]int{"a": 1, "b": 2, "c": 3, "d": 4, "e": 5}
for round := 0; round < 3; round++ {
    for k := range big {
        fmt.Print(k, " ")
    }
    fmt.Println()
}
```

```text
e a b c d 
c d e a b 
e a b c d 
```

每轮顺序都可能不同。这不是 bug——Go 运行时**故意随机化**遍历起点，就是防止你写出依赖顺序的代码。对比 JS：对象和 `Map` 的遍历顺序有规范保证（插入序），这个依赖搬到 Go 必炸。要稳定顺序怎么办？下一篇给方案（先取 key 排序）。

---

## 8. map 的键必须可比较

能当 key 的类型必须支持 `==` 比较（map 内部要靠它判断"是不是同一个 key"）。

| 能当 key | 不能当 key |
|------|------|
| `string`、整数、浮点数、布尔 | slice |
| 指针 | map |
| 数组（元素可比较） | function |
| struct（所有字段可比较） | |

规律很好记：上一篇说 slice 不能 `==`，所以它当不了 key；后面会看到 map、function 同理。struct 当 key 很实用：

```go
type Point struct {
    X int
    Y int
}

visited := map[Point]bool{}
visited[Point{X: 1, Y: 2}] = true

fmt.Println(visited[Point{X: 1, Y: 2}])
fmt.Println(visited[Point{X: 3, Y: 4}])
```

```text
true
false
```

注意是**按内容**匹配的：两次写的 `Point{X: 1, Y: 2}` 是两个独立的值，但字段相等就是同一个 key。JS 的 `Map` 用对象当 key 时比的是引用，两个内容相同的对象是不同的 key——又一个反着来的地方。

---

## 9. 用 map 表示集合

Go 没有内置 Set，惯用法是拿 map 凑：

```go
seen := make(map[string]bool)
seen["go"] = true

if seen["go"] {
    fmt.Println("seen")
}
```

`map[string]bool` 直白好用（不存在的 key 读出 `false`，刚好）。追求极致的写法是 value 用空 struct，零内存占用：

```go
seen := make(map[string]struct{})
seen["go"] = struct{}{}

if _, ok := seen["go"]; ok {
    fmt.Println("seen")
}
```

```text
seen
```

拆开那行怪符号：

```text
seen["go"] = struct{}{}
//           ──┬───  ┬
//        空struct类型 │
//              它的字面量（没有字段，所以花括号是空的）
```

日常写 `map[T]bool` 就够，`struct{}` 版本看得懂即可。

---

## 10. 嵌套 map

```go
users := make(map[string]map[string]int)
```

外层 make 了，**内层还是 nil map**——直接写内层就撞上第 1 节的 panic：

```go
package main

func main() {
	users := make(map[string]map[string]int)
	users["alice"]["score"] = 100
}
```

```text
panic: assignment to entry in nil map

goroutine 1 [running]:
main.main()
	./main.go:5 +0x98
exit status 2
```

正确姿势：写内层之前先确认它初始化过：

```go
if users["alice"] == nil {
    users["alice"] = make(map[string]int)
}

users["alice"]["score"] = 100
```

---

## 本篇重点

- [ ] nil map 能读不能写，写入直接 panic；用前必须 `make` 或字面量初始化（嵌套 map 的内层同理）。
- [ ] 读不存在的 key 返回零值而不是 undefined；要区分"值是零"和"key 不存在"，用 comma ok。
- [ ] 零值兜底让计数器一行搞定：`counts[word]++`。
- [ ] map 遍历顺序被运行时故意随机化——JS 对象"插入序"的依赖在 Go 必炸。
- [ ] key 必须可比较：slice/map/function 当不了 key；struct key 按字段内容匹配（不是按引用）。

---

## 练习

1. 用 `map[string]int` 统计一组单词出现次数。
2. 写函数 `Has(seen map[string]struct{}, key string) bool`。
3. 创建 `map[string][]string`，表示城市到街道列表的映射。
4. 读取一个不存在的 key，分别观察 `map[string]int`、`map[string]string`、`map[string][]int` 的结果。

提示：第 2 题里 comma ok 的两个返回值刚好一个用一个丢；第 3 题往某个城市追加街道时，想想 nil slice 能不能直接 append（第 02 篇）——这正是它和嵌套 map 的差别；第 4 题重点看第三个：nil slice 打印出来长什么样。
