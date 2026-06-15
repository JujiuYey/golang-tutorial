# 07: flag 与 os.Args：命令行参数

`flag` 包用于解析简单命令行参数。它适合小型 CLI、脚本工具和教学项目。

---

## 1. os.Args

```go
fmt.Println(os.Args)
```

`os.Args[0]` 是程序名，后面是命令行参数。

```go
if len(os.Args) < 2 {
    fmt.Println("missing file")
    return
}
path := os.Args[1]
```

直接使用 `os.Args` 适合非常简单的位置参数。

---

## 2. flag 基本用法

```go
name := flag.String("name", "world", "name to greet")
verbose := flag.Bool("v", false, "enable verbose output")

flag.Parse()

fmt.Println(*name, *verbose)
```

`flag.String` 返回 `*string`。解析后通过指针取值。

---

## 3. Var 形式

```go
var port int
flag.IntVar(&port, "port", 8080, "server port")
flag.Parse()
```

`IntVar` 把解析结果写入已有变量。

两种写法都常见。小程序用返回指针更快，大一点的配置结构可以用 Var 形式。

---

## 4. 非 flag 参数

```go
flag.Parse()
args := flag.Args()
```

例如：

```bash
app -v input.txt output.txt
```

`flag.Args()` 会得到 `input.txt` 和 `output.txt`。

---

## 5. 自定义 FlagSet

```go
fs := flag.NewFlagSet("serve", flag.ContinueOnError)
port := fs.Int("port", 8080, "port")

if err := fs.Parse(args); err != nil {
    return err
}
fmt.Println(*port)
```

`FlagSet` 适合子命令：

```bash
app serve -port=8080
app migrate -dry-run
```

---

## 6. 自定义 Usage

```go
flag.Usage = func() {
    fmt.Fprintf(flag.CommandLine.Output(), "Usage: app [options] <file>\n")
    flag.PrintDefaults()
}
```

当参数错误或用户请求帮助时，清晰的 Usage 很重要。

---

## 7. flag 的限制

标准库 `flag` 不支持复杂 CLI 体验，比如：

- 多层子命令自动帮助。
- 必填参数声明。
- 环境变量绑定。
- shell completion。

课程阶段先掌握标准库。复杂 CLI 可以在项目章节再讨论第三方库。

---

## 8. 参数校验

解析成功不代表参数合法。

```go
if *port <= 0 || *port > 65535 {
    return fmt.Errorf("invalid port: %d", *port)
}
```

flag 负责解析，业务代码负责校验。

---

## 9. 错误输出

```go
fmt.Fprintln(os.Stderr, "error:", err)
os.Exit(1)
```

错误信息通常写到 stderr。正常结果写到 stdout。CLI 工具要区分这两者，方便脚本组合。

---

## 10. 一个小 CLI

```go
func run(args []string) error {
    fs := flag.NewFlagSet("greet", flag.ContinueOnError)
    name := fs.String("name", "world", "name to greet")
    if err := fs.Parse(args); err != nil {
        return err
    }
    fmt.Printf("hello, %s\n", *name)
    return nil
}
```

把逻辑写进 `run(args []string) error`，测试时可以直接传参数，不必真的启动进程。

---

## 练习

1. 写一个 `greet` CLI，支持 `-name` 和 `-v`。
2. 用 `flag.Args()` 读取输入文件和输出文件路径。
3. 用 `FlagSet` 实现 `serve` 子命令。
4. 对 `-port` 做范围校验。
5. 把 CLI 逻辑写成 `run(args []string) error`，方便测试。
