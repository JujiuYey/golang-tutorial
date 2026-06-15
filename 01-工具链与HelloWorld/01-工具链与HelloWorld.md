# 01: 工具链与 Hello World

> 预计学习时长: 2小时  
> 目标: 掌握Go环境配置，能够编写并运行第一个Go程序

---

## 目录

1. [Go语言简介](#1-go语言简介)
2. [开发环境搭建](#2-开发环境搭建)
3. [Hello World](#3-hello-world)
4. [Go Module](#4-go-module)
5. [开发工具](#5-开发工具)
6. [练习作业](#6-练习作业)

---

## 1. Go语言简介

### 1.1 什么是Go语言？

Go（又称Golang）是Google于2009年发布的开源编程语言，由Robert Griesemer、Rob Pike和Ken Thompson设计。

### 1.2 Go的核心特点

| 特点 | 说明 |
|------|------|
| **简洁高效** | 语法简单，关键字少（仅25个），学习曲线平缓 |
| **高性能** | 编译型语言，接近C的性能，执行效率高 |
| **并发原生** | Goroutine + Channel，天然支持高并发 |
| **标准库丰富** | 网络、加密、IO、测试等内置支持 |
| **部署简单** | 编译成单个可执行文件，无依赖 |
| **静态类型** | 类型安全，编译时检查 |

### 1.3 Go能做什么？

- ✅ Web服务器/API开发
- ✅ 微服务架构
- ✅ 云原生应用（Kubernetes/Docker用Go编写）
- ✅ 网络工具和代理
- ✅ 数据库和缓存系统
- ✅ CLI工具开发

### 1.4 Go vs 其他语言

```
对比维度          | Go        | Python    | Java      | Node.js
----------------|-----------|-----------|-----------|----------
学习曲线          | ★★★☆☆    | ★☆☆☆☆    | ★★★★☆    | ★★☆☆☆
运行速度          | ★★★★★    | ★★☆☆☆    | ★★★★☆    | ★★★☆☆
并发支持          | ★★★★★    | ★★☆☆☆    | ★★★★☆    | ★★★☆☆
开发效率          | ★★★★☆    | ★★★★★    | ★★★☆☆    | ★★★★☆
生态丰富度        | ★★★★☆    | ★★★★★    | ★★★★★    | ★★★★☆
```

---

## 2. 开发环境搭建

### 2.1 Windows 安装

#### 步骤1: 下载安装包
访问 https://go.dev/dl/ 下载 Windows 安装包（`.msi` 文件）

#### 步骤2: 安装
双击运行，一路点击 "Next" 即可

#### 步骤3: 验证安装
打开命令提示符（CMD）或PowerShell，输入：

```bash
go version
```

输出类似：
```
go version go1.22.0 windows/amd64
```

### 2.2 Mac 安装

#### 使用 Homebrew（推荐）
```bash
brew install go
```

#### 手动安装
访问 https://go.dev/dl/ 下载 macOS 安装包

#### 验证
```bash
go version
```

### 2.3 Linux 安装

#### 使用包管理器
```bash
# Ubuntu/Debian
sudo apt-get update
sudo apt-get install golang-go

# CentOS/RHEL
sudo yum install golang
```

#### 验证
```bash
go version
```

### 2.4 环境变量说明

| 变量 | 说明 | 默认值(Windows) |
|------|------|----------------|
| GOROOT | Go安装目录 | C:\Program Files\Go |
| GOPATH | 工作目录 | %USERPROFILE%\go |
| PATH | 添加Go bin | %GOROOT%\bin |

### 2.5 常见问题

**Q: 提示 "go 不是内部或外部命令"？**  
A: 检查PATH环境变量是否包含Go的bin目录

**Q: Go版本过低？**  
A: 下载最新版 https://go.dev/dl/

---

## 3. Hello World

### 3.1 创建项目

```bash
# 创建项目目录
mkdir hello-world
cd hello-world

# 初始化Go模块（后面会详细讲）
go mod init hello-world
```

### 3.2 编写代码

创建文件 `main.go`：

```go
package main

import "fmt"

func main() {
    fmt.Println("Hello, World!")
}
```

### 3.3 代码解释

```go
package main          // 1. 包声明，每个Go文件都属于一个包

import "fmt"          // 2. 导入语句，引入fmt包用于打印输出

func main() {         // 3. main函数，程序入口
    fmt.Println("Hello, World!")  // 4. 调用fmt包的Println函数打印
}                     // 5. 函数体用花括号包裹
```

### 3.4 运行程序

```bash
# 方式1: 直接运行（边编译边运行，临时文件）
go run main.go
# 输出: Hello, World!

# 方式2: 编译成可执行文件
go build -o hello.exe main.go
./hello.exe
# 输出: Hello, World!
```

### 3.5 Go代码规范

1. **花括号** `{` 不能单独一行
2. **缩进** 使用Tab，不是空格
3. **包名** 与文件夹名通常一致
4. **文件名** 全部小写，用下划线分隔

```go
// ✅ 正确
func main() {
    fmt.Println("正确")
}

// ❌ 错误
func main()
{
    fmt.Println("错误")
}
```

---

## 4. Go Module

### 4.1 什么是Go Module？

Go Module 是Go 1.11引入的依赖管理工具，用于管理项目的依赖包。

### 4.2 初始化项目

```bash
# 在项目目录下执行
go mod init 模块名
```

### 4.3 go.mod 文件

```go
module github.com/username/myproject    // 模块名（通常用仓库路径）

go 1.22                                 // Go版本

require (                                // 依赖包
    github.com/gin-gonic/gin v1.9.1
    github.com/go-sql-driver/mysql v1.7.1
)
```

### 4.4 常用命令

```bash
go mod tidy        # 自动添加缺失的依赖，删除未使用的依赖
go get xxx         # 添加依赖包
go mod download    # 下载所有依赖到本地
go list -m all     # 列出所有依赖
go mod why xxx     # 解释为什么需要某个依赖
```

### 4.5 依赖下载加速

Go模块默认从 proxy.golang.org 下载，可配置国内镜像：

```bash
# 设置GOPROXY
go env -w GOPROXY=https://goproxy.cn,direct
```

---

## 5. 开发工具

### 5.1 VS Code（推荐）

**优点**: 免费、轻量、插件丰富

**安装步骤**:
1. 下载安装 VS Code: https://code.visualstudio.com/
2. 安装 Go 插件: 搜索 "Go" (作者: Go Team at Google)
3. 配置工具: `Cmd/Ctrl + Shift + P` → 输入 "Go: Install/Update Tools"
4. 全选所有工具，点击确定

**常用快捷键**:
| 功能 | Windows | Mac |
|------|---------|-----|
| 运行程序 | F5 | F5 |
| 格式化代码 | Shift+Alt+F | Shift+Option+F |
| 快速修复 | Ctrl+. | Cmd+. |
| 跳转到定义 | F12 | F12 |

### 5.2 GoLand（JetBrains）

**优点**: 功能全面，智能提示强大

**特点**:
- 重构、调试、代码导航
- 内置版本控制
- 专业项目管理

**适用场景**: 团队项目、专业开发

### 5.3 其他工具

| 工具 | 说明 |
|------|------|
| Vim/Neovim | 使用 gomodetags 等插件 |
| Sublime Text | 安装 GoSublime 插件 |
| LiteIDE | 专为Go设计的轻量IDE |

---

## 6. 练习作业

### 练习1: 环境验证
```bash
# 验证Go安装
go version

# 查看环境配置
go env
```

### 练习2: Hello World
```bash
# 创建并运行Hello World程序
# 修改输出内容为你的名字
```

### 练习3: 加法计算器
创建 `calculator.go`：

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

### 挑战题（选做）
编写摄氏度和华氏度转换程序：

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

---

## 下一步

完成以上练习后，继续学习：
- **02**: 类型、状态与控制流

---

## 参考资料

- [Go官方网站](https://go.dev/)
- [Go中文文档](https://studygolang.com/)
- [Go by Example](https://gobyexample.com/)
- [Go语言标准库](https://pkg.go.dev/)

---

*祝学习愉快！*
