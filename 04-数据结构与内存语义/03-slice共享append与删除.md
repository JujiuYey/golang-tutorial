# 03: slice 共享、append 与删除

slice 最容易出错的地方在于：多个 slice 可能共享同一个底层数组。理解共享关系，才能判断修改和 `append` 的影响范围。

---

## 1. 多个 slice 可以共享底层数组

```go
nums := []int{10, 20, 30, 40, 50}
a := nums[1:4]

a[0] = 200

fmt.Println(a)    // [200 30 40]
fmt.Println(nums) // [10 200 30 40 50]
```

`a[0]` 对应的是 `nums[1]`。修改元素时，改的是共享的底层数组。

---

## 2. append 容量足够时会复用底层数组

```go
nums := []int{10, 20, 30, 40, 50}
a := nums[1:3]

fmt.Println(len(a), cap(a)) // 2 4

a = append(a, 99)

fmt.Println(a)    // [20 30 99]
fmt.Println(nums) // [10 20 30 99 50]
```

`a` 的容量足够，`append` 会把新元素写进原来的底层数组。

---

## 3. append 容量不足时会分配新数组

```go
nums := []int{10, 20, 30}
a := nums[:3]

fmt.Println(len(a), cap(a)) // 3 3

a = append(a, 99)
a[0] = 100

fmt.Println(a)    // [100 20 30 99]
fmt.Println(nums) // [10 20 30]
```

容量不足时，`append` 会分配新底层数组并复制原有元素。之后 `a` 和 `nums` 不再共享同一段底层数组。

---

## 4. 用完整切片表达式限制共享影响

```go
nums := []int{10, 20, 30, 40, 50}
a := nums[1:3:3]

fmt.Println(len(a), cap(a)) // 2 2

a = append(a, 99)

fmt.Println(a)    // [20 30 99]
fmt.Println(nums) // [10 20 30 40 50]
```

`nums[1:3:3]` 把 `a` 的容量限制为 2。再次 append 时容量不足，会分配新数组，从而避免覆盖 `nums[3]`。

---

## 5. 预分配容量

如果知道大概会追加多少元素，可以提前分配容量，减少扩容次数。

```go
func squares(n int) []int {
    result := make([]int, 0, n)
    for i := 0; i < n; i++ {
        result = append(result, i*i)
    }
    return result
}
```

`make([]int, 0, n)` 表示当前长度为 0，容量为 n。这样可以从空结果开始追加，同时提前准备好空间。

---

## 6. 删除元素

保留顺序的删除：

```go
func deleteAt(s []int, i int) []int {
    return append(s[:i], s[i+1:]...)
}
```

使用前必须保证 `i` 合法：

```go
if i < 0 || i >= len(s) {
    return s
}
```

不保留顺序的删除可以更快：

```go
func deleteAtUnordered(s []int, i int) []int {
    s[i] = s[len(s)-1]
    return s[:len(s)-1]
}
```

这种写法会把最后一个元素移到被删除的位置。

---

## 7. 原地过滤

如果要保留满足条件的元素，可以复用原 slice 的底层数组：

```go
func positive(nums []int) []int {
    result := nums[:0]
    for _, n := range nums {
        if n > 0 {
            result = append(result, n)
        }
    }
    return result
}
```

`nums[:0]` 长度为 0，容量保留。后续 `append` 会尽量复用原底层数组。

---

## 8. 克隆 slice

如果需要得到完全独立的一份数据，可以复制元素。

```go
func clone(nums []int) []int {
    out := make([]int, len(nums))
    copy(out, nums)
    return out
}
```

也可以用 append 写法：

```go
out := append([]int(nil), nums...)
```

克隆之后，修改 `out` 的元素不会影响 `nums`。

---

## 9. 避免无意保留大数组

从大 slice 中切出小 slice，会继续引用原底层数组。

```go
func firstTen(data []byte) []byte {
    return data[:10]
}
```

如果 `data` 很大，而返回值长期存在，原来的大数组也可能被保留在内存中。可以复制需要的部分：

```go
func firstTen(data []byte) []byte {
    out := make([]byte, 10)
    copy(out, data[:10])
    return out
}
```

这类问题在处理大文件、大响应、大缓存时尤其值得注意。

---

## 10. append 后不要继续依赖旧 slice

```go
s := []int{1, 2, 3}
t := append(s, 4)
```

`t` 是 append 后的结果。之后应该围绕 `t` 继续写逻辑。不要假设 `s` 和 `t` 一定共享，也不要假设它们一定不共享。

更清晰的写法：

```go
s = append(s, 4)
```

如果确实需要保留原始数据，先 clone，再 append。

---

## 练习

1. 写一段代码验证 `append` 容量足够时会影响原底层数组。
2. 使用完整切片表达式 `s[low:high:max]`，让 append 不影响原 slice。
3. 实现 `DeleteAt(s []int, i int) ([]int, bool)`，下标非法时返回 `false`。
4. 实现 `CloneStrings(s []string) []string`，确认修改克隆结果不会影响原数据。
5. 实现 `CompactPositive(nums []int) []int`，原地过滤出所有正数。
