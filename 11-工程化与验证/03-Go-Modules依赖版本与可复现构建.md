# Go Modules、依赖版本与可复现构建

Go Modules 负责描述模块路径、Go 版本和依赖版本。工程化里很重要的一件事是：同一份代码在不同机器上能构建出一致的结果。

---

## 1. go.mod 的基本结构

```go
module example.com/myapp

go 1.24

require (
	github.com/google/uuid v1.6.0
)
```

字段含义：

| 字段 | 含义 |
| --- | --- |
| `module` | 当前模块路径 |
| `go` | 这个模块声明的 Go 语言版本 |
| `require` | 直接或间接依赖 |

模块路径决定别人如何导入你的包。应用项目也应该有清楚的 module path。

---

## 2. go.sum 的作用

`go.sum` 保存依赖内容的校验信息。它不是锁文件的完全等价物，但对可复现构建非常重要。

不要手动编辑 `go.sum`。使用 Go 命令维护它：

```bash
go mod tidy
go test ./...
```

如果依赖内容被篡改，Go 命令会通过校验失败提醒你。

---

## 3. 添加依赖

添加依赖：

```bash
go get github.com/google/uuid@v1.6.0
```

使用依赖：

```go
package idgen

import "github.com/google/uuid"

func New() string {
	return uuid.NewString()
}
```

随后运行：

```bash
go mod tidy
go test ./...
```

`tidy` 会删除未使用依赖，补齐缺失依赖。

---

## 4. 升级和降级依赖

升级某个依赖：

```bash
go get github.com/google/uuid@latest
go mod tidy
go test ./...
```

降级到指定版本：

```bash
go get github.com/google/uuid@v1.5.0
go mod tidy
go test ./...
```

升级依赖后一定要跑测试。依赖版本变动本身就是一次代码变更。

---

## 5. 查看依赖图

```bash
go list -m all
```

查看某个包来自哪里：

```bash
go list -deps ./... | sort
```

解释为什么引入某个模块：

```bash
go mod why github.com/google/uuid
```

这些命令能帮你发现项目是否引入了意外依赖。

---

## 6. 验证依赖

```bash
go mod verify
```

它会检查模块缓存中的依赖是否与下载时记录的校验值一致。

CI 中可以加入：

```bash
go mod download
go mod verify
go test ./...
```

这样 CI 在测试前就能发现依赖下载和校验问题。

---

## 7. vendor 模式

生成 vendor：

```bash
go mod vendor
```

使用 vendor 构建：

```bash
go build -mod=vendor ./...
```

vendor 会把依赖源码放进项目目录。它适合对构建环境有强控制需求的团队。普通项目通常先使用模块缓存和 `go.sum`。

---

## 8. 依赖管理原则

1. 只引入真正需要的依赖。
2. 优先选择维护活跃、API 稳定、文档清楚的库。
3. 小功能不要轻易引入大型依赖。
4. 升级依赖要有测试保护。
5. 提交 `go.mod` 和 `go.sum`。
6. CI 中运行 `go mod tidy` 检查是否干净。

---

## 9. 学习检查

检查 Go Modules 时，重点看：

1. module path 是否合理。
2. `go.mod` 是否只包含必要依赖。
3. `go.sum` 是否提交。
4. `go mod tidy` 后是否还有 diff。
5. `go mod verify` 是否通过。
6. 依赖升级是否触发完整测试。
7. 是否存在用不到的大型依赖。
