# 01: 工具链与 Hello World

这是全系列第一篇,解决三件事:把 Go 装好、跑起你的第一个程序、看懂 `go.mod` 是什么。学完你能用 `go run` 和 `go build` 跑代码,并且不会被"go.mod file not found"这类报错卡住。

先给 JS 用户校准坐标:Go 的工具链体验和 Node 很像——`go` 命令 ≈ `node` + `npm` 合体,`go.mod` ≈ `package.json`。但有一个根本差异:**Go 是编译型语言**,代码先编译成一个独立的可执行文件再运行,不需要装运行时、没有 node_modules 要随身带。

---

## 1. Go 语言简介

### 1.1 Go 是什么

Go(又称 Golang)是 Google 2009 年发布的开源编程语言,设计者是 Robert Griesemer、Rob Pike 和 Ken Thompson(最后这位也是 Unix 和 C 的作者之一)。

它的设计哲学一句话概括:**把语言做小,把工程做顺**。关键字只有 25 个(JS 有 60 多个),但格式化、测试、依赖管理全部内置在官方工具链里。

### 1.2 核心特点

| 特点 | 说明 | 对 JS 用户的意义 |
|------|------|----------------|
| 简洁 | 语法小,25 个关键字 | 上手比 TS 类型体操轻松 |
| 高性能 | 编译型,接近 C 的速度 | 不再有解释器/JIT 开销 |
| 并发原生 | goroutine + channel | 比回调/Promise 更直接的并发模型 |
| 标准库丰富 | HTTP、加密、测试都内置 | 很多场景不用找第三方包 |
| 部署简单 | 编译成单个可执行文件 | 没有 node_modules,扔个二进制就能跑 |
| 静态类型 | 编译时检查 | 相当于强制开启的 TypeScript |

### 1.3 Go 能做什么

- Web 服务器 / API 开发
- 微服务架构
- 云原生应用(Kubernetes、Docker 就是 Go 写的)
- 网络工具和代理
- 数据库和缓存系统
- CLI 工具开发

### 1.4 和其他语言比一比

```text
对比维度          | Go        | Python    | Java      | Node.js
----------------|-----------|-----------|-----------|----------
学习曲线          | ★★★☆☆    | ★☆☆☆☆    | ★★★★☆    | ★★☆☆☆
运行速度          | ★★★★★    | ★★☆☆☆    | ★★★★☆    | ★★★☆☆
并发支持          | ★★★★★    | ★★☆☆☆    | ★★★★☆    | ★★★☆☆
开发效率          | ★★★★☆    | ★★★★★    | ★★★☆☆    | ★★★★☆
生态丰富度        | ★★★★☆    | ★★★★★    | ★★★★★    | ★★★★☆
```

**一句话总结:Go 用"比 Node 略陡一点的学习曲线",换来"接近 C 的性能 + 单文件部署"。**

---

## 2. 开发环境搭建

### 2.1 安装

**Windows**:访问 https://go.dev/dl/ 下载 `.msi` 安装包,双击一路 Next。

**Mac**(推荐 Homebrew):

```bash
brew install go
```

也可以去 https://go.dev/dl/ 下载 `.pkg` 手动装。

**Linux**:

```bash
# Ubuntu/Debian
sudo apt-get update
sudo apt-get install golang-go

# CentOS/RHEL
sudo yum install golang
```

### 2.2 验证安装

装完打开终端(Windows 用 CMD 或 PowerShell),敲:

```bash
go version
```

```text
go version go1.25.3 darwin/arm64
```

这是我机器(Apple 芯片的 Mac)上的真实输出。你的最后一段会不一样:Windows 是 `windows/amd64`,Intel Mac 是 `darwin/amd64`——格式是 `版本号 操作系统/CPU架构`。只要能打印出版本号,安装就成功了。

### 2.3 环境变量:大多数时候不用管

| 变量 | 说明 |
|------|------|
| GOROOT | Go 自身的安装目录 |
| GOPATH | 下载的依赖包缓存在这里 |
| GOPROXY | 依赖下载源(类比 npm registry) |

用 `go env 变量名` 可以查看,这是我机器上的:

```bash
go env GOROOT GOPATH GOPROXY
```

```text
/Users/jujiuyey/.gvm/gos/go1.25.3
/Users/jujiuyey/.gvm/pkgsets/go1.25.3/global
https://proxy.golang.org,direct
```

注意:老教程会花大篇幅讲"必须把代码放进 GOPATH"——那是 2018 年以前的规矩。现在用 Go Module(第 4 节),**项目放哪都行**,GOPATH 只是个依赖缓存目录,知道有这回事就够了。

### 2.4 常见问题

**Q: 提示 "go 不是内部或外部命令" / "command not found"?**
A: PATH 环境变量没包含 Go 的 bin 目录。安装器一般会自动加,重开一个终端窗口再试;还不行就手动把 `GOROOT/bin` 加进 PATH。

**Q: Go 版本过低?**
A: 去 https://go.dev/dl/ 下载最新版覆盖安装。本教程的代码在 1.21+ 都能跑。

---

## 3. Hello World

### 3.1 创建项目

```bash
mkdir hello-world
cd hello-world

go mod init hello-world
```

```text
go: creating new go.mod: module hello-world
```

`go mod init` 相当于 `npm init`:给项目"登记户口",生成一个 `go.mod` 文件(相当于 `package.json`,第 4 节细讲)。现在先照做。

### 3.2 编写代码

创建文件 `main.go`:

```go
package main

import "fmt"

func main() {
    fmt.Println("Hello, World!")
}
```

### 3.3 运行

```bash
go run main.go
```

```text
Hello, World!
```

`go run` 相当于 `node main.js`——但幕后多了一步:Go 先把代码**编译**成临时可执行文件,再运行它,跑完就删。所以哪怕是语法小错,`go run` 也会在编译阶段直接拦下来,程序一行都不会执行(JS 是跑到那行才炸)。

### 3.4 逐行拆解

这 7 行里有三个你没见过的东西,拆开看:

```text
package main
// ─────┬────
//  ①包声明:每个 Go 文件第一行必须声明自己属于哪个包
//    "main" 包是特殊的——告诉编译器"这是个可执行程序,不是库"

import "fmt"
// ─────┬────
//  ②导入标准库的 fmt 包(format 的缩写,负责打印/格式化)
//    类比 JS 的 import,但路径不带 ./ 也不用 npm install——标准库自带

func main() {
// ────┬──────
//  ③程序入口:main 包里的 main 函数,程序从这里开始执行
//    类比:Node 里整个文件就是入口,Go 必须显式写一个 main 函数
```

`fmt.Println(...)` 就是 Go 的 `console.log(...)`:打印内容并换行。

### 3.5 🕳️ 坑:导入了不用,直接编译失败

以为会怎样:像 JS 一样,import 了不用最多被 ESLint 黄一下。
实际怎样:编译错误,程序根本跑不起来。

```go
package main

import (
    "fmt"
    "os"       // 导入了但没用到
)

func main() {
    fmt.Println("Hello, World!")
}
```

```text
# command-line-arguments
./main.go:5:2: "os" imported and not used
```

为什么:Go 把"没用的导入"当错误而不是警告——这是语言层面强制的代码卫生。别跟它对抗,删掉没用的 import 就好(编辑器装好插件后会自动帮你删,见第 5 节)。顺便看一眼这个多行 import 的写法:导入多个包时用括号包起来,一行一个,不用逗号。

### 3.6 编译成可执行文件

`go run` 适合开发时快速跑;要交付,用 `go build` 编译出一个真正的可执行文件:

```bash
go build -o hello main.go
./hello
ls -lh hello
```

```text
Hello, World!
-rwxr-xr-x@ 1 jujiuyey  staff   2.3M Jul 16 10:06 hello
```

Windows 上把 `-o hello` 换成 `-o hello.exe`,运行用 `hello.exe`。

注意那个 2.3M:Go 把运行时和依赖**全部打包**进了这一个文件。把它拷到另一台同系统的机器上,不装 Go、不带任何依赖,直接就能跑——这就是 1.2 节说的"部署简单"。对比 Node:交付要带上源码 + node_modules + 对方还得装对版本的 Node。

| 命令 | 干什么 | 类比 |
|------|--------|------|
| `go run main.go` | 编译到临时目录并立即运行 | `node main.js` |
| `go build -o hello` | 编译出可执行文件,不运行 | 无直接对应(Node 不编译) |

### 3.7 代码规范:不用背,交给 gofmt

Go 官方直接规定了唯一的代码格式,并给了工具 `gofmt` 自动格式化。比如这段缩进乱七八糟的代码:

```go
func main() {
        x:=1
    fmt.Println( x )
}
```

对文件跑一下 `gofmt -w main.go`,变成:

```go
func main() {
	x := 1
	fmt.Println(x)
}
```

缩进统一成 Tab、`:=` 两边加空格、括号内多余空格删掉——全自动。**Go 没有"格式之争":不用配 Prettier 规则,官方格式就是唯一格式。**

只有一条规则 gofmt 救不了你,因为它是语法错误:

### 🕳️ 坑:左花括号不能另起一行

```go
func main()
{
    fmt.Println("错误")
}
```

```text
./main.go:6:1: syntax error: unexpected semicolon or newline before {
```

以为会怎样:和 C#/Java 的 Allman 风格一样,花括号换行只是风格问题。
实际怎样:编译直接报错。
为什么:Go 编译器会在某些行尾**自动补分号**,`func main()` 独占一行时,行尾被补了个分号,后面的 `{` 就成了孤儿。所以 `{` 必须跟在同一行末尾——这不是建议,是语法。

---

## 4. Go Module:Go 的 package.json

### 4.1 对照着看就懂了

Go Module 是 Go 1.11 引入的官方依赖管理机制。你在 Node 里的肌肉记忆几乎能全部平移:

| Node.js | Go | 干什么 |
|---------|-----|--------|
| `npm init` | `go mod init 模块名` | 初始化项目 |
| `package.json` | `go.mod` | 记录模块名和依赖 |
| `package-lock.json` | `go.sum` | 锁定依赖校验和 |
| `npm install xxx` | `go get xxx` | 添加依赖 |
| `node_modules/` | 全局缓存(GOPATH 下) | 依赖存放位置 |

最后一行是关键差异:**Go 没有项目内的 node_modules**,所有依赖下载到全局缓存,按版本共享——十个项目用同一个包,磁盘上只存一份。

### 4.2 go.mod 长什么样

第 3 节 `go mod init hello-world` 生成的文件就两行:

```text
module hello-world

go 1.25.3
```

`module` 是模块名(正式项目通常用仓库路径,如 `github.com/username/myproject`,这样别人才能 `go get` 到你),`go` 是语言版本。以后添加依赖,文件里会多出一个 `require` 段,格式是"包路径 + 版本号",一眼能看懂:

```text
require (
    github.com/gin-gonic/gin v1.9.1
)
```

### 4.3 缺依赖时 Go 会直接告诉你怎么办

试试导入一个第三方包但不安装,`go run` 会怎样:

```go
import (
    "fmt"

    "rsc.io/quote"   // 一个第三方示例包,还没安装
)
```

```text
main.go:6:2: no required module provides package rsc.io/quote; to add it:
	go get rsc.io/quote
```

报错信息把解决方案都写好了:跑 `go get rsc.io/quote` 就行。对比 JS 的 `Cannot find module 'xxx'`——Go 的报错通常自带"下一步该敲什么命令"。

### 4.4 常用命令

```bash
go mod tidy        # 自动补齐缺失依赖、删掉未使用依赖(最常用,一把梭)
go get xxx         # 添加依赖包
go mod download    # 下载所有依赖到本地缓存
go list -m all     # 列出所有依赖
go mod why xxx     # 解释为什么需要某个依赖
```

日常记住一个就够:改完 import 之后跑 `go mod tidy`,它会把 `go.mod` 收拾到和代码一致。

### 4.5 依赖下载加速

默认从 `proxy.golang.org` 下载(第 2.3 节 `go env GOPROXY` 查到的就是它),国内访问慢的话换镜像:

```bash
go env -w GOPROXY=https://goproxy.cn,direct
```

类比 npm 换 taobao registry,`go env -w` 是全局配置,设一次就行。

### 🕳️ 坑:没有 go.mod,`go run .` 直接罢工

以为会怎样:就跑个本目录的代码,应该不需要什么配置文件。
实际怎样:

```bash
go run .
```

```text
go: go.mod file not found in current directory or any parent directory; see 'go help modules'
```

为什么:`go run .`(跑整个目录)、`go build`、`go get` 这些命令都以模块为单位工作,必须先有 `go.mod`——就像 `npm run` 找不到 `package.json` 一样。而 `go run main.go`(指定单文件)不需要模块也能跑,所以偶尔会出现"单文件能跑、目录跑不了"的诡异现象。**养成习惯:新项目第一件事 `go mod init`。**

---

## 5. 开发工具

### 5.1 VS Code(推荐)

免费、轻量,写 Go 完全够用:

1. 下载安装 VS Code:https://code.visualstudio.com/
2. 装 Go 插件:扩展里搜 "Go"(作者 Go Team at Google)
3. 配置工具:`Cmd/Ctrl + Shift + P` → 输入 "Go: Install/Update Tools" → 全选安装

装好后保存文件自动 gofmt、自动删无用 import(第 3.5 节的坑就不会再踩了)、悬停看文档。

常用快捷键:

| 功能 | Windows | Mac |
|------|---------|-----|
| 运行程序 | F5 | F5 |
| 格式化代码 | Shift+Alt+F | Shift+Option+F |
| 快速修复 | Ctrl+. | Cmd+. |
| 跳转到定义 | F12 | F12 |

### 5.2 GoLand(JetBrains)

收费但功能全面:重构、调试、代码导航、内置版本控制。如果你用惯了 WebStorm,GoLand 就是同一套操作换了语言,适合团队项目和专业开发。

### 5.3 其他选择

| 工具 | 说明 |
|------|------|
| Vim/Neovim | 配 gopls(官方语言服务器)插件 |
| Sublime Text | 安装 GoSublime 插件 |
| LiteIDE | 专为 Go 设计的轻量 IDE |

编辑器不影响学习,选一个顺手的就往下走。

---

## 本篇重点

- [ ] Go 是编译型语言:`go run` = 编译+立即运行(类比 `node main.js`),`go build` 产出单个无依赖的可执行文件。
- [ ] 可执行程序的骨架三件套:`package main` + `import` + `func main()`,程序从 main 函数开始跑。
- [ ] 两个编译期铁律:import 了不用是错误,不是警告;左花括号 `{` 必须跟在行尾,不能另起一行。
- [ ] `go mod init` = `npm init`,`go.mod` = `package.json`;新项目第一件事就是 `go mod init`,改完依赖跑 `go mod tidy`。
- [ ] 格式不用背:`gofmt` 是唯一官方格式,让编辑器保存时自动格式化。

---

## 6. 练习作业

### 练习 1: 环境验证

```bash
go version
go env
```

确认 `go version` 能打印出版本号;浏览一遍 `go env` 的输出,找到 GOROOT、GOPATH、GOPROXY 三个变量的值。

### 练习 2: Hello World

从零走一遍完整流程:建目录 → `go mod init` → 写 `main.go` → `go run`,然后把输出内容改成你的名字。

提示:不看第 3 节,先凭记忆写;卡住了再回去对照。

### 练习 3: 加法计算器

创建 `calculator.go`,输入以下代码并跑通:

```go
package main

import "fmt"

func main() {
    a := 10
    b := 20
    sum := a + b
    fmt.Printf("%d + %d = %d\n", a, b, sum)
}
```

验收标准:输出 `10 + 20 = 30`。

提示:`:=` 是声明并赋值(下一模块细讲),`Printf` 里的 `%d` 是整数占位符,类比 JS 模板字符串里的 `${}`。

### 挑战题(选做)

编写摄氏度转华氏度的程序:

```go
package main

import "fmt"

func main() {
    var celsius float64
    fmt.Print("请输入摄氏度: ")
    fmt.Scan(&celsius)

    fahrenheit := celsius*9/5 + 32
    fmt.Printf("华氏度: %.2f\n", fahrenheit)
}
```

验收标准:输入 `36.6`,输出 `华氏度: 97.88`。

提示:`fmt.Scan(&celsius)` 从键盘读输入,`&` 取地址的含义到指针一章再讲,先照抄;试着把公式反过来,再写一个华氏转摄氏。

---

## 下一步

完成以上练习后,继续学习:
- **02**: 类型、状态与控制流

---

## 参考资料

- [Go 官方网站](https://go.dev/)
- [Go 中文文档](https://studygolang.com/)
- [Go by Example](https://gobyexample.com/)
- [Go 语言标准库](https://pkg.go.dev/)
