# Makefile 与本地任务编排

Makefile 的价值是统一命令入口。项目里常见的格式化、测试、race、构建、清理，都可以放进 Makefile。

---

## 1. 最小 Makefile

Makefile 的命令行必须用 tab 缩进。

```makefile
.PHONY: fmt vet test build check

fmt:
	go fmt ./...

vet:
	go vet ./...

test:
	go test ./...

build:
	go build ./...

check: fmt vet test build
```

运行：

```bash
make check
```

---

## 2. 变量

```makefile
BINARY_NAME := myapp
BUILD_DIR := build
MAIN_PACKAGE := ./cmd/myapp

.PHONY: build
build:
	mkdir -p $(BUILD_DIR)
	go build -o $(BUILD_DIR)/$(BINARY_NAME) $(MAIN_PACKAGE)
```

变量让命令更容易调整。

---

## 3. 加入 race 和覆盖率

```makefile
.PHONY: race cover

race:
	go test -race ./...

cover:
	go test -coverprofile=coverage.out ./...
	go tool cover -html=coverage.out -o coverage.html
```

`cover` 会生成 HTML 覆盖率报告。

---

## 4. 检查 go mod tidy

CI 中常见检查：

```makefile
.PHONY: tidy-check

tidy-check:
	go mod tidy
	git diff --exit-code go.mod go.sum
```

如果有人忘了提交依赖变化，这个命令会失败。

---

## 5. 构建版本信息

```makefile
VERSION ?= dev
COMMIT := $(shell git rev-parse --short HEAD)
BUILD_TIME := $(shell date -u '+%Y-%m-%dT%H:%M:%SZ')
LDFLAGS := -ldflags "-X main.version=$(VERSION) -X main.commit=$(COMMIT) -X main.buildTime=$(BUILD_TIME)"

.PHONY: build
build:
	mkdir -p $(BUILD_DIR)
	go build $(LDFLAGS) -o $(BUILD_DIR)/$(BINARY_NAME) $(MAIN_PACKAGE)
```

应用可以提供 `myapp version` 或启动日志输出这些信息。

---

## 6. 帮助信息

```makefile
.PHONY: help

help:
	@echo "targets:"
	@echo "  fmt          format go files"
	@echo "  vet          run go vet"
	@echo "  test         run tests"
	@echo "  race         run tests with race detector"
	@echo "  cover        generate coverage report"
	@echo "  build        build binary"
	@echo "  check        run standard verification"
```

新人进项目后，先跑 `make help` 就能看到常用命令。

---

## 7. 完整示例

```makefile
.PHONY: help fmt vet test race cover tidy-check build clean check

BINARY_NAME := myapp
BUILD_DIR := build
MAIN_PACKAGE := ./cmd/myapp
VERSION ?= dev
COMMIT := $(shell git rev-parse --short HEAD)
BUILD_TIME := $(shell date -u '+%Y-%m-%dT%H:%M:%SZ')
LDFLAGS := -ldflags "-X main.version=$(VERSION) -X main.commit=$(COMMIT) -X main.buildTime=$(BUILD_TIME)"

help:
	@echo "make fmt | vet | test | race | cover | build | check"

fmt:
	go fmt ./...

vet:
	go vet ./...

test:
	go test ./...

race:
	go test -race ./...

cover:
	go test -coverprofile=coverage.out ./...
	go tool cover -html=coverage.out -o coverage.html

tidy-check:
	go mod tidy
	git diff --exit-code go.mod go.sum

build:
	mkdir -p $(BUILD_DIR)
	go build $(LDFLAGS) -o $(BUILD_DIR)/$(BINARY_NAME) $(MAIN_PACKAGE)

clean:
	rm -rf $(BUILD_DIR) coverage.out coverage.html

check: fmt vet tidy-check test race build
```

---

## 8. 学习检查

检查 Makefile 时，重点看：

1. 常用命令是否集中。
2. target 是否声明 `.PHONY`。
3. 命令是否能在干净机器上运行。
4. `check` 是否覆盖关键验证。
5. 构建产物是否进入 `build` 目录。
6. 版本信息是否能注入。
7. README 是否说明如何使用 Makefile。
