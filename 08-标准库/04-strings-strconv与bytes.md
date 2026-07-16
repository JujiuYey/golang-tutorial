# 04: strings、strconv 与 bytes

这篇解决文本处理的日常三件事:查找替换(`strings`)、字符串和数字互转(`strconv`)、以及同一套操作的 `[]byte` 版本(`bytes`)。学完你能躲开一个从 JS 带过来的隐患:**`parseInt("12a")` 在 JS 里悄悄给你 12,Go 的 `Atoi` 会直接报错**——这是好事。

先校准:JS 的字符串方法长在原型上(`s.includes(...)`),Go 的长在包里(`strings.Contains(s, ...)`)——因为 Go 的 string 是纯粹的值类型,没有方法,功能全靠 `strings` 包的函数。

---

## 1. 查找:Contains、HasPrefix、HasSuffix

```go
func main() {
    s := "hello, go"

    fmt.Println(strings.Contains(s, "go"))
    fmt.Println(strings.HasPrefix(s, "he"))
    fmt.Println(strings.HasSuffix(s, "go"))
    fmt.Println(strings.Contains(s, "GO"))
}
```

```text
true
true
true
false
```

对应 JS 的 `includes` / `startsWith` / `endsWith`。最后一行提醒:**全部区分大小写**。要忽略大小写,常用 `strings.EqualFold(a, b)` 或先 `strings.ToLower` 再比。

---

## 2. 拆与合:Split 和 Join

```go
func main() {
    parts := strings.Split("a,b,c", ",")
    fmt.Println(parts, len(parts))

    s := strings.Join([]string{"a", "b", "c"}, "-")
    fmt.Println(s)
}
```

```text
[a b c] 3
a-b-c
```

和 JS 的 `split` / `join` 一一对应,只是 `Join` 的分隔符是第二个参数,不是方法调用。

🕳️ 坑:别拿 `Split` 解析 CSV。CSV 有引号包裹、字段内逗号、转义等规则,`Split(",")` 遇到 `"Smith, John"` 就拆错了——用 `encoding/csv`。

---

## 3. 修剪:Trim 家族

```go
func main() {
    fmt.Printf("%q\n", strings.TrimSpace("  hello\n"))
    fmt.Println(strings.TrimPrefix("go.mod", "go."))
    fmt.Println(strings.TrimSuffix("main_test.go", "_test.go"))
    fmt.Println(strings.Trim("--go--", "-"))
}
```

```text
"hello"
mod
main
go
```

| 函数 | 剪什么 |
|------|--------|
| `TrimSpace` | 两端的 Unicode 空白(空格、`\n`、`\t`……) |
| `TrimPrefix` / `TrimSuffix` | 指定的前缀/后缀(没有就原样返回,不报错) |
| `Trim` | 两端所有出现在字符集合里的字符 |

🕳️ 坑:`Trim("--go--", "-")` 的第二个参数是**字符集合**,不是子串——`Trim(s, "ab")` 会把两端的 `a` 和 `b` 都剪掉,不是剪 `"ab"` 这个整体。想剪整体用 `TrimPrefix` / `TrimSuffix`。

---

## 4. 替换:Replace 和 ReplaceAll

```go
func main() {
    fmt.Println(strings.ReplaceAll("hello world world", "world", "go"))
    fmt.Println(strings.Replace("a-a-a", "a", "b", 2))
}
```

```text
hello go go
b-b-a
```

和 JS 对齐记:`ReplaceAll` = JS 的 `replaceAll`;`Replace(s, old, new, n)` 多了个次数参数,`n` 传 `-1` 等价于全换。注意 JS 的 `replace` 默认只换第一处,Go 没有这个默认——**要么给次数,要么用 ReplaceAll,意图都是显式的**。

---

## 5. 高效拼接:strings.Builder

02-05 说过字符串不可变,`+` 拼接每次都造新字符串。循环里大量拼接时,用 `Builder` 在一块缓冲区里攒:

```go
func main() {
    var b strings.Builder
    for i := 0; i < 3; i++ {
        b.WriteString("go")
    }
    fmt.Println(b.String())
}
```

```text
gogogo
```

`var b strings.Builder` 零值直接可用(不用 New、不用 make),攒完 `b.String()` 一次取出。

分寸感:**少量拼接直接 `+`,循环里成百上千次才值得上 Builder**。为三段字符串掏出 Builder 属于过度设计。

---

## 6. 字符串转数字:strconv.Atoi

```go
func main() {
    n, err := strconv.Atoi("123")
    fmt.Println(n, err)

    m, err := strconv.Atoi("12a")
    fmt.Println(m, err)
}
```

```text
123 <nil>
0 strconv.Atoi: parsing "12a": invalid syntax
```

### 🕳️ 坑:别拿 JS 的 parseInt 直觉套过来

以为会怎样:像 `parseInt("12a")` 一样宽容地给你 `12`。
实际怎样:返回 `0` 和一个错误。
为什么:Go 要求**整串都是合法数字**,拒绝"尽力解析"——静默截断在 JS 里是无数 bug 的来源,Go 用 error 逼你正视脏数据。

需要指定进制或位数时用完整版 `ParseInt`:

```go
n64, err := strconv.ParseInt("ff", 16, 64)
fmt.Println(n64, err)
```

```text
255 <nil>
```

三个参数拆开:

```text
strconv.ParseInt("ff", 16, 64)
//               ─┬──  ─┬  ─┬
//              ①输入 ②进制 ③位数(结果必须装进 int64/32/16/8)
```

`Atoi(s)` 就是 `ParseInt(s, 10, 0)` 的便捷版,日常够用。

---

## 7. 数字转字符串:strconv.Itoa

```go
func main() {
    n := 65

    fmt.Println(strconv.Itoa(n) + "!")
    fmt.Println(string(rune(n)))
    fmt.Println(strconv.FormatInt(255, 16))
    fmt.Println(strconv.FormatInt(5, 2))
}
```

```text
65!
A
ff
101
```

🕳️ 坑:想把 `65` 变成 `"65"`,写 `string(65)` 是错的——那是 02-05 学过的"码点转字符",得到 `"A"`。**数字转字符串必须走 `strconv.Itoa`**(或 `fmt.Sprintf("%d", n)`)。

指定进制用 `FormatInt`:16 进制的 `255` 是 `ff`,2 进制的 `5` 是 `101`——对应 JS 的 `(255).toString(16)`。

---

## 8. float 和 bool 转换

```go
func main() {
    f, err := strconv.ParseFloat("3.14", 64)
    fmt.Println(f, err)
    fmt.Println(strconv.FormatFloat(f, 'f', 1, 64))

    b, err := strconv.ParseBool("true")
    fmt.Println(b, err)

    b2, err := strconv.ParseBool("yes")
    fmt.Println(b2, err)
}
```

```text
3.14 <nil>
3.1
true <nil>
false strconv.ParseBool: parsing "yes": invalid syntax
```

`FormatFloat` 的四个参数:值、格式(`'f'` 普通小数)、小数位数、位数。`ParseBool` 只认 `1/t/T/true/True/TRUE` 和对应的 false 系——`"yes"` 不行,又是"拒绝猜测"的风格。

**一句话总结:strconv 的 Parse 系列全部返回 error,Format/Itoa 系列不会失败。**

---

## 9. bytes 包:strings 的 []byte 版本

上一篇说过,文件、网络来的数据是 `[]byte`。`bytes` 包提供和 `strings` 几乎同名同义的函数,让你**不用先转 string 就能操作**:

```go
func main() {
    data := []byte("hello, go")

    fmt.Println(bytes.Contains(data, []byte("go")))
    fmt.Printf("%q\n", bytes.TrimSpace([]byte("  hello\n")))
}
```

```text
true
"hello"
```

为什么不转成 string 再处理?02-05 讲过:`string(data)` 是一次**复制**。数据量大、操作频繁时,留在 `[]byte` 世界里处理更省。规则:**数据本来是什么就用哪个包,别来回倒手。**

---

## 10. bytes.Buffer:能读能写的字节容器

```go
func main() {
    var buf bytes.Buffer

    buf.WriteString("hello")
    buf.WriteByte(' ')
    buf.WriteString("go")

    fmt.Println(buf.String())
}
```

```text
hello go
```

`bytes.Buffer` 和 `strings.Builder` 都能攒内容,怎么选:

| 类型 | 实现的接口 | 适合 |
|------|-----------|------|
| `strings.Builder` | 只有 Writer | 纯拼字符串,结果要 string |
| `bytes.Buffer` | Reader **和** Writer | 需要"写进去再读出来",比如捕获输出、模拟 IO |

上一篇第 9 节和第 10 篇的测试都会用到 Buffer 的这个"双向"特性。

---

## 本篇重点

- [ ] Go 字符串操作靠 `strings` 包函数,不是方法;`Contains` / `HasPrefix` / `Split` / `Join` 对应 JS 的 `includes` / `startsWith` / `split` / `join`,全部区分大小写。
- [ ] `Trim(s, cutset)` 的第二个参数是字符集合;剪整体前后缀用 `TrimPrefix` / `TrimSuffix`。
- [ ] `strconv.Atoi("12a")` 直接报错,不像 JS 的 `parseInt` 静默给 12;Parse 系列都要接 error。
- [ ] 数字转字符串用 `strconv.Itoa`,写 `string(65)` 得到的是 `"A"`。
- [ ] 数据是 `[]byte` 就用 `bytes` 包原地处理;`strings.Builder` 只写,`bytes.Buffer` 能读能写。

---

## 练习

1. 写函数 `ParsePort(s string) (int, error)`,要求端口在 1 到 65535(提示:先 `Atoi`,err 非 nil 时包装返回;再做范围检查,两种失败都要有清楚的错误信息)。
2. 写函数 `NormalizeName(s string) string`,去掉两端空白并转小写(提示:两个 strings 函数串起来,一行搞定)。
3. 写函数把 `[]string{"a", "b", "c"}` 转成 `"a,b,c"`(提示:一个函数调用)。
4. 用 `strings.Builder` 拼接 100 个数字,格式如 `0,1,2,...`(提示:数字转字符串该用第 7 节的哪个函数?最后一个逗号怎么避免?)。
5. 用 `bytes.Buffer` 测试一个接收 `io.Writer` 的函数(提示:函数往 Writer 里写,测试从 `buf.String()` 里验)。
