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

`[]byte(s)` 和 `[]rune(s)` 是**类型转换**（和 `float64(10)` 是同一种语法）：把同一个字符串，分别"倒进"两种不同的切片容器里。

```go
func main() {
    s := "语言"

    fmt.Println([]byte(s))
    fmt.Println([]rune(s))
}
```

输出：

```text
[232 175 173 232 168 128]
[35821 35328]
```

同一个 `"语言"`，为什么一个输出 6 个数、一个输出 2 个数？

### []byte(s)：按"存储"看 —— 6 个字节

字符串在内存里就是一串字节。中文在 UTF-8 里每个字占 3 字节，所以 2 个字 = 6 个字节：

```text
[232 175 173 | 232 168 128]
 └── 语 ────┘  └── 言 ────┘
```

这 6 个数字就是 `"语言"` 在内存里的原始模样。`len(s)` 数的正是它，所以 `len("语言") = 6`。

### []rune(s)：按"字符"看 —— 2 个码点

`[]rune(s)` 会把字节**解码**成一个个 Unicode 码点（每个字一个数字）：

```text
[35821, 35328]
  └─语    └─言
```

`string(35821)` 能转回 `"语"` —— 和第 5 节 `rune` 是码点的说法接上了。

### 一句话总结

| 转换 | 视角 | `"语言"` 的结果 | 长度 |
|------|------|----------------|------|
| `[]byte(s)` | 机器怎么存 | 6 个字节 | 6 |
| `[]rune(s)` | 人怎么数字 | 2 个码点 | 2 |

**什么时候用哪个**：要按"第几个字"操作（取字符、截断、反转），先转 `[]rune`，否则会从中文的 3 个字节中间切开、产生乱码；处理网络数据、文件 IO 这类原始字节流时用 `[]byte`。

验证：

```go
s := "语言"
fmt.Println(len(s), len([]byte(s)), len([]rune(s))) // 6 6 2
fmt.Println(string([]rune(s)[0]))                   // 语（按字符取第 1 个）
```

---

## 7. 字符串不可变

### 不可变是什么意思

**读可以，写不行。** `s[0]` 取值没问题（第 3 节做过），但给 `s[0]` 赋值直接编译错误：

```go
func main() {
    s := "hello"
    s[0] = 'H'
    // 编译错误：cannot assign to s[0] (neither addressable nor a map index expression)
}
```

一旦创建，字符串的内容到死都不会变。看起来在"改字符串"的操作（比如拼接），其实都是**造了一个新字符串**：

```go
a := "go"
c := a + "lang"
fmt.Println(a, c) // go golang —— a 本身没变
```

### 为什么这样设计

- **可以放心共享**：多个变量、多个 goroutine 引用同一个字符串，谁都改不了它，不需要加锁、不需要防御性拷贝。
- **可以当 map 的 key**：内容永不变，哈希值才稳定。
- 对比 JS：JS 的字符串同样不可变（`s[0] = 'H'` 静默失败），Go 只是把它变成了显式的编译错误。

### 那想改怎么办：转出来改，再转回去

字符串本身改不了，但可以把内容**复制**到切片里（切片可变），改完再转回新字符串。三步：转出 → 修改 → 转回。

改 ASCII 字符，转 `[]byte`：

```go
s := "hello"
b := []byte(s)   // ① 复制成字节切片
b[0] = 'H'       // ② 改切片（切片可变）
s2 := string(b)  // ③ 转回新字符串
fmt.Println(s, s2) // hello Hello —— 原字符串 s 完好无损
```

含中文时必须转 `[]rune`（第 6 节的选择规则）：

```go
t := "go语言"
r := []rune(t)
r[2] = '雅'          // 按"第几个字"定位，不会切坏 3 字节的中文
fmt.Println(string(r)) // go雅言
```

如果对 `"go语言"` 用 `[]byte` 去改第 2 个位置，改到的是"语"3 个字节中的某 1 个字节，结果是乱码——这就是第 6 节"按字符操作用 `[]rune`"的原因。

### 注意：转换是复制，不是引用

`[]byte(s)` 会把字符串内容**复制一份**（正因为字符串不可变，不复制就没法给你一个可写的切片）。所以：

- 改切片不影响原字符串（上面 `s` 依然是 `hello`）
- 超大字符串上频繁来回转换有性能成本，后面学 `strings.Builder` 时会给出高效拼接方案

---

## 练习

写一个程序，输出字符串 `"Go语言"` 的字节数和字符数。

要求：

1. 字节数使用 `len`。
2. 字符数使用 `for range` 统计。
3. 打印每个字符对应的字节索引。
