# 09: range、复制与内存语义陷阱

`range` 写起来很顺手，但它会引入副本。理解副本从哪里来，是读懂 Go 代码的重要能力。

---

## 1. range 基本形式

```go
nums := []int{10, 20, 30}

for i, v := range nums {
    fmt.Println(i, v)
}
```

对 slice 来说，`i` 是下标，`v` 是元素值的副本。

如果只需要下标：

```go
for i := range nums {
    fmt.Println(i)
}
```

如果只需要值：

```go
for _, v := range nums {
    fmt.Println(v)
}
```

---

## 2. 修改 range 变量不会修改元素

```go
nums := []int{1, 2, 3}

for _, v := range nums {
    v *= 10
}

fmt.Println(nums) // [1 2 3]
```

`v` 是元素副本。修改 `v` 不会修改 slice 中的元素。

要修改元素，使用下标：

```go
for i := range nums {
    nums[i] *= 10
}

fmt.Println(nums) // [10 20 30]
```

---

## 3. struct 元素也会被复制

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
    u.Age++
}

fmt.Println(users[0].Age) // 18
```

要修改原元素：

```go
for i := range users {
    users[i].Age++
}
```

---

## 4. 取 range 变量地址

```go
var ptrs []*User

for _, u := range users {
    ptrs = append(ptrs, &u)
}
```

这段代码取到的是 range 变量 `u` 的地址，不是 slice 元素的地址。即使在新版本 Go 中每次迭代变量的作用域更合理，`&u` 仍然指向副本。

如果要保存元素地址，使用下标：

```go
for i := range users {
    ptrs = append(ptrs, &users[i])
}
```

---

## 5. range 数组会复制数组

```go
arr := [3]int{1, 2, 3}

for _, v := range arr {
    fmt.Println(v)
}
```

对数组执行 `range` 时，range 表达式会被求值。数组是值，可能发生整个数组复制。

如果数组很大，可以对数组指针或 slice range：

```go
for _, v := range &arr {
    fmt.Println(v)
}
```

更常见的写法是使用 slice：

```go
s := arr[:]
for _, v := range s {
    fmt.Println(v)
}
```

---

## 6. range map

```go
scores := map[string]int{
    "alice": 100,
    "bob":   90,
}

for name, score := range scores {
    fmt.Println(name, score)
}
```

map 的 key 和 value 也会被复制到循环变量。遍历顺序不稳定。

如果 value 是 struct，修改副本不会改 map：

```go
for _, u := range usersByID {
    u.Age++
}
```

要修改 map 中的 struct 值，需要取出、修改、放回：

```go
for id, u := range usersByID {
    u.Age++
    usersByID[id] = u
}
```

---

## 7. range string

对 string range 时，得到的是字节下标和 rune。

```go
s := "Go语言"

for i, r := range s {
    fmt.Printf("%d %c\n", i, r)
}
```

`i` 是字节下标，不是第几个字符。`r` 是 Unicode code point，也就是 `rune`。

如果只用普通下标访问：

```go
for i := 0; i < len(s); i++ {
    fmt.Printf("%d %x\n", i, s[i])
}
```

这里访问的是 byte。

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
```

nil slice 和 nil map 上的 range 都会执行 0 次，不会 panic。

---

## 9. range 时 append

```go
nums := []int{1, 2, 3}

for _, v := range nums {
    nums = append(nums, v*10)
}

fmt.Println(nums) // [1 2 3 10 20 30]
```

range 开始时会先确定要遍历的 slice 头。循环中 append 出来的新元素不会继续被本轮 range 访问。

在同一个循环中边遍历边 append，容易让代码难读。更清晰的做法是写入另一个结果 slice：

```go
out := make([]int, 0, len(nums)*2)
for _, v := range nums {
    out = append(out, v)
    out = append(out, v*10)
}
```

---

## 10. 判断副本的通用方法

看到 `range` 时问四个问题：

1. range 的对象是什么：数组、slice、map、string 还是 channel。
2. 循环变量是不是元素副本。
3. 修改循环变量能不能影响原数据。
4. 如果取地址，取到的是循环变量地址，还是原元素地址。

只要能回答这四个问题，绝大多数 range 相关 bug 都能看出来。

---

## 练习

1. 写代码验证修改 `range` 的值变量不会修改 slice 元素。
2. 写代码验证使用下标可以修改 slice 元素。
3. 对 `[]User` range，分别尝试修改值变量和 `users[i]`。
4. 写出错误的 `&u` 版本，再改成 `&users[i]`。
5. 对 `"Go语言"` range，打印字节下标和 rune。
