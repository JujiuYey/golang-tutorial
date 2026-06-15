# 04: strings、strconv 与 bytes

文本处理是标准库的高频场景。`strings` 处理字符串，`strconv` 处理字符串和基础类型转换，`bytes` 处理 `[]byte`。

---

## 1. strings.Contains、Prefix、Suffix

```go
s := "hello, go"

fmt.Println(strings.Contains(s, "go"))   // true
fmt.Println(strings.HasPrefix(s, "he"))  // true
fmt.Println(strings.HasSuffix(s, "go"))  // true
```

这些函数区分大小写。

---

## 2. Split 和 Join

```go
parts := strings.Split("a,b,c", ",")
fmt.Println(parts) // [a b c]

s := strings.Join([]string{"a", "b", "c"}, "-")
fmt.Println(s) // a-b-c
```

`Split` 不适合完整 CSV 解析。CSV 有引号、转义等规则，应使用 `encoding/csv`。

---

## 3. Trim

```go
fmt.Println(strings.TrimSpace("  hello\n"))
fmt.Println(strings.TrimPrefix("go.mod", "go."))
fmt.Println(strings.TrimSuffix("main_test.go", "_test.go"))
```

`TrimSpace` 去掉 Unicode 空白字符。

---

## 4. Replace

```go
s := strings.ReplaceAll("hello world", "world", "go")
fmt.Println(s) // hello go
```

如果只替换前 n 次：

```go
s := strings.Replace("a-a-a", "a", "b", 2)
fmt.Println(s) // b-b-a
```

---

## 5. strings.Builder

大量拼接字符串时，使用 `strings.Builder`。

```go
var b strings.Builder
for i := 0; i < 3; i++ {
    b.WriteString("go")
}
fmt.Println(b.String())
```

少量拼接直接用 `+` 就可以。不要为了几段字符串过度使用 Builder。

---

## 6. strconv 转整数

```go
n, err := strconv.Atoi("123")
if err != nil {
    return err
}
fmt.Println(n)
```

`Atoi` 是 `ParseInt` 的便捷版本。

```go
n64, err := strconv.ParseInt("123", 10, 64)
```

参数含义：

- `"123"`: 输入字符串。
- `10`: 进制。
- `64`: 返回整数的位数。

---

## 7. 数字转字符串

```go
s := strconv.Itoa(123)
fmt.Println(s)
```

指定格式：

```go
s := strconv.FormatInt(123, 10)
```

---

## 8. float 和 bool 转换

```go
f, err := strconv.ParseFloat("3.14", 64)
if err != nil {
    return err
}

s := strconv.FormatFloat(f, 'f', 2, 64)
fmt.Println(s) // 3.14
```

布尔值：

```go
b, err := strconv.ParseBool("true")
s := strconv.FormatBool(b)
```

---

## 9. bytes 包

`bytes` 包处理 `[]byte`，很多函数和 `strings` 类似。

```go
data := []byte("hello, go")

fmt.Println(bytes.Contains(data, []byte("go")))
fmt.Println(bytes.TrimSpace([]byte("  hello\n")))
```

当数据来自网络、文件、编码器时，常常是 `[]byte`，不一定要立刻转成 string。

---

## 10. bytes.Buffer

```go
var buf bytes.Buffer

buf.WriteString("hello")
buf.WriteByte(' ')
buf.WriteString("go")

fmt.Println(buf.String())
```

`bytes.Buffer` 同时实现了 `io.Reader` 和 `io.Writer`，很适合在测试里模拟输入输出。

---

## 练习

1. 写函数 `ParsePort(s string) (int, error)`，要求端口在 1 到 65535。
2. 写函数 `NormalizeName(s string) string`，去掉两端空白并转小写。
3. 写函数把 `[]string{"a", "b", "c"}` 转成 `"a,b,c"`。
4. 用 `strings.Builder` 拼接 100 个数字。
5. 用 `bytes.Buffer` 测试一个接收 `io.Writer` 的函数。
