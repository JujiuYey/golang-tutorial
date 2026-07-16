# 03: slice 共享、append 与删除

这篇解决 slice 的头号事故源：**多个 slice 趴在同一个底层数组上**，改一个、另一个跟着变，append 一下还可能把邻居的数据踩了。JS 里两个数组永远各管各的，这套直觉搬到 Go 会连环翻车——下面全程用对照实验说话。

---

## 1. 多个 slice 可以共享底层数组

上一篇说过：切片表达式不复制元素，只是开一扇新窗口。验证一下"两扇窗口、一块玻璃"：

```go
nums := []int{10, 20, 30, 40, 50}
a := nums[1:4]

a[0] = 200

fmt.Println(a)
fmt.Println(nums)
```

```text
[200 30 40]
[10 200 30 40 50]
```

改的是 `a[0]`，`nums[1]` 也变了——因为它们本来就是底层数组里的**同一个格子**：

```text
底层数组   10   20→200  30   40   50
nums 窗口 └──────── 全部5格 ────────┘
a 窗口          └── a[0]..a[2] ──┘
```

对比 JS：`const a = nums.slice(1, 4)` 复制出独立数组，改 `a[0]` 绝不影响 `nums`。**Go 的切片是共享视图，JS 的 slice() 是复印件**——这是本篇所有坑的总根源。

---

## 2. 🕳️ 坑：append 容量足够时，会写进共享的底层数组

以为会怎样：`append(a, 99)` 只是给 `a` 加个元素，关 `nums` 什么事。
实际怎样：`nums[3]` 的 `40` 被 `99` 覆盖了。
为什么：`a` 的窗口右边还有备用格子（cap 有富余），append 直接往那格写——而那格正是 `nums[3]`。

```go
nums := []int{10, 20, 30, 40, 50}
a := nums[1:3]

fmt.Println(len(a), cap(a))

a = append(a, 99)

fmt.Println(a)
fmt.Println(nums)
```

```text
2 4
[20 30 99]
[10 20 30 99 50]
```

案发过程：

```text
append 前   10   20   30   40   50
a 窗口           └ len=2 ┘
a 的备用地盘            └── 40、50 ──┘

append 后   10   20   30   99   50
                          ↑ append 把 40 踩了，nums 躺枪
```

---

## 3. append 容量不足时会分配新数组

同样是 append，容量不够时行为完全不同——搬家：

```go
nums := []int{10, 20, 30}
a := nums[:3]

fmt.Println(len(a), cap(a))

a = append(a, 99)
a[0] = 100

fmt.Println(a)
fmt.Println(nums)
```

```text
3 3
[100 20 30 99]
[10 20 30]
```

`len == cap`，没有备用格子，append 只好**分配一个更大的新数组**、把旧元素复制过去。从这一刻起 `a` 和 `nums` 分家——所以之后 `a[0] = 100` 对 `nums` 毫无影响。

**一句话总结：append 是个"看容量脸色"的操作——cap 有富余就原地写（共享继续），不够就搬家（共享断开）。** 同一行代码两种结局，全看当时的 len/cap，这就是它难缠的地方。

---

## 4. 用完整切片表达式限制共享影响

不想让 append 踩到原数据？切的时候就用 `s[low:high:max]` 把备用地盘没收：

```go
nums := []int{10, 20, 30, 40, 50}
a := nums[1:3:3]

fmt.Println(len(a), cap(a))

a = append(a, 99)

fmt.Println(a)
fmt.Println(nums)
```

```text
2 2
[20 30 99]
[10 20 30 40 50]
```

`cap` 被掐到 2，append 时容量必然不足，只能走"搬家"路线——`nums` 完好无损。**把切出去给别人用的 slice 掐死 cap，是防御式编程的常用招。**

---

## 5. 预分配容量

append 搬一次家就要复制一次全部元素。看看放任它自己扩容时发生了什么：

```go
var g []int
for i := 0; i < 9; i++ {
    g = append(g, i)
    fmt.Printf("len=%d cap=%d\n", len(g), cap(g))
}
```

```text
len=1 cap=4
len=2 cap=4
len=3 cap=4
len=4 cap=4
len=5 cap=8
len=6 cap=8
len=7 cap=8
len=8 cap=8
len=9 cap=16
```

cap 按 4 → 8 → 16 跳着涨，每次跳变都是一次"分配新数组 + 整体复制"（具体数字是 Go 运行时的内部策略，不同版本可能不同，不用背）。如果事先知道大概要装多少，直接一次备够：

```go
func squares(n int) []int {
    result := make([]int, 0, n)   // len=0 从空开始，cap=n 空间备足
    for i := 0; i < n; i++ {
        result = append(result, i*i)
    }
    return result
}
```

```text
[0 1 4 9 16]
```

`make([]int, 0, n)` 这个 `0` 别写漏——写成 `make([]int, n)` 的话前 n 格已经是 0 值元素，append 会从第 n+1 格接着加，结果多出一堆 0。

---

## 6. 删除元素

Go 没有 `splice`，删除靠切片拼接。保序删除下标 `i`：

```go
func deleteAt(s []int, i int) []int {
    return append(s[:i], s[i+1:]...)
}
```

```text
[10 30 40]
```

（输入 `[10 20 30 40]`、删下标 1。）拆开这行：

```text
append(s[:i], s[i+1:]...)
//     ──┬──  ────┬─────
//   i 之前的部分  i 之后的部分逐个追加（... 是展开，03 模块的可变参数）
```

注意这是**原地挪动**——`i` 之后的元素集体左移一格，原 slice 的内容被改写了。另外用之前必须验下标：

```go
if i < 0 || i >= len(s) {
    return s
}
```

不在乎顺序的话有更快的写法——拿最后一个元素补洞：

```go
func deleteAtUnordered(s []int, i int) []int {
    s[i] = s[len(s)-1]
    return s[:len(s)-1]
}
```

```text
[10 40 30]
```

只动一个元素，代价 O(1)，但顺序变了（40 跑到了中间）。

---

## 7. 原地过滤

要"只保留满足条件的元素"，可以复用原底层数组，零新分配：

```go
func positive(nums []int) []int {
    result := nums[:0]           // len=0 但 cap 保留：同一块玻璃上的空窗口
    for _, n := range nums {
        if n > 0 {
            result = append(result, n)
        }
    }
    return result
}
```

`nums[:0]` 的窗口长度为 0、容量拉满，后续 append 全都写回原底层数组——读的位置永远不落后于写的位置，所以不会自己踩自己。

### 🕳️ 坑：原地过滤之后，原 slice 报废

```go
raw := []int{1, -2, 3, -4, 5}
got := positive(raw)

fmt.Println(got)
fmt.Println(raw)
```

```text
[1 3 5]
[1 3 5 -4 5]
```

`raw` 变成了一堆压缩后的数据加残渣。**用了原地过滤，就默认原 slice 已交出所有权，之后只碰返回值。**

---

## 8. 克隆 slice

要一份彻底独立的数据（JS `slice()` 的那种复印件），显式复制：

```go
func clone(nums []int) []int {
    out := make([]int, len(nums))
    copy(out, nums)
    return out
}
```

也有 append 一行流：

```go
out := append([]int(nil), nums...)
```

验证独立性：

```go
orig := []int{1, 2, 3}
cp := clone(orig)
cp[0] = 100
fmt.Println(orig, cp)
```

```text
[1 2 3] [100 2 3]
```

---

## 9. 避免无意保留大数组

从大 slice 切一小片，小片会**拽住整个底层数组**不放：

```go
func firstTen(data []byte) []byte {
    return data[:10]   // 只要10字节，却让整个 data 的底层数组不能被回收
}
```

如果 `data` 是个 100MB 的文件内容，而返回的 10 字节被长期持有，那 100MB 就一直赖在内存里（GC 只看"有没有人引用底层数组"，不看你用了几格）。解法还是复制：

```go
func firstTen(data []byte) []byte {
    out := make([]byte, 10)
    copy(out, data[:10])
    return out
}
```

处理大文件、大响应、大缓存时，"切小片长期保存"前先想想要不要 copy。

---

## 10. append 后不要继续依赖旧 slice

```go
s := []int{1, 2, 3}
t := append(s, 4)
```

`t` 和 `s` 现在共享吗？**不知道**——取决于 append 那一刻 cap 够不够（第 2、3 节的两种结局）。基于"它们共享"或"它们独立"写的逻辑都是赌博。清晰的写法只有两种：

```go
s = append(s, 4)                     // 覆盖自己，旧头作废
t := append(clone(s), 4)             // 真要留原数据，先克隆再 append
```

**口诀：append 完，旧变量当它已死。**

---

## 本篇重点

- [ ] 切片是共享视图：多个 slice 一块底层数组，改元素互相可见——JS 的"数组各自独立"直觉在这里失效。
- [ ] append 看 cap 行事：容量够 → 写进共享数组（可能踩到别人）；不够 → 搬新家（共享断开）。
- [ ] `s[low:high:max]` 掐断 cap，强迫 append 搬家，保护原数据。
- [ ] 已知规模就 `make([]T, 0, n)` 预分配，省掉反复扩容复制；克隆用 `copy` 或 `append([]T(nil), s...)`。
- [ ] 原地过滤和保序删除都会改写原 slice；从大数据上切小片长期保存要先 copy，否则整个大数组回收不掉。

---

## 练习

1. 写一段代码验证 `append` 容量足够时会影响原底层数组。
2. 使用完整切片表达式 `s[low:high:max]`，让 append 不影响原 slice。
3. 实现 `DeleteAt(s []int, i int) ([]int, bool)`，下标非法时返回 `false`。
4. 实现 `CloneStrings(s []string) []string`，确认修改克隆结果不会影响原数据。
5. 实现 `CompactPositive(nums []int) []int`，原地过滤出所有正数。

提示：第 1、2 题先手算 len/cap 再预测输出，跑完对答案；第 3 题非法下标包括负数；第 5 题参考第 7 节，写完顺手打印一下原 slice，看看"报废现场"长什么样。
