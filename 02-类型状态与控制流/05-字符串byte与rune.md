# 05: 字符串、byte 与 rune

字符串是字节序列。`len(s)` 返回字节数，不是字符数。

---

## 1. 双引号字符串

双引号字符串支持转义字符。

```go
func main() {
    s := "hello\nworld"
    fmt.Println(s)
}
```

常见转义：

| 写法 | 含义 |
|------|------|
| `\n` | 换行 |
| `\t` | 制表符 |
| `\"` | 双引号 |
| `\\` | 反斜杠 |

---

## 2. 反引号字符串

反引号字符串保留原始内容，不处理转义。

```go
func main() {
    s := `hello\nworld`
    fmt.Println(s)
}
```

反引号常用于多行字符串、SQL、模板等。

```go
query := `
SELECT id, name
FROM users
WHERE active = true
`
```

---

## 3. 字符串索引拿到的是 byte

```go
package main

import "fmt"

func main() {
    s := "Go语言"

    fmt.Println(len(s))       // 8
    fmt.Println(s[0])         // 71
    fmt.Println(string(s[0])) // G
}
```

`s[0]` 取到的是第 0 个字节，不是第 0 个字符。

---

## 4. range 按 rune 遍历

中文字符通常占多个字节。按字符遍历时，用 `range`：

```go
package main

import "fmt"

func main() {
    s := "Go语言"

    for i, r := range s {
        fmt.Printf("index=%d rune=%c type=%T\n", i, r, r)
    }
}
```

可能输出：

```text
index=0 rune=G type=int32
index=1 rune=o type=int32
index=2 rune=语 type=int32
index=5 rune=言 type=int32
```

索引是字节位置，所以 `语` 后面的索引从 `2` 跳到 `5`。

---

## 5. byte 和 rune

- `byte` 是 `uint8` 的别名，适合处理原始字节。
- `rune` 是 `int32` 的别名，适合表示 Unicode 码点。

```go
func main() {
    var b byte = 'A'
    var r rune = '你'

    fmt.Printf("%T %v %c\n", b, b, b)
    fmt.Printf("%T %v %c\n", r, r, r)
}
```

---

## 6. string、[]byte、[]rune

```go
func main() {
    s := "语言"

    fmt.Println([]byte(s))
    fmt.Println([]rune(s))
}
```

`[]byte` 看字节，`[]rune` 看 Unicode 码点。

---

## 7. 字符串不可变

Go 字符串不可变，不能直接修改某个位置。

```go
func main() {
    s := "hello"
    // s[0] = 'H' // 编译错误
    fmt.Println(s)
}
```

需要修改时，通常转成 `[]byte` 或 `[]rune`，修改后再转回 `string`。

---

## 练习

写一个程序，输出字符串 `"Go语言"` 的字节数和字符数。

要求：

1. 字节数使用 `len`。
2. 字符数使用 `for range` 统计。
3. 打印每个字符对应的字节索引。
