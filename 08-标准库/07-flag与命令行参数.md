# 07: flag 与 os.Args:命令行参数

这篇解决"写个带参数的命令行工具"这件事:`app -port=8080 -v input.txt` 这样的调用,参数怎么declare、怎么解析、怎么校验。学完你还能拿到一个让 CLI 可测试的套路:`run(args []string) error`。

对 Node 用户:`os.Args` ≈ `process.argv`,`flag` 包 ≈ 一个内置的迷你版 yargs/commander——功能少,但零依赖。

本篇示例都是编译后运行的(`go build -o greet . && ./greet ...`),因为要传命令行参数;`go run . -name=x` 也可以。

---

## 1. 最原始的入口:os.Args

```go
func main() {
    fmt.Println(os.Args)

    if len(os.Args) < 2 {
        fmt.Println("missing file")
        return
    }
    fmt.Println("file:", os.Args[1])
}
```

```bash
$ ./app
[./app]
missing file

$ ./app notes.txt
[./app notes.txt]
file: notes.txt
```

`os.Args[0]` 是程序名,真正的参数从 `[1]` 开始——和 `process.argv` 的差异注意一下:Node 是 `[node路径, 脚本路径, 参数...]`,参数从 `[2]` 开始;Go 从 `[1]` 开始。

裸用 `os.Args` 只适合"就一个位置参数"的极简场景,带选项就该上 `flag`。

---

## 2. flag 基本用法

```go
func main() {
    name := flag.String("name", "world", "name to greet")
    verbose := flag.Bool("v", false, "enable verbose output")

    flag.Parse()

    fmt.Println(*name, *verbose)
}
```

```bash
$ ./greet
world false

$ ./greet -name=gopher -v
gopher true
```

拆一下声明这行:

```text
name := flag.String("name", "world", "name to greet")
//      ────┬────── ───┬──  ───┬───  ──────┬───────
//     返回 *string  ①参数名  ②默认值   ③帮助文本(-h 时显示)
```

两个要点:

- `flag.String` 返回的是**指针**(`*string`)——声明时值还不存在,`flag.Parse()` 之后才填进去,所以取值要解引用 `*name`(04 章的指针)。
- 🕳️ 坑:所有 flag 声明必须在 `flag.Parse()` **之前**,取值在之后。忘了调 `Parse`,拿到的永远是默认值,不报错。

传了没声明的参数会怎样?flag 直接罢工并打印用法(这是真实输出):

```bash
$ ./greet -port=1
flag provided but not defined: -port
Usage of ./greet:
  -name string
    	name to greet (default "world")
  -v	enable verbose output
```

---

## 3. Var 形式:写进已有变量

不想跟指针打交道,用 `XxxVar` 把结果直接写进你的变量:

```go
func main() {
    var port int
    flag.IntVar(&port, "port", 8080, "server port")
    flag.Parse()

    fmt.Println("port =", port)
}
```

```bash
$ ./server -port=9000
port = 9000
```

传 `&port` 的原因和上一篇 `Unmarshal` 一样:**要往你的变量里写,就得给地址**。两种写法怎么选:小工具用返回指针的写法省事;参数多、想集中放进一个 config struct 时,`Var` 形式更整齐。

---

## 4. 非 flag 参数:flag.Args()

选项之外的"裸参数"(输入文件、输出文件)从 `flag.Args()` 拿:

```go
func main() {
    verbose := flag.Bool("v", false, "verbose")
    flag.Parse()

    fmt.Println("v =", *verbose)
    fmt.Println("args =", flag.Args())
}
```

```bash
$ ./app2 -v input.txt output.txt
v = true
args = [input.txt output.txt]
```

### 🕳️ 坑:flag 必须写在位置参数前面

以为会怎样:`./app2 input.txt -v` 和上面等价(很多 Unix 工具确实随便放)。
实际怎样:

```bash
$ ./app2 input.txt -v
v = false
args = [input.txt -v]
```

`-v` 没生效,还被当成了普通参数!为什么:标准库 flag 遇到**第一个非 flag 参数就停止解析**,后面的全进 `Args()`。这是 Go flag 和 GNU 风格工具的著名差异,写文档时提醒用户"选项在前"。

---

## 5. 子命令:flag.NewFlagSet

想做 `app serve -port=8080`、`app migrate -dry-run` 这种子命令结构,给每个子命令建一个独立的 `FlagSet`:

```go
func main() {
    if len(os.Args) < 2 {
        fmt.Fprintln(os.Stderr, "expected subcommand: serve")
        os.Exit(1)
    }

    switch os.Args[1] {
    case "serve":
        fs := flag.NewFlagSet("serve", flag.ContinueOnError)
        port := fs.Int("port", 8080, "port")
        if err := fs.Parse(os.Args[2:]); err != nil {
            os.Exit(1)
        }
        fmt.Println("serving on port", *port)
    default:
        fmt.Fprintln(os.Stderr, "unknown subcommand:", os.Args[1])
        os.Exit(1)
    }
}
```

```bash
$ ./tool serve -port=9090
serving on port 9090
```

思路:`os.Args[1]` 是子命令名,切下来 switch;剩下的 `os.Args[2:]` 交给对应的 FlagSet 解析。`flag.ContinueOnError` 表示解析失败返回 error 而不是直接退出进程——方便自己控制错误处理。

---

## 6. 自定义 Usage

`-h`/`--help` 和解析出错时打印的用法文本,可以换成自己的:

```go
flag.Usage = func() {
    fmt.Fprintf(flag.CommandLine.Output(), "Usage: app [options] <file>\n")
    flag.PrintDefaults()
}
```

```bash
$ ./app3 -h
Usage: app [options] <file>
  -name string
    	name to greet (default "world")
```

`PrintDefaults()` 负责列出所有已声明的 flag 和默认值,你只需要补上开头那行"这程序怎么用"。CLI 的第一印象就是 `-h`,值得花两行代码。

---

## 7. flag 的边界:它不做的事

标准库 flag 是**迷你**工具,这些它都没有:

- 多层子命令的自动帮助
- 必填参数声明
- 环境变量绑定
- shell 自动补全

对比你可能用过的 commander/yargs,flag 大概只有它们的核心 20%。课程阶段这就够了;真要做复杂 CLI,项目章节再引入第三方库(如 cobra)。

---

## 8. 解析 ≠ 合法:参数校验

flag 只保证"是个整数",不保证"是个合理的端口"——校验是业务代码的事:

```go
port := flag.Int("port", 8080, "server port")
flag.Parse()

if *port <= 0 || *port > 65535 {
    fmt.Fprintf(os.Stderr, "error: invalid port: %d\n", *port)
    os.Exit(1)
}
fmt.Println("listening on", *port)
```

```bash
$ ./server2 -port=99999
error: invalid port: 99999
exit=1

$ ./server2 -port=8080
listening on 8080
```

**一句话总结:flag 负责"解析成类型",你负责"值是否合法"。**

---

## 9. 错误去 stderr,结果去 stdout

上面示例里反复出现的 `os.Stderr` 不是装饰:

```go
fmt.Fprintln(os.Stderr, "error:", err) // 错误 → stderr
os.Exit(1)                             // 非零退出码 = 失败
```

CLI 工具的正常输出走 stdout,诊断和错误走 stderr——这样 `./tool > result.txt` 重定向结果时,错误信息仍然打在屏幕上;管道组合 `./tool | grep ...` 也不会把错误混进数据流。Node 里 `console.log` vs `console.error` 是同一个区分。

---

## 10. 可测试的 CLI:run(args) 模式

把逻辑从 `main` 里抽出来,收 `args` 返回 `error`:

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

func main() {
    if err := run(os.Args[1:]); err != nil {
        fmt.Fprintln(os.Stderr, "error:", err)
        os.Exit(1)
    }
}
```

```bash
$ ./greet2 -name=go
hello, go
```

好处在测试时兑现:`run([]string{"-name", "test"})` 直接调用就能测,**不用真的启动进程、不用碰全局的 `flag.CommandLine`**。`main` 退化成三行胶水:调 run、打错误、定退出码。这个模式第 10 篇写测试时直接受益。

---

## 本篇重点

- [ ] `os.Args[0]` 是程序名,参数从 `[1]` 开始(Node 从 `[2]`);裸用只适合极简场景。
- [ ] flag 三步:声明(返回指针或 `XxxVar` 写入变量)→ `flag.Parse()` → 解引用取值;声明必须在 Parse 前。
- [ ] 大坑:标准库 flag 遇到第一个非 flag 参数就停——选项必须写在位置参数前面。
- [ ] 子命令用 `flag.NewFlagSet` + `os.Args[2:]`;解析负责类型,校验(端口范围等)是你自己的事。
- [ ] 错误走 stderr + 非零退出码,结果走 stdout;逻辑抽成 `run(args []string) error`,CLI 就可测试。

---

## 练习

1. 写一个 `greet` CLI,支持 `-name` 和 `-v`(提示:`-v` 时多打印一行诊断信息到 stderr)。
2. 用 `flag.Args()` 读取输入文件和输出文件路径(提示:先检查 `len(flag.Args()) == 2`,不够就打印用法并退出)。
3. 用 `FlagSet` 实现 `serve` 子命令,支持 `-port`(提示:第 5 节的 switch 骨架)。
4. 对 `-port` 做范围校验,非法时输出到 stderr 并以退出码 1 结束(提示:第 8、9 节合起来)。
5. 把上面的 CLI 逻辑重构成 `run(args []string) error`,让 `main` 只剩胶水代码(提示:FlagSet 用 `flag.ContinueOnError`,错误一路 return 到 main 统一处理)。
