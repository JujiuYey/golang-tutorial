# 02: 错误包装、errors.Is 与 errors.As

上一篇的错误只是"一句话"。这篇解决一个更真实的问题：错误层层往上传的过程中，**怎么既给每一层补上下文，又不弄丢最底层"到底出了什么事"**——让调用方既能看到完整来龙去脉，又能程序化地判断"是不是文件不存在"这类具体原因。

---

## 1. 用 %w 包装错误

想象错误往上传的过程是**套快递盒**：底层错误是货物，每一层函数套一个写着自己上下文的盒子。`fmt.Errorf` 的 `%w` 动词（wrap，包装）就是干这个的：

```go
func readConfig(path string) ([]byte, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        return nil, fmt.Errorf("read config %s: %w", path, err)
        //                     ─────┬───────   ─┬─
        //                  ①自己这层的上下文   ②%w:把底层 err 包进去,不弄丢
    }
    return data, nil
}

func main() {
    _, err := readConfig("missing.yaml")
    fmt.Println(err)
}
```

```text
read config missing.yaml: open missing.yaml: no such file or directory
```

看输出：冒号左边是我们补的上下文，右边是 `os.ReadFile` 的原始错误——**一层套一层，链条完整**。

🕳️ 坑：用 `%v` 也能拼出一模一样的字符串，但原始错误被"打印成字"后就死了，链条断掉，后面的 `errors.Is` 会失灵（第 2 节末尾验证）。**包装错误，永远用 `%w`。**

对比 JS：类似 `new Error("read config", { cause: err })` 的 cause 链，只是 Go 用一个格式化动词就搞定了。

---

## 2. errors.Is：问"链条里有没有它"

拿到一个包装过的错误，怎么判断根源是"文件不存在"？直接 `==` 比较不行——错误已经被套了盒子。`errors.Is` 会**一层层拆盒子**，看链条里有没有目标错误：

```go
data, err := readConfig("missing.yaml")
if err != nil {
    if errors.Is(err, os.ErrNotExist) {   // 链条里有"不存在"这个错误吗?
        fmt.Println("配置文件不存在")
        return
    }
    fmt.Println(err)
    return
}
_ = data
```

```text
配置文件不存在
```

`os.ErrNotExist` 是标准库定义好的"文件不存在"错误，`os.ReadFile` 失败时把它埋在了错误链的最底层，`errors.Is` 顺着链条找到了它。

验证第 1 节的坑——`%w` 和 `%v` 的生死区别：

```go
_, err := os.ReadFile("missing.yaml")

wrapped := fmt.Errorf("read config: %w", err)   // 链条还在
flattened := fmt.Errorf("read config: %v", err) // 链条断了

fmt.Println(errors.Is(wrapped, os.ErrNotExist))
fmt.Println(errors.Is(flattened, os.ErrNotExist))
```

```text
true
false
```

两个错误**打印出来一模一样**，但 `%v` 那个已经查不出根源了。

**一句话总结：`%w` 保链，`%v` 断链；判断根源用 `errors.Is`，不要用 `==`。**

---

## 3. 哨兵错误：预先定义好的"暗号"

上面用的 `os.ErrNotExist` 是标准库的**哨兵错误（sentinel error）**：在包级别预先声明的错误变量，当作调用方和函数之间的"暗号"。你也可以定义自己的：

```go
var ErrNotFound = errors.New("not found")   // 包级变量,全包共用这一个

func findUser(id int64) error {
    return ErrNotFound   // 返回的就是那个暗号本身
}
```

调用方对暗号：

```go
err := findUser(1)
if errors.Is(err, ErrNotFound) {
    fmt.Println("user not found")
}
```

```text
user not found
```

命名惯例：哨兵错误以 `Err` 开头、首字母大写（大写 = 包外可见，调用方才能拿它来对）。

---

## 4. 自定义错误类型：错误需要携带数据时

哨兵错误只能表达"是/不是这种错"。如果错误还要携带**结构化信息**（哪个字段、什么问题），就定义一个自己的错误类型：

```go
type ValidationError struct {   // struct:把几个字段打包成一个类型
    Field string
    Msg   string
}

func (e *ValidationError) Error() string {   // 实现 Error() string 方法
    return e.Field + ": " + e.Msg            // → 实现 error 接口
}
```

这里正好把第 04、05 章串起来：`ValidationError` 是一个 struct，`Error()` 是它的方法；有了这个方法，它就隐式实现了内置的 `error` 接口。错误因此既能打印成一句话，也能保留 `Field` 和 `Msg` 这些结构化数据。

---

## 5. errors.As：从链条里"捞出"某种类型的错误

`errors.Is` 问的是"有没有**这一个**错误"；`errors.As` 问的是"有没有**这一种类型**的错误，有就取出来给我用"：

```go
var err error = &ValidationError{Field: "email", Msg: "不能为空"}
fmt.Println(err)

var validationErr *ValidationError
if errors.As(err, &validationErr) {      // 链条里有 *ValidationError 吗?
    fmt.Println(validationErr.Field)     // 有 → 捞出来,读它携带的数据
}
```

```text
email: 不能为空
email
```

拿到 `validationErr` 之后，就能访问 `.Field`、`.Msg` 这些具体字段——这是哨兵错误做不到的。

| 工具 | 问题 | 拿到什么 |
|------|------|---------|
| `errors.Is(err, ErrX)` | 链条里有 ErrX 这**个**错误吗 | true/false |
| `errors.As(err, &target)` | 链条里有这**种**类型的错误吗 | true/false + 错误本体（可读字段） |

---

## 6. 什么时候包装错误

推荐包装：

- 跨函数边界往上传时，补一句自己这层的上下文（做了什么、操作对象是谁）。
- 底层错误对调用方有判断价值（比如"不存在"和"没权限"要走不同分支）。

不推荐：

- 上下文只是把同样的话再说一遍（`fmt.Errorf("error: %w", err)` 毫无信息量）。
- 刻意**不想**让调用方依赖底层细节——那就用 `%v` 或重新造一个错误，主动断链（断链从坑变成了手段，关键是"故意的"）。

---

## 本篇重点

- `fmt.Errorf("...: %w", err)` 包装错误：补上下文的同时保留错误链；`%v` 会断链。
- 判断链条里是否有某个错误，用 `errors.Is`，不要直接 `==`。
- 哨兵错误 = 包级预定义的错误变量（`var ErrXxx = errors.New(...)`），当"暗号"用。
- 错误要携带数据时，自定义错误类型 + `errors.As` 捞出来读字段（结构体后面会细讲）。
- 包装的目的是补上下文，不是复读；想隐藏底层细节时才故意断链。

---

## 练习

实现函数：

```go
func readRequiredFile(path string) ([]byte, error)
```

要求：

1. 使用 `os.ReadFile` 读取文件。
2. 读取失败时用 `%w` 包装错误，带上文件路径。
3. 调用方使用 `errors.Is(err, os.ErrNotExist)` 判断文件不存在。

提示：故意传一个不存在的路径来验证；再把 `%w` 换成 `%v` 跑一次，观察 `errors.Is` 的结果变化——亲眼看一次断链。
