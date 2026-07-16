# 02: slice 基础：slice 头、len 与 cap

这篇认识 Go 里最常用的序列结构 slice。你在 JS 里天天用的 `[1, 2, 3]`，在 Go 里对应的就是它——但 slice 不是"装元素的箱子"，而是**趴在一个底层数组上的窗口**。看懂"窗口"这个模型，下一篇那些"改一个 slice 另一个也变了"的灵异现象就全都能解释。

---

## 1. slice 的概念模型

一个 slice 变量本身非常小，就三个字段（slice 头，slice header）：

```go
type sliceHeader struct {
    data *T  // 指向底层数组中的某个元素
    len  int // 这扇窗口能看到几个元素
    cap  int // 从窗口起点到底层数组末尾，总共有几个格子
}
```

画出来是这样：

```text
slice 头            底层数组
┌──────┐
│ data ────────▶ ┌────┬────┬────┬────┬────┐
│ len=3│         │ 10 │ 20 │ 30 │ ?? │ ?? │
│ cap=5│         └────┴────┴────┴────┴────┘
└──────┘         └── len 看得到 ──┘└─ 备用 ─┘
```

两个立刻有用的推论：

- **slice 头本身是个普通的小值**。复制 slice 头很便宜（三个字段），但复制出来的两个头指向**同一个**底层数组——共享由此而来（下一篇的主题）。
- 很多资料管 slice 叫"引用类型"，这个词凑合能用但容易带偏：slice 头就是个值，只是它内部**含一个指针**。至于底层数组住在栈还是堆，那是另一个维度的问题，第 08 篇会正面拆开。

（这个 `sliceHeader` 只是帮助理解的模型，不要在业务代码里用 `unsafe` 去掏 slice 的内部结构。）

### slice 头 = 一个"含指针字段的 struct"而已

slice 头的真相特别朴素——它跟下面这个 `Box` struct 在机制上**完全一样**，没多任何神秘：

```go
type Box struct {
    name string
    ref  *int
}

n := 42
a := Box{name: "a", ref: &n}    // Box 在 Go 里就是个普通 struct
b := a                          // 复制整个 Box —— 现在 b 也是独立 struct

fmt.Println(a.ref == b.ref)     // true —— 两个 ref 字段值一样,都指向 n
*b.ref = 99
fmt.Println(n)                  // 99 —— 通过 b.ref 改的就是 n
```

`Box` 在 Go 里是值类型,但 `a`、`b` 因为字段 `ref` 都指向 `n`,能间接影响同一份底层数据。**slice 头就是 `Box` 把字段从 `name/ref` 换成 `data/len/cap`**——结构体、值类型、复制独立、靠指针字段共享底层数据,这些性质一模一样。所谓"slice 是引用类型"的来源,就是这一个指针字段。**认识到这一点,下面所有"灵异现象"都不灵异了**。

### 把上面那点用代码按两个时刻跑一遍

看完下面这两段,你脑子里 slice 的内部模型就成立了:

```go
var a []int               // a 这个变量就是 [data=nil,  len=0, cap=0] 三件套
d := a                    // 复制 a 的三件套给 d —— d 是独立 struct
d = append(d, 7, 8)       // 这一步里 a 完全没被碰过
```

```text
时刻 1：d := a 复制完
─────────────────────────────────────────────────────────────
a:  [data = nil,   len = 0, cap = 0]   ← 一个栈槽
d:  [data = nil,   len = 0, cap = 0]   ← 另一个独立的栈槽

→ 两份独立 struct,内容这一刻相同
   (data 字段值都是同一个地址 —— 也就是 nil)


时刻 2：d = append(d, 7, 8)
─────────────────────────────────────────────────────────────
append 看到 d.cap = 0,装不下 → 必须分配新数组 NewArr
返回新头 (data = &NewArr, len = 2, cap = 2),赋值给 d
a 完全没碰过

a:  [data = nil,      len = 0, cap = 0]   ← 仍是 nil
d:  [data = &NewArr,  len = 2, cap = 2]   ← 指向新数组

→ a == nil 还是 true,d = [7 8]
   两份 struct 的 data 字段值不再相同
```

注意全程没有任何"对象身份"的戏码。**两个变量各自独立的 struct,data 字段值碰巧相同 → 共享底层;append 改了 d 的 data 字段值 → 各管各的**。

### Go 跟 JS 在"值类型/引用类型"上的差异

如果你从 JS 转过来,这一点要主动"洗掉"——因为 JS 的分类直接影响了你对 `array` 的默认直觉:

|                      | JS                                  | Go                                               |
| -------------------- | ----------------------------------- | ------------------------------------------------ |
| 值类型               | number, string, boolean, null, ...  | int, float, bool, string, struct, **数组**, slice 头 |
| 引用类型             | object, **array**, function         | slice, map, channel, pointer, interface          |
| 数组归类             | **引用类型**                        | **值类型**                                       |

JS 里数组是引用类型,你脑子的默认模式是"数组 = 共享同一实体"。Go 里数组 `[3]int` 是值类型(复制独立),slice 才是"值类型 + 含指针字段"的复合品。第一次接触 Go 的 slice **普遍会被这个组合绕进去**,这不是你的问题,是语言的心智负担点。

**替代 JS 思维的口诀**:忘掉"变量 → 对象"的指向模型。Go 里 slice 变量就是栈上一个装着三个字段的小 struct;共享 = 两份 struct 里 `data` 字段值碰巧是同一个地址。

---

## 2. 声明 slice

```go
var s []int

fmt.Println(s == nil)
fmt.Println(len(s))
fmt.Println(cap(s))
```

```text
true
0
0
```

只声明不初始化，得到的是 **nil slice**——一个 `data` 指针为空的 slice 头。注意 `[]int` 方括号里是空的，这就是它和数组 `[3]int` 在写法上的唯一区别：**长度写在类型里的是数组，不写的是 slice。**

nil slice 出乎意料地好用：`len`、`cap`、`range`、`append` 全都能直接上：

```go
var s []int
s = append(s, 1, 2, 3)
fmt.Println(s)
```

```text
[1 2 3]
```

对比 JS：`let s;` 之后 `s.push(1)` 会炸（`undefined` 没有方法）；Go 的 nil slice 被设计成"可以直接开始 append 的空列表"。所以 Go 代码里经常看到 `var result []int` 开头、循环里直接 append 的写法。

---

## 3. 空 slice 与 nil slice

```go
var a []int          // nil slice
b := []int{}         // 空 slice
c := make([]int, 0)  // 空 slice

fmt.Println(a == nil)
fmt.Println(b == nil)
fmt.Println(c == nil)
fmt.Println(len(a), len(b), len(c))
```

```text
true
false
false
0 0 0
```

三个的长度都是 0，日常逻辑里几乎可以混用。差别在对外输出的时候暴露——序列化成 JSON 时：

```go
ja, _ := json.Marshal(a)
jb, _ := json.Marshal(b)

fmt.Println(string(ja))
fmt.Println(string(jb))
```

```text
null
[]
```

**一句话总结：内部逻辑不区分 nil 和空 slice，写 API 返回值时要区分——前端拿到 `null` 还是 `[]` 可不是一回事。**

---

## 4. 使用字面量创建 slice

```go
nums := []int{1, 2, 3}
names := []string{"alice", "bob"}
```

再对照一次数组和 slice 的字面量，差别只在方括号：

```go
a := [3]int{1, 2, 3} // 数组：长度在类型里
s := []int{1, 2, 3}  // slice：不写长度
```

背后的动作不一样：slice 字面量会先建一个底层数组，再造一个趴在上面的 slice 头给你。

---

## 5. 使用 make 创建 slice

`make` 用来创建"指定长度/容量"的 slice：

```go
s := make([]int, 3)

fmt.Println(s)
fmt.Println(len(s), cap(s))
```

```text
[0 0 0]
3 3
```

第三个参数单独指定容量：

```go
s := make([]int, 3, 10)
fmt.Println(len(s), cap(s))
```

```text
3 10
```

拆开这行：

```text
make([]int, 3, 10)
//    ─┬──  ┬  ─┬
//   类型  len  cap：底层数组给10格，先亮出3格
```

### 🕳️ 坑：cap 里多出来的格子不能直接用下标访问

以为会怎样：都分配了 10 格，`s[3]` 应该能写。
实际怎样：panic。
为什么：下标访问的合法范围由 `len` 决定，不是 `cap`。备用格子只能靠 `append` 把 `len` 推过去之后才可见。

```go
package main

import "fmt"

func main() {
	s := make([]int, 3, 10)
	s[3] = 1
	fmt.Println(s)
}
```

```text
panic: runtime error: index out of range [3] with length 3

goroutine 1 [running]:
main.main()
	./main.go:7 +0x38
exit status 2
```

注意 panic 消息说的是 `with length 3`——它只关心 len。

---

## 6. 切片表达式

从现有 slice（或数组）上再开一扇窗口：

```go
nums := []int{10, 20, 30, 40, 50}

a := nums[1:4]
fmt.Println(a)
```

```text
[20 30 40]
```

> 💡 `a` 本身就是 slice，类型是 `[]int`，跟 `var s []int` 或 `[]int{1, 2, 3}` 是同一种类型。区别只在数据怎么来——`arr[1:4]` 是"开窗"，字面量是"直接造数据"。`fmt.Printf("%T", a)` 跟任何 slice 一样会打出 `[]int`。

语法和 JS 的 `slice(1, 4)` 一样是**左闭右开**：

```text
nums[1:4]
//   ┬ ┬
//   │ └ 到下标4之前（不含4）
//   └ 从下标1开始（含1）
```

省略写法也和 JS 类似：

```go
nums[:3] // 从开头到下标 3 之前
nums[2:] // 从下标 2 到末尾
nums[:]  // 整个 slice
```

但有个致命差异先剧透：JS 的 `arr.slice()` 返回**复制出来的新数组**；Go 的 `nums[1:4]` 只是**在同一个底层数组上开了扇新窗口**，不复制任何元素。证据在下一节和下一篇。

---

## 7. 切片后的 len 与 cap

```go
nums := []int{10, 20, 30, 40, 50}
a := nums[1:3]

fmt.Println(a)
fmt.Println(len(a), cap(a))
```

```text
[20 30]
2 4
```

`len=2` 好理解（下标 1 到 3 之前，两个元素）。`cap` 为什么是 4？因为容量从窗口起点**一直数到底层数组末尾**：

```text
底层数组   10   20   30   40   50
                └── a 的窗口 len=2
                └────── a 的 cap=4 ──────┘
```

窗口右边那两格（40、50）虽然 `a` 现在看不到，但它们是 `a` 的"备用地盘"——`append` 时会先往那里写。这正是下一篇共享事故的案发现场，先把这张图记住。

---

## 8. 完整切片表达式

多写一个数，可以把新窗口的**容量也掐断**：

```go
nums := []int{10, 20, 30, 40, 50}
a := nums[1:3:3]

fmt.Println(a)
fmt.Println(len(a), cap(a))
```

```text
[20 30]
2 2
```

语法：

```text
s[low : high : max]
//  ┬     ┬     ┬
//  起点  len终点  cap终点：容量 = max - low
```

对比第 7 节：同样是 `[1:3]`，加上 `:3` 之后 cap 从 4 变成 2——备用地盘被没收了。这个写法专门用来**限制后续 append 波及原底层数组**，下一篇会用实验演示它怎么救命。

---

## 9. append 必须接收返回值

`append` 返回追加后的新 slice 头，必须接住：

```go
s := []int{1, 2}
s = append(s, 3)
```

不接不行——这在 Go 里直接是编译错误：

```go
package main

func main() {
	s := []int{1, 2}
	append(s, 3)
}
```

```text
# command-line-arguments
./main.go:5:2: append(s, 3) (value of type []int) is not used
```

### 🕳️ 坑：append 不是 JS 的 push

以为会怎样：像 `arr.push(3)` 一样原地改，不用管返回值。
实际怎样：`append` 可能复用原底层数组，也可能**搬去一个全新的数组**（容量不够时），返回的 slice 头才指向正确的位置。
为什么：旧的 slice 头可能还趴在废弃的旧数组上。所以标准姿势永远是 `s = append(s, ...)`。

---

## 10. copy

`copy(dst, src)` 把元素从一个 slice 复制到另一个：

```go
src := []int{1, 2, 3}
dst := make([]int, len(src))

n := copy(dst, src)

fmt.Println(n)
fmt.Println(dst)
```

```text
3
[1 2 3]
```

返回值是实际复制的元素个数（两边长度取小）。复制完就是两份独立数据了：

```go
dst[0] = 100
fmt.Println(src, dst)
```

```text
[1 2 3] [100 2 3]
```

---

## 11. slice 不能直接比较

slice 只能和 `nil` 比，两个 slice 之间用 `==` 是编译错误：

```go
package main

import "fmt"

func main() {
	a := []int{1, 2}
	b := []int{1, 2}
	fmt.Println(a == b)
}
```

```text
# command-line-arguments
./main.go:8:14: invalid operation: a == b (slice can only be compared to nil)
```

对比 JS：`[1,2] === [1,2]` 是 `false`（比引用），也够迷惑的；Go 干脆不让比。要比内容，自己循环，或用标准库的 `slices.Equal`（标准库模块再展开）。

---

## 本篇重点

- [ ] slice 变量 = 一个三字段的小值（data 指针 + len + cap），趴在底层数组上的**窗口**，不是箱子。
- [ ] nil slice 可以直接 `len`/`range`/`append`；对外输出 JSON 时 nil 是 `null`、空 slice 是 `[]`。
- [ ] `len` 管下标访问的合法范围，`cap` 是到底层数组末尾的总格数；`s[low:high:max]` 能掐断 cap。
- [ ] 切片表达式不复制元素（和 JS 的 `slice()` 相反），新旧窗口共享同一个底层数组。
- [ ] `append` 必须写成 `s = append(s, x)`——它可能搬家，不是 JS 的 `push`。

---

## 练习

1. 创建 `[]int{10, 20, 30, 40, 50}`，分别打印 `s[1:4]`、`s[:2]`、`s[2:]` 的 `len` 和 `cap`。
2. 用 `make([]string, 0, 5)` 创建 slice，连续 append 3 个字符串，观察 `len` 和 `cap`。
3. 创建 nil slice 和空 slice，分别打印它们的 `len`、`cap` 和 `s == nil`。
4. 用 `copy` 克隆一个 `[]int`，修改克隆后的 slice，确认原 slice 不变。

提示：第 1 题先照第 7 节的图自己算一遍 cap 再运行验证；第 2 题注意 cap 会不会变——想想为什么。
