# 02: slice 基础：slice 头、len 与 cap

slice 是 Go 中最常用的序列结构。它可以动态增长，但 slice 变量本身不是数组，它描述的是底层数组中的一段连续区域。

---

## 1. slice 的概念模型

可以把 slice 理解成一个很小的描述符：

```go
type sliceHeader struct {
    data *T
    len  int
    cap  int
}
```

这只是帮助理解的模型，不建议在业务代码里直接使用 `reflect.SliceHeader` 或 `unsafe` 操作 slice 内部结构。

三个关键点：

- `data`: 指向底层数组中的某个元素。
- `len`: 当前 slice 能访问多少个元素。
- `cap`: 从 `data` 开始到底层数组末尾最多还能容纳多少个元素。

---

## 2. 声明 slice

```go
var s []int

fmt.Println(s == nil) // true
fmt.Println(len(s))   // 0
fmt.Println(cap(s))   // 0
```

未初始化的 slice 是 nil slice。nil slice 可以 `len`、`cap`、`range`、`append`。

```go
var s []int
s = append(s, 1, 2, 3)
fmt.Println(s) // [1 2 3]
```

---

## 3. 空 slice 与 nil slice

```go
var a []int
b := []int{}
c := make([]int, 0)

fmt.Println(a == nil) // true
fmt.Println(b == nil) // false
fmt.Println(c == nil) // false
```

它们的长度都是 0：

```go
fmt.Println(len(a), len(b), len(c)) // 0 0 0
```

多数业务逻辑里，nil slice 和空 slice 可以一样使用。需要序列化成 JSON、对外返回 API 数据时，二者可能表现不同：nil slice 通常编码为 `null`，空 slice 通常编码为 `[]`。

---

## 4. 使用字面量创建 slice

```go
nums := []int{1, 2, 3}
names := []string{"alice", "bob"}
```

注意数组和 slice 字面量的区别：

```go
a := [3]int{1, 2, 3} // 数组
s := []int{1, 2, 3}  // slice
```

`[3]int` 的长度写在类型里；`[]int` 没有固定长度。

---

## 5. 使用 make 创建 slice

```go
s := make([]int, 3)

fmt.Println(s)      // [0 0 0]
fmt.Println(len(s)) // 3
fmt.Println(cap(s)) // 3
```

指定长度和容量：

```go
s := make([]int, 3, 10)

fmt.Println(len(s)) // 3
fmt.Println(cap(s)) // 10
```

这里 `len=3` 的位置已经存在，可以直接读写：

```go
s[0] = 100
```

`cap=10` 中多出来的容量不能直接用下标访问，只能通过 `append` 扩展长度后访问。

---

## 6. 切片表达式

```go
nums := []int{10, 20, 30, 40, 50}

a := nums[1:4]
fmt.Println(a) // [20 30 40]
```

语法是左闭右开：

```go
s[low:high]
```

常见写法：

```go
nums[:3] // 从开头到下标 3 之前
nums[2:] // 从下标 2 到末尾
nums[:]  // 整个 slice
```

---

## 7. 切片后的 len 与 cap

```go
nums := []int{10, 20, 30, 40, 50}
a := nums[1:3]

fmt.Println(a)      // [20 30]
fmt.Println(len(a)) // 2
fmt.Println(cap(a)) // 4
```

`a` 从 `nums[1]` 开始，长度到 `nums[3]` 之前，所以 `len=2`。容量从 `nums[1]` 一直算到底层数组末尾，所以 `cap=4`。

---

## 8. 完整切片表达式

完整切片表达式可以限制新 slice 的容量：

```go
nums := []int{10, 20, 30, 40, 50}
a := nums[1:3:3]

fmt.Println(a)      // [20 30]
fmt.Println(len(a)) // 2
fmt.Println(cap(a)) // 2
```

语法：

```go
s[low:high:max]
```

新 slice 的容量是 `max - low`。这个写法常用于限制后续 `append` 对原底层数组的影响。

---

## 9. append 必须接收返回值

`append` 返回新的 slice。

```go
s := []int{1, 2}
s = append(s, 3)
```

不要这样写：

```go
// append(s, 3) // 编译错误：append 的结果没有使用
```

`append` 可能复用原底层数组，也可能分配新底层数组。调用后应该使用返回的 slice。

---

## 10. copy

`copy` 可以把一个 slice 的元素复制到另一个 slice。

```go
src := []int{1, 2, 3}
dst := make([]int, len(src))

n := copy(dst, src)

fmt.Println(n)   // 3
fmt.Println(dst) // [1 2 3]
```

返回值是实际复制的元素数量，等于两个 slice 长度中的较小值。

---

## 11. slice 不能直接比较

slice 只能和 `nil` 比较。

```go
var s []int
fmt.Println(s == nil)
```

不能这样比较：

```go
// a := []int{1, 2}
// b := []int{1, 2}
// fmt.Println(a == b) // 编译错误
```

如果要比较元素内容，可以自己循环，也可以在合适章节学习标准库里的 `slices.Equal`。

---

## 练习

1. 创建 `[]int{10, 20, 30, 40, 50}`，分别打印 `s[1:4]`、`s[:2]`、`s[2:]` 的 `len` 和 `cap`。
2. 用 `make([]string, 0, 5)` 创建 slice，连续 append 3 个字符串，观察 `len` 和 `cap`。
3. 创建 nil slice 和空 slice，分别打印它们的 `len`、`cap` 和 `s == nil`。
4. 用 `copy` 克隆一个 `[]int`，修改克隆后的 slice，确认原 slice 不变。
