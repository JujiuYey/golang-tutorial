# 08: for 循环与 range

这篇解决的问题：JS 里 `for`、`while`、`do-while`、`for-of`、`for-in`、`forEach` 一大家子，到了 Go 全部归并成**一个关键字 `for`**。学完你能用 for 写出所有循环形态，并避开 range 的头号坑——循环变量是副本。

---

## 1. 标准 for 循环

和 JS 的 for 几乎一样，只是不写圆括号（花括号必须有，和 if 同款规则）：

```go
func main() {
    for i := 0; i < 5; i++ {
        fmt.Println(i)
    }
}
```

```text
0
1
2
3
4
```

三段结构：

```text
for i := 0; i < 5; i++ {
//  ──┬───  ──┬──  ─┬─
//  初始化   条件   后置语句（每轮结束执行）
```

`i` 只活在循环里——上一篇 if 初始化语句的作用域规则，本来就是从这里搬过去的。

---

## 2. 类似 while 的写法

Go 没有 `while` 关键字。把 for 的三段砍掉两段、只留条件，就是 while：

```go
func main() {
    n := 3
    for n > 0 {
        fmt.Println(n)
        n--
    }
}
```

```text
3
2
1
```

记忆锚点：**JS 的 `while (cond)` = Go 的 `for cond`**，换个关键字而已。

---

## 3. 无限循环

条件也砍掉，就是无限循环（等于 JS 的 `while (true)`）：

```go
func main() {
    for {
        fmt.Println("run once")
        break
    }
}
```

```text
run once
```

无限循环必须有明确的退出路径：`break`、`return`，或者后面并发篇会讲的外部取消信号。

**一句话总结：三段是 for，一段是 while，零段是 while(true)——全都叫 `for`。**

---

## 4. range 遍历 slice

`range` 对应 JS 的 `for...of`，但一次给你两个值：索引和元素（相当于自带 `entries()`）：

```go
func main() {
    nums := []int{10, 20, 30}

    for i, v := range nums {
        fmt.Println(i, v)
    }
}
```

```text
0 10
1 20
2 30
```

不需要索引时，用 `_` 扔掉（还记得吗：Go 不允许声明了不用的变量）：

```go
for _, v := range nums {
    fmt.Println(v)
}
```

只需要索引时，直接少写一个变量：

```go
for i := range nums {
    fmt.Println(i)
}
```

---

## 5. range 的值是副本

### 🕳️ 坑：改循环变量，改不到原 slice

以为会怎样：循环里 `v *= 10`，slice 变成 `[10 20 30]`。
实际怎样：slice 纹丝不动。
为什么：每轮迭代，range 把元素的值**复制**给 `v`——`v` 是个独立副本，改它等于改一张复印件。

```go
func main() {
    nums := []int{1, 2, 3}

    for _, v := range nums {
        v *= 10
    }

    fmt.Println(nums)
}
```

```text
[1 2 3]
```

想修改原 slice，通过**索引**直击本体：

```go
func main() {
    nums := []int{1, 2, 3}

    for i := range nums {
        nums[i] *= 10
    }

    fmt.Println(nums)
}
```

```text
[10 20 30]
```

JS 里其实有同一个坑：`arr.forEach(v => v *= 10)` 同样改不动原数组。**一句话总结：读元素用 `v`，改元素用 `nums[i]`。**

---

## 6. range 遍历 string

02-05 的知识在这里派上用场：range 遍历字符串是**按 rune（字符）解码**的，索引是字节位置：

```go
func main() {
    s := "Go语言"

    for i, r := range s {
        fmt.Println(i, string(r))
    }
}
```

```text
0 G
1 o
2 语
5 言
```

索引 2 直接跳到 5——`语` 占 3 个字节。忘了为什么的话回看 02-05 第 4 节。

---

## 7. range 遍历 map

range 也能遍历 map，一次给出 key 和 value：

```go
func main() {
    scores := map[string]int{
        "Alice": 90,
        "Bob":   80,
    }

    for name, score := range scores {
        fmt.Println(name, score)
    }
}
```

某一次运行的输出：

```text
Alice 90
Bob 80
```

### 🕳️ 坑：map 的遍历顺序是故意随机的

以为会怎样：像 JS 对象一样按插入顺序出来。
实际怎样：同一段代码多跑几次，顺序会变（上面的例子实测跑三次，第三次就变成了 `Bob 80` 在前）。
为什么：Go 运行时**故意**随机化 map 遍历起点，就是为了防止你写出依赖顺序的代码。

需要稳定顺序时，先把 key 取出来排序，再按排序后的 key 访问。JS 转来的同学特别注意：JS 对象的"字符串键按插入序"是 Go map 没有的保证。

---

## 本篇重点

- [ ] Go 只有 `for`：三段式是经典 for，只留条件是 while，全空是无限循环。
- [ ] `for i, v := range xs` = JS 的 `for...of` + `entries()`；不要的变量用 `_` 扔掉。
- [ ] range 的 `v` 是副本，改它无效——修改原 slice 必须写 `nums[i]`。
- [ ] range 字符串按 rune 解码，索引是字节位置（会跳格）。
- [ ] map 遍历顺序故意随机，永远不要依赖它；要顺序就先排 key。

---

## 练习

写一个函数：

```go
func sum(nums []int) int
```

要求：

1. 使用 `for range`。
2. 空 slice 返回 `0`。
3. 再写一个函数把 slice 中每个元素乘以 `10`，要求修改原 slice。

提示：第 2 点不需要特判——想想累加变量的零值是多少。第 3 点回看第 5 节的坑。
