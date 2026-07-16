# 09: range、复制与内存语义陷阱

这篇解决一类高频 bug："循环里明明改了，怎么一个都没改上？"根源只有一个：**range 的循环变量是元素的副本**——相当于 JS 的 `for...of` 给你的每个元素都先过了一遍复印机。把"副本从哪来"看清楚，这类 bug 一眼就能识破。

---

## 1. range 基本形式

```go
nums := []int{10, 20, 30}

for i, v := range nums {
    fmt.Println(i, v)
}
```

```text
0 10
1 20
2 30
```

对 slice：`i` 是下标，`v` 是元素值的**副本**。只要其中一个的写法：

```go
for i := range nums { ... }    // 只要下标
for _, v := range nums { ... } // 只要值
```

对比 JS：`for (const [i, v] of nums.entries())`。差别马上揭晓——JS 的 `v` 对对象元素是引用，Go 的 `v` 永远是复印件。

---

## 2. 修改 range 变量不会修改元素

对照实验：

```go
nums := []int{1, 2, 3}

for _, v := range nums {
    v *= 10          // 改的是复印件
}
fmt.Println(nums)

for i := range nums {
    nums[i] *= 10    // 通过下标改本体
}
fmt.Println(nums)
```

```text
[1 2 3]
[10 20 30]
```

**口诀：改 `v` 改了个寂寞，要改就 `nums[i]`。** `v` 只是每轮迭代从数组格子里复制出来的临时变量，改完就扔。

---

## 3. struct 元素也会被复制

int 复制便宜没感觉，struct 元素被整个复印时坑最深：

```go
type User struct {
    Name string
    Age  int
}

users := []User{
    {Name: "alice", Age: 18},
    {Name: "bob", Age: 20},
}

for _, u := range users {
    u.Age++          // 给复印件过生日
}
fmt.Println(users[0].Age)

for i := range users {
    users[i].Age++   // 给本人过生日
}
fmt.Println(users[0].Age)
```

```text
18
19
```

JS 里 `for (const u of users) u.age++` 是能改到的（数组存的是对象引用）——这条直觉在 Go 的 `[]User` 上必翻车。如果 slice 存的是指针 `[]*User`，那 `u` 复制的是指针、指向同一对象，`u.Age++` 就生效了——又回到第 08 篇那张"复制了什么"的表。

---

## 4. 取 range 变量地址

### 🕳️ 坑：`&u` 拿到的是复印件的地址

以为会怎样：收集 `&u` 就等于收集了各元素的地址，之后能通过指针改 slice。
实际怎样：拿到的是循环变量（副本）的地址，改它影响不到 slice。
为什么：`u` 本来就是复制品，取它的地址当然也只是复制品的地址。

```go
users := []User{
    {Name: "alice", Age: 18},
    {Name: "bob", Age: 20},
}

var ptrs []*User
for _, u := range users {
    ptrs = append(ptrs, &u)
}

ptrs[0].Age = 99
fmt.Println(users[0].Age)
fmt.Println(ptrs[0].Name, ptrs[1].Name)
```

```text
18
alice bob
```

两个观察：改 `ptrs[0]` 后 `users[0]` 岿然不动（指针指向副本）；`ptrs` 里两个指针倒是各指各的（Go 1.22 起每轮迭代是新变量，更老的版本还有"所有指针指向同一个最后元素"的加倍惨案）。**不管新旧版本，`&u` 指向的都是副本**——要元素本体的地址，用下标：

```go
var ptrs2 []*User
for i := range users {
    ptrs2 = append(ptrs2, &users[i])
}

ptrs2[0].Age = 99
fmt.Println(users[0].Age)
```

```text
99
```

---

## 5. range 数组会复制整个数组

range 开始时会先对被遍历的对象求值。对象是**数组**（值！）时，复制的是整个数组——循环中改原数组，`v` 看不见：

```go
arr := [3]int{1, 2, 3}
for i, v := range arr {
    if i == 0 {
        arr[1] = 100   // 改原数组
    }
    fmt.Println(i, v)
}
fmt.Println(arr)
```

```text
0 1
1 2
2 3
[1 100 3]
```

`arr[1]` 确实被改成了 100（最后一行），但循环里的 `v` 打出来还是 2——`v` 读的是 range 开场时复制的那份快照。换成 slice 再做一遍：

```go
sl := []int{1, 2, 3}
for i, v := range sl {
    if i == 0 {
        sl[1] = 100
    }
    fmt.Println(i, v)
}
```

```text
0 1
1 100
2 3
```

这次 `v` 看见 100 了——range 复制的只是 slice 头（三个字段），底层数组还是同一个。**同一段循环，数组和 slice 两种结果，再次验证"复制了什么"决定一切。** 大数组不想被整个复制，用 `range &arr`（复制指针）或直接 `range arr[:]`（转成 slice）。

---

## 6. range map

map 遍历时，key 和 value 同样被复制进循环变量，顺序照旧不稳定（第 04 篇）。value 是 struct 时，老坑新形态：

```go
usersByID := map[int]User{
    1: {Name: "alice", Age: 18},
}

for _, u := range usersByID {
    u.Age++              // 改副本，无效
}
fmt.Println(usersByID[1].Age)
```

```text
18
```

而且 map 没有"下标改本体"这条路（第 05 篇：map 元素不可取地址），只能取出、修改、放回：

```go
for id, u := range usersByID {
    u.Age++
    usersByID[id] = u    // 放回去才算数
}
fmt.Println(usersByID[1].Age)
```

```text
19
```

---

## 7. range string

对 string 用 range，拿到的是**字节下标 + rune**（02 模块字符串篇的老朋友，这里放进 range 全家福）：

```go
s := "Go语言"

for i, r := range s {
    fmt.Printf("%d %c\n", i, r)
}
```

```text
0 G
1 o
2 语
5 言
```

下标 2 直接跳到 5——"语"占 3 个字节。而普通 for 循环按下标访问，拿到的是一个个**字节**：

```go
for i := 0; i < len(s); i++ {
    fmt.Printf("%d %x\n", i, s[i])
}
```

```text
0 47
1 6f
2 e8
3 af
4 ad
5 e8
6 a8
7 80
```

**一句话总结：range string 按字符走（给 rune），下标访问按字节走（给 byte）。**

---

## 8. range nil slice 和 nil map

```go
var s []int
for _, v := range s {
    fmt.Println(v)
}

var m map[string]int
for k, v := range m {
    fmt.Println(k, v)
}

fmt.Println("done, no panic")
```

```text
done, no panic
```

nil 上 range 就是执行 0 次，不 panic。所以"遍历前判空"在 Go 里通常是多余的——放心直接 range。

---

## 9. range 时 append

边遍历边 append 会不会死循环？不会：

```go
nums := []int{1, 2, 3}

for _, v := range nums {
    nums = append(nums, v*10)
}

fmt.Println(nums)
```

```text
[1 2 3 10 20 30]
```

原因和第 5 节同源：range 开场时把 slice 头快照了（len=3），循环只走这 3 轮；append 出来的新元素本轮 range 看不见。合法，但读代码的人容易懵，更清晰的写法是产出到另一个结果 slice：

```go
out := make([]int, 0, len(nums)*2)
for _, v := range nums {
    out = append(out, v)
    out = append(out, v*10)
}
```

---

## 10. 判断副本的通用方法

看到 range 就默念四连问：

1. range 的对象是什么——数组（整个复制）、slice（复制头）、map（复制句柄）、string（按 rune 解码）还是 channel（并发模块见）。
2. 循环变量是不是元素副本。（答案永远是：是。）
3. 改循环变量能不能影响原数据——取决于副本里有没有"线索"（指针/slice 头/map 句柄）。
4. 取地址取到的是谁的地址——循环变量的，不是元素的。

四个问题全部能用第 08 篇那张"复制了什么"的表回答。**range 没有新规则，它只是把赋值复制的规则每轮迭代执行一遍。**

---

## 本篇重点

- [ ] range 的循环变量是副本：改 `v` 无效，改 `nums[i]` 才有效——JS "for...of 拿到对象引用"的直觉在 `[]User` 上失效。
- [ ] `&u` 是副本的地址（任何 Go 版本都是），要元素地址用 `&users[i]`。
- [ ] range 数组复制整个数组，range slice 只复制头——循环中改原数据，前者看不见、后者看得见。
- [ ] map 的 value 副本改完必须放回（`m[k] = u`）；range string 给字节下标 + rune，下标会跳。
- [ ] nil slice/map 可以放心 range（0 次）；range 开场快照 slice 头，循环内 append 的新元素本轮不可见。

---

## 练习

1. 写代码验证修改 `range` 的值变量不会修改 slice 元素。
2. 写代码验证使用下标可以修改 slice 元素。
3. 对 `[]User` range，分别尝试修改值变量和 `users[i]`。
4. 写出错误的 `&u` 版本，再改成 `&users[i]`。
5. 对 `"Go语言"` range，打印字节下标和 rune。

提示：第 4 题除了看能不能改到原数据，再打印一下收集到的指针指向的 Name，确认它们各是谁；第 5 题先预测"言"的下标是几再运行。
