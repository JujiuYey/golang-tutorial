# Go 诊断式陪练教程

这是一套用来系统学习 Go 的教程仓库。它不只整理知识点，更强调一种学习方式：先暴露当前理解，再围绕一个具体卡点解释、练习和验证。

仓库内置 opencode 陪练配置。陪练不会一次抛出一串问题，也不会直接替你完成项目。它会先读取你的学习画像，然后一次只问一个诊断问题，根据你的回答判断你现在处在哪一层：没见过、见过但说不清、能说但不会验证、能验证但还不稳。

目标不是“看完 Go 语法”，而是逐步做到：

1. 能用自己的话讲清 Go 核心概念。
2. 能写最小代码验证一个机制。
3. 能看懂运行结果、测试输出和错误信息。
4. 能把小知识点组合到 CLI、HTTP API、数据库和 Web 服务里。
5. 能通过学习日志看见自己的进度和薄弱点。

---

## 仓库结构

| 路径 | 作用 |
|------|------|
| `01-工具链与HelloWorld/` 到 `11-工程化与验证/` | Go 主线课程模块 |
| `project-01-CLI任务管理器/` | 综合项目 01：CLI 任务管理器 |
| `project-02-FiberRESTAPI/` | 综合项目 02：Fiber REST API |
| `.opencode/agent/go-feynman-tutor.md` | 诊断式 Go 陪练 agent |
| `.opencode/skill/go-feynman-4-questions/` | 单问式费曼自检 skill |
| `learning-log/` | 学习画像和每日学习日志 |
| `AGENTS.md` | 本仓库 agent 行为守则 |

---

## 课程模块

| 模块 | 内容 | 训练重点 |
|------|------|----------|
| [01](./01-工具链与HelloWorld/01-工具链与HelloWorld.md) | 工具链与 Hello World | 安装 Go、运行第一个程序、理解 Go Module |
| [02](./02-类型状态与控制流/README.md) | 类型、状态与控制流 | 值、变量、常量、表达式、分支、循环 |
| [03](./03-函数错误与边界/README.md) | 函数、错误与边界 | 函数组织、错误返回、defer、panic 边界 |
| [04](./04-数据结构与内存语义/README.md) | 数据结构与内存语义 | array、slice、map、struct、pointer、range 语义 |
| [05](./05-接口组合与抽象/README.md) | 接口、组合与泛型 | 小接口、组合、泛型、抽象选择 |
| [06](./06-并发模型/README.md) | 并发模型 | goroutine、channel、select、context、锁、race |
| [07](./07-标准库/README.md) | 标准库 | IO、文件、JSON、时间、flag、slog、testing |
| [08](./08-网络编程/README.md) | 网络编程 | HTTP 服务端、客户端、JSON API、TCP/UDP、httptest |
| [09](./09-数据库与持久化/README.md) | 数据库与持久化 | database/sql、连接池、事务、SQL 参数、测试 |
| [10](./10-Web服务/README.md) | Web 服务 | 分层结构、handler、service、repository、中间件、认证 |
| [11](./11-工程化与验证/README.md) | 工程化与验证 | go mod、gofmt、vet、测试、race、benchmark、CI |
| [Project 01](./project-01-CLI任务管理器/project-01-CLI任务管理器.md) | CLI 任务管理器 | 命令行、文件持久化、结构拆分、本地验证 |
| [Project 02](./project-02-FiberRESTAPI/project-02-FiberRESTAPI.md) | Fiber REST API + Bun | Web API、认证、数据库、服务分层、综合练习 |

---

## 推荐学习循环

每篇文章都按一个很小的闭环推进：

1. **读**：先读当前文章，不急着记完整。
2. **讲**：合上文章，用自己的话说一个概念。
3. **诊断**：让 agent 只问你一个问题，定位卡点。
4. **写**：围绕这个卡点写一个最小 Go 例子。
5. **测**：用命令验证，而不是只凭感觉判断。
6. **记**：小节结束后，把学习事实写进 `learning-log/`。

常用验证命令：

```bash
go run .
go test ./...
go vet ./...
go test -race ./...
```

网络和 Web 服务相关内容，可以继续用 `curl`、日志和 `httptest` 验证请求、响应、错误路径和超时行为。

---

## 怎么和 agent 一起学

本仓库的 agent 不是答题机，而是诊断式陪练。

### 开始学习时

你可以这样说：

```text
我想学 04 的 slice，先诊断一下我哪里不懂
```

或：

```text
我刚读完 03-函数错误与边界/04-defer.md，陪我过一遍
```

agent 应该先读取 `learning-log/profile.md`，再只问你一个诊断问题。

### 回答问题时

你不用一次回答“是什么、解决什么问题、哪里易错、怎么验证”。每轮只处理一个点。agent 会根据你的回答决定下一步：

- 如果你说得模糊，它继续追问一个更小的问题。
- 如果你说得部分正确，它只补一个关键缺口。
- 如果你说得比较稳，它给一个最小例子或验证命令。

### 小节结束时

agent 会先给出学习日志草稿，例如：

```md
## 14:30 - slice append 行为

- 学习内容：slice 底层数组共享与 append 重新分配。
- 用户当前理解：能说出 slice 是描述符，但对容量变化后的共享关系不稳。
- 卡点或易错点：误以为 append 后一定仍共享原数组。
- 已验证命令：go run slice_append.go
- 下一步：复习 len/cap 与底层数组关系。
```

你确认后，agent 才会写入 `learning-log/daily/YYYY-MM-DD.md`，必要时更新 `learning-log/profile.md`。

---

## learning-log 怎么用

`learning-log/` 是长期学习记忆。

| 文件 | 用途 |
|------|------|
| [learning-log/profile.md](./learning-log/profile.md) | 当前学习画像：目标、偏好、水平、卡点、下一步 |
| [learning-log/daily/](./learning-log/daily/) | 每日学习记录 |
| [learning-log/README.md](./learning-log/README.md) | 日志格式和写入规则 |

它解决两个问题：

1. agent 每次开始时不用从零猜你的水平。
2. 你可以回看自己学过什么、卡过什么、下一步该复习什么。

日志只记录学习事实，不把“看过”当成“掌握”。是否掌握，要通过你的解释、代码和验证命令来判断。

---

## 学习节奏建议

### 第 1-2 周：工具链与基础语法

学习 01、02。重点掌握 Go 环境、`go run`、`go test`、变量、常量、类型转换、字符串、条件和循环。

复习目标：能解释零值、作用域、变量遮蔽、`range` 的基本行为。

### 第 3-4 周：函数、错误、数据结构

学习 03、04。重点掌握函数签名、多返回值、错误返回、`defer`、slice、map、struct 和 pointer。

复习目标：能判断一段代码复制了什么、共享了什么、错误有没有被正确处理。

### 第 5 周：接口、组合与泛型

学习 05。重点掌握小接口、隐式实现、接口值 nil、组合、泛型函数和类型约束。

复习目标：能判断什么时候先写具体代码，什么时候再抽象接口或泛型。

### 第 6 周：并发模型

学习 06。重点掌握 goroutine 生命周期、channel 关闭、`select`、`context`、锁和 race detector。

复习目标：能解释谁退出、谁关闭、谁取消，以及共享状态是否被保护。

### 第 7 周：标准库

学习 07。重点掌握文件 IO、JSON、时间、命令行参数、结构化日志和测试工具。

复习目标：遇到常见任务时，能先想到标准库和对应验证方式。

### 第 8-9 周：网络、数据库与 Web 服务

学习 08、09、10。重点掌握 HTTP handler、client timeout、JSON API、数据库连接池、事务、repository 和 Web 服务分层。

复习目标：能用 `httptest`、`curl`、日志和测试验证 API 行为。

### 第 10 周：工程化与验证

学习 11。重点掌握 `gofmt`、`go vet`、表驱动测试、覆盖率、race detector、benchmark 和 CI。

复习目标：能用工具链证明代码质量，而不是只靠口头判断。

### 第 11-12 周：综合项目

完成 Project 01 和 Project 02。项目要由你主导拆解，agent 只做单点辅导。

推荐节奏：

1. 需求拆解
2. 最小实现
3. 测试验证
4. 重构整理
5. 写入学习日志

---

## 陪练边界

这个仓库的 agent 应该做：

- 先诊断，再解释。
- 一次只推进一个认知点。
- 给最小例子，而不是长篇代码。
- 要求用命令验证关键判断。
- 小节结束后生成学习日志草稿。

这个仓库的 agent 不应该做：

- 一次问多个问题。
- 直接替你完成综合项目。
- 生成大段脚手架。
- 没确认就写学习日志。
- 把看过、听过、复制过当成已经掌握。

---

## 下一步

从任意一篇文章开始，然后对 agent 说：

```text
我准备学这一篇，先用一个问题诊断我现在懂到哪
```

学完一个小节后，让 agent 生成学习日志草稿。确认无误后，把它写入 `learning-log/daily/`。
