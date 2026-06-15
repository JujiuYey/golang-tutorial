# 09: errors、sort、slices、maps 与常用辅助包

标准库里有很多小而实用的辅助包。本节讲错误判断、排序、slice/map 辅助和常见编码工具。

---

## 1. errors.Is

```go
if errors.Is(err, os.ErrNotExist) {
    fmt.Println("file does not exist")
}
```

当错误被 `%w` 包装后，仍然可以用 `errors.Is` 判断错误链中是否包含目标错误。

```go
return fmt.Errorf("open config: %w", os.ErrNotExist)
```

---

## 2. errors.As

```go
var pathErr *fs.PathError
if errors.As(err, &pathErr) {
    fmt.Println(pathErr.Path)
}
```

`errors.As` 用来从错误链中提取特定错误类型。

---

## 3. sort

```go
nums := []int{3, 1, 2}
sort.Ints(nums)
fmt.Println(nums) // [1 2 3]
```

字符串：

```go
names := []string{"bob", "alice"}
sort.Strings(names)
```

自定义排序：

```go
sort.Slice(users, func(i, j int) bool {
    return users[i].Age < users[j].Age
})
```

---

## 4. slices

较新的 Go 标准库提供 `slices` 包。

```go
nums := []int{1, 2, 3}

fmt.Println(slices.Contains(nums, 2))
fmt.Println(slices.Index(nums, 3))
```

排序：

```go
slices.Sort(nums)
```

比较：

```go
slices.Equal([]int{1, 2}, []int{1, 2})
```

---

## 5. maps

`maps` 包提供 map 辅助函数。

```go
a := map[string]int{"go": 1}
b := maps.Clone(a)

b["go"] = 2
fmt.Println(a["go"]) // 1
```

比较：

```go
maps.Equal(map[string]int{"a": 1}, map[string]int{"a": 1})
```

---

## 6. math 和 math/rand

```go
fmt.Println(math.Sqrt(9))
fmt.Println(math.Round(3.6))
```

随机数：

```go
r := rand.New(rand.NewSource(time.Now().UnixNano()))
fmt.Println(r.Intn(10))
```

安全随机数不要用 `math/rand`。密码、token、密钥应该使用 `crypto/rand`。

---

## 7. crypto/rand

```go
buf := make([]byte, 16)
if _, err := rand.Read(buf); err != nil {
    return err
}
```

这里的 `rand` 来自 `crypto/rand`。如果同时导入 `math/rand` 和 `crypto/rand`，通常需要起别名。

---

## 8. encoding/base64

```go
encoded := base64.StdEncoding.EncodeToString(data)

decoded, err := base64.StdEncoding.DecodeString(encoded)
if err != nil {
    return err
}
```

base64 是编码，不是加密。不要把它当成安全保护。

---

## 9. regexp

```go
re := regexp.MustCompile(`^[a-z0-9_]+$`)
fmt.Println(re.MatchString("go_123"))
```

`MustCompile` 在正则非法时 panic，适合包级变量或确定无误的常量正则。动态输入的正则应该用 `regexp.Compile` 并处理 error。

---

## 10. cmp

`cmp` 包提供有序值比较辅助：

```go
fmt.Println(cmp.Compare(1, 2)) // -1
fmt.Println(cmp.Or("", "fallback"))
```

`cmp.Or` 返回第一个非零值，适合简单默认值场景。

---

## 练习

1. 用 `errors.Is` 判断文件不存在。
2. 用 `errors.As` 提取 `*fs.PathError`。
3. 按年龄排序 `[]User`。
4. 用 `slices.Contains` 判断元素是否存在。
5. 用 `maps.Clone` 克隆 map，验证修改克隆不影响原 map。
