# 08: for 循环与 range

Go 只有 `for`，没有 `while`。所有循环都用 `for` 表达。

---

## 1. 标准 for 循环

```go
func main() {
    for i := 0; i < 5; i++ {
        fmt.Println(i)
    }
}
```

三个部分分别是：

```text
初始化语句; 循环条件; 后置语句
```

---

## 2. 类似 while 的写法

```go
func main() {
    n := 3
    for n > 0 {
        fmt.Println(n)
        n--
    }
}
```

只保留条件，就相当于其他语言里的 `while`。

---

## 3. 无限循环

```go
func main() {
    for {
        fmt.Println("run once")
        break
    }
}
```

无限循环必须有明确退出路径，比如 `break`、`return`、程序退出或外部取消信号。

---

## 4. range 遍历 slice

```go
func main() {
    nums := []int{10, 20, 30}

    for i, v := range nums {
        fmt.Println(i, v)
    }
}
```

不需要索引时，用 `_`：

```go
for _, v := range nums {
    fmt.Println(v)
}
```

只需要索引时：

```go
for i := range nums {
    fmt.Println(i)
}
```

---

## 5. range 的值是副本

`range` 遍历 slice 时，第二个变量是值的副本。

```go
func main() {
    nums := []int{1, 2, 3}

    for _, v := range nums {
        v *= 10
    }

    fmt.Println(nums) // [1 2 3]
}
```

要修改原 slice，使用索引：

```go
func main() {
    nums := []int{1, 2, 3}

    for i := range nums {
        nums[i] *= 10
    }

    fmt.Println(nums) // [10 20 30]
}
```

---

## 6. range 遍历 string

遍历字符串时，`range` 按 rune 遍历，索引是字节位置。

```go
func main() {
    s := "Go语言"

    for i, r := range s {
        fmt.Println(i, string(r))
    }
}
```

---

## 7. range 遍历 map

map 遍历顺序不保证稳定。

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

不要依赖 map 的遍历顺序。需要稳定顺序时，先取出 key 排序。

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
