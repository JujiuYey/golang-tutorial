# 项目结构、包边界与 internal

Go 项目结构的重点是包边界。目录只是包边界的外在表现。

---

## 1. 一个可扩展的基础结构

一个 CLI 或 Web 服务可以从这个结构开始：

```text
myapp/
  go.mod
  cmd/
    myapp/
      main.go
  internal/
    config/
      config.go
    app/
      app.go
    task/
      model.go
      service.go
      repository.go
      handler.go
  README.md
  Makefile
```

含义：

| 目录 | 职责 |
| --- | --- |
| `cmd/myapp` | 应用入口，负责组装依赖和启动 |
| `internal/config` | 配置读取和校验 |
| `internal/app` | 应用级组装，例如路由、生命周期 |
| `internal/task` | 某个业务域 |
| `Makefile` | 常用命令入口 |

`cmd` 下可以有多个入口，例如 `server`、`worker`、`migrate`。

---

## 2. internal 的作用

`internal` 是 Go 工具链支持的特殊目录。外部模块不能导入它下面的包。

```text
myapp/
  internal/task
other-project/
  main.go
```

`other-project` 无法导入：

```go
import "example.com/myapp/internal/task"
```

这能防止业务内部包被外部项目误用。对于应用项目，大部分代码放在 `internal` 里是合理的。

---

## 3. pkg 要谨慎使用

`pkg` 通常表示可以被外部项目复用的公共包：

```text
pkg/
  authclient/
  retry/
```

如果代码只服务当前应用，就先放在 `internal`。过早放入 `pkg` 会给维护带来额外承诺：命名、API、兼容性都要更慎重。

---

## 4. 按领域聚合

推荐把一个业务功能相关的代码放近一些：

```text
internal/task/
  model.go
  service.go
  repository.go
  handler.go
  service_test.go
```

这种结构读起来顺着业务走。你要改 task 功能，大多数文件都在同一个目录里。

技术分层目录也能用：

```text
internal/
  handler/
  service/
  repository/
  model/
```

小项目里它还可以。功能变多后，相关代码会散在多个目录中，阅读成本会上升。

---

## 5. 包名要短而具体

好包名：

- `config`
- `task`
- `auth`
- `httpx`
- `storage`

容易变乱的包名：

- `utils`
- `common`
- `helper`
- `manager`

如果一个包只能叫 `utils`，通常说明职责还没想清楚。可以按能力命名，例如 `idgen`、`clock`、`httperror`。

---

## 6. main 包保持薄

入口文件只负责组装：

```go
package main

import (
	"log"

	"example.com/myapp/internal/app"
	"example.com/myapp/internal/config"
)

func main() {
	if err := run(); err != nil {
		log.Fatal(err)
	}
}

func run() error {
	cfg, err := config.Load()
	if err != nil {
		return err
	}

	application, err := app.New(cfg)
	if err != nil {
		return err
	}

	return application.Run()
}
```

这样 `run` 可以被测试，启动流程也容易阅读。

---

## 7. 避免包之间循环依赖

循环依赖通常表示边界不清：

```text
task -> user -> task
```

常见解决方式：

1. 抽出更小的类型到独立包。
2. 把接口放在使用方。
3. 让依赖方向统一，例如 handler 依赖 service，service 依赖 repository。
4. 合并两个本来就不可分的包。

Go 不允许循环 import，这会逼你更早面对边界问题。

---

## 8. 学习检查

检查项目结构时，重点看：

1. 入口是否在 `cmd` 下。
2. 应用内部代码是否放在 `internal`。
3. 业务相关文件是否靠近。
4. `pkg` 下是否真的是公共 API。
5. 包名是否表达职责。
6. `main` 是否足够薄。
7. 是否存在循环依赖迹象。
