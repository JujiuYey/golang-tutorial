# 03: path/filepath 与文件系统操作

这篇解决一类"在我机器上好好的"问题:路径怎么拼才能跨平台、怎么判断文件存在、怎么建目录/扫目录/用临时目录。核心纪律一句话:**路径永远交给 `path/filepath`,别手写字符串拼接。**

对 Node 用户:`path/filepath` ≈ Node 的 `path` 模块 + `fs` 里的目录操作。思路几乎一样,连"别硬编码 `/`"的理由都一样。

---

## 1. filepath.Join:拼路径的唯一正确姿势

```go
func main() {
    fmt.Println(filepath.Join("configs", "app.json"))
    fmt.Println(filepath.Join("configs/", "/app.json"))
    fmt.Println(filepath.Join("a", "..", "b"))
}
```

```text
configs/app.json
configs/app.json
b
```

三个输出说明了 `Join` 帮你干的事:用当前系统的分隔符拼接、吃掉多余的斜杠、顺手把 `..` 这类冗余清理掉(自动 Clean)。

不要这样写:

```go
path := "configs/" + "app.json" // ❌ 硬编码分隔符
```

在 Windows 上分隔符是 `\`,手拼 `/` 就是给自己埋跨平台的雷——和 Node 里推荐 `path.join` 而不是模板字符串是同一个道理。

---

## 2. 拆路径:Clean、Dir、Base、Ext

拿到一条路径,四个函数各拆一块:

```go
func main() {
    p := "/tmp/app/config.json"

    fmt.Println(filepath.Clean("/tmp/app/../app/config.json"))
    fmt.Println(filepath.Dir(p))
    fmt.Println(filepath.Base(p))
    fmt.Println(filepath.Ext(p))
}
```

```text
/tmp/app/config.json
/tmp/app
config.json
.json
```

| 函数 | 拿到什么 | Node 对应 |
|------|---------|-----------|
| `Clean` | 规范化(消掉 `..`、多余斜杠) | `path.normalize` |
| `Dir` | 目录部分 | `path.dirname` |
| `Base` | 文件名部分 | `path.basename` |
| `Ext` | 扩展名(带点) | `path.extname` |

---

## 3. 绝对路径:filepath.Abs

```go
func main() {
    abs, err := filepath.Abs("config.json")
    if err != nil {
        fmt.Println(err)
        return
    }
    fmt.Println(abs)
}
```

```text
/Users/jujiuyey/Desktop/golang-tutorial/config.json
```

(输出取决于你在哪个目录运行——你的路径会不同。)

`Abs` 是"当前工作目录 + 相对路径"。🕳️ 坑:**当前工作目录不是程序所在目录**,而是"你在哪里敲的命令"。同一个程序,`cd` 到不同地方启动,`Abs` 结果就不同。所以业务逻辑不要过度依赖相对路径,配置文件路径最好由参数或环境变量显式传入。

---

## 4. 判断文件是否存在

Go 没有 `fs.existsSync`,惯用写法是 `os.Stat` + 错误判断:

```go
func check(path string) {
    _, err := os.Stat(path)
    if err == nil {
        fmt.Println(path, "=> 存在")
    } else if errors.Is(err, os.ErrNotExist) {
        fmt.Println(path, "=> 不存在")
    } else {
        fmt.Println(path, "=>", err) // 存在与否未知,比如没权限
    }
}

func main() {
    check("data/app.json")
    check("data/ghost.json")
}
```

```text
data/app.json => 存在
data/ghost.json => 不存在
```

注意是三个分支,不是两个:`Stat` 出错**不一定**代表文件不存在(可能是权限问题),所以用 `errors.Is(err, os.ErrNotExist)` 精确判断"不存在",其他错误照常上报。这正是第 09 篇 `errors.Is` 的典型用武之地。

---

## 5. 创建目录:os.MkdirAll

```go
func main() {
    if err := os.MkdirAll("build/cache/tmp", 0755); err != nil {
        fmt.Println(err)
        return
    }
    fmt.Println("第一次: 创建成功")

    if err := os.MkdirAll("build/cache/tmp", 0755); err != nil {
        fmt.Println(err)
        return
    }
    fmt.Println("第二次: 已存在,也不报错")
}
```

```text
第一次: 创建成功
第二次: 已存在,也不报错
```

`MkdirAll` = Node 的 `fs.mkdirSync(p, { recursive: true })`:一口气建多级,已存在不报错——所以可以放心地在程序启动时无脑调用。权限 `0755` 是目录的常规值(上一篇拆过这组数字)。

---

## 6. 读取目录:os.ReadDir

下面几节的示例都基于这个目录结构:

```text
data/
├── app.json
├── notes.txt
└── cache/
    └── user.json
```

```go
func main() {
    entries, err := os.ReadDir("data")
    if err != nil {
        fmt.Println(err)
        return
    }

    for _, entry := range entries {
        fmt.Println(entry.Name(), entry.IsDir())
    }
}
```

```text
app.json false
cache true
notes.txt false
```

`os.ReadDir` 只看一层、不递归,结果按文件名排好序。每个 `entry` 能问出名字、是不是目录,要更多信息(大小、修改时间)再调 `entry.Info()`。

---

## 7. 遍历目录树:filepath.WalkDir

要递归,用 `WalkDir`——它带着你的回调函数走遍整棵树:

```go
func main() {
    err := filepath.WalkDir("data", func(path string, d fs.DirEntry, err error) error {
        if err != nil {
            return err // 某个子目录读失败,先上报
        }
        if d.IsDir() {
            return nil // 目录本身跳过,继续往里走
        }
        fmt.Println(path)
        return nil
    })
    if err != nil {
        fmt.Println(err)
    }
}
```

```text
data/app.json
data/cache/user.json
data/notes.txt
```

回调的三个参数拆开看:

```text
func(path string, d fs.DirEntry, err error) error
//   ────┬─────  ──────┬───────  ────┬────
//  ①当前走到的路径  ②它的目录项信息  ③走到这里之前有没有出错
```

🕳️ 坑:回调里的 `err` 参数要先处理(第 ③ 个)——它表示"进入这个条目时就出错了"(比如子目录没权限),忽略它会静默漏掉一部分文件树。

---

## 8. 通配符匹配:filepath.Glob

```go
func main() {
    matches, err := filepath.Glob("data/*.json")
    if err != nil {
        fmt.Println(err)
        return
    }
    fmt.Println(matches)

    none, err := filepath.Glob("data/*.yaml")
    fmt.Println("匹配数:", len(none), "err:", err)
}
```

```text
[data/app.json]
匹配数: 0 err: <nil>
```

两个注意点:

- **没匹配到不算错误**:返回空 slice 和 nil error。别用 `err != nil` 判断"没找到"。
- `*` 不跨目录:`data/*.json` 匹配不到 `data/cache/user.json`。要递归匹配,用第 7 节的 `WalkDir` + `filepath.Ext` 过滤。

---

## 9. 临时文件和目录

```go
func main() {
    dir, err := os.MkdirTemp("", "app-*")
    if err != nil {
        fmt.Println(err)
        return
    }
    defer os.RemoveAll(dir)

    fmt.Println("临时目录:", dir)
}
```

```text
临时目录: /var/folders/mj/7lrl_grd2gn9xzgdlvt_tlwh0000gn/T/app-2058419586
```

(随机后缀每次都不同,你的输出也会不同。)

第一个参数传 `""` 表示用系统默认临时目录;`app-*` 里的 `*` 会被随机数替换,避免撞名。**创建成功后马上 `defer os.RemoveAll(dir)`**,别留垃圾。

在测试里有更省心的版本——`t.TempDir()`,测试结束自动清理,连 defer 都不用写(第 10 篇细讲):

```go
dir := t.TempDir()
```

---

## 10. 权限速查

本章反复出现的三个八进制权限,记住用途就行:

| 权限 | 含义 | 用在哪 |
|------|------|--------|
| `0644` | 所有者读写,其他人只读 | 普通文件 |
| `0600` | 只有所有者能读写 | 密钥、凭证等敏感文件 |
| `0755` | 所有者全权,其他人可读可进入 | 目录、可执行文件 |

实际生效还受操作系统和 umask 影响,现阶段掌握常见写法即可。

---

## 本篇重点

- [ ] 拼路径只用 `filepath.Join`,它管分隔符、去冗余、跨平台;拆路径用 `Dir` / `Base` / `Ext`。
- [ ] 判断文件存在:`os.Stat` + `errors.Is(err, os.ErrNotExist)`,注意是三分支不是二分支。
- [ ] `os.MkdirAll` 可重复调用不报错;`os.ReadDir` 只看一层,递归用 `filepath.WalkDir`(回调里先处理 err 参数)。
- [ ] `Glob` 没匹配到返回空 slice + nil error,且 `*` 不跨目录。
- [ ] 临时目录:普通代码 `os.MkdirTemp` + `defer os.RemoveAll`;测试里直接 `t.TempDir()`。

---

## 练习

1. 用 `filepath.Join` 拼接配置文件路径 `configs/dev/app.json`(提示:三个参数)。
2. 写函数 `Exists(path string) (bool, error)`(提示:第 4 节的三个分支怎么映射到两个返回值?权限错误应该走哪个?)。
3. 写函数列出某个目录下所有 `.json` 文件(提示:`Glob` 或 `ReadDir` + `filepath.Ext` 都行,想想两者差异)。
4. 用 `filepath.WalkDir` 统计目录里的普通文件数量(提示:闭包捕获一个计数器,回调里 `d.IsDir()` 时跳过)。
5. 用 `os.MkdirTemp` 创建临时目录,并在函数结束时清理(提示:defer 的位置在错误检查之后)。
