# 构建、版本信息与 Docker 镜像

工程化不止测试通过，还要能产出可运行、可追踪的构建产物。

---

## 1. 构建主程序

常见项目入口：

```text
cmd/myapp/main.go
```

构建：

```bash
go build -o build/myapp ./cmd/myapp
```

如果有多个入口：

```bash
go build -o build/server ./cmd/server
go build -o build/worker ./cmd/worker
```

---

## 2. 注入版本信息

代码：

```go
package main

import "fmt"

var (
	version   = "dev"
	commit    = "unknown"
	buildTime = "unknown"
)

func printVersion() {
	fmt.Printf("version=%s commit=%s build_time=%s\n", version, commit, buildTime)
}
```

构建：

```bash
VERSION=v1.2.3
COMMIT=$(git rev-parse --short HEAD)
BUILD_TIME=$(date -u '+%Y-%m-%dT%H:%M:%SZ')

go build \
  -ldflags "-X main.version=$VERSION -X main.commit=$COMMIT -X main.buildTime=$BUILD_TIME" \
  -o build/myapp \
  ./cmd/myapp
```

版本信息能帮助你确认线上跑的是哪一次构建。

---

## 3. 交叉编译

构建 Linux 版本：

```bash
GOOS=linux GOARCH=amd64 go build -o build/myapp-linux-amd64 ./cmd/myapp
```

构建 macOS ARM64：

```bash
GOOS=darwin GOARCH=arm64 go build -o build/myapp-darwin-arm64 ./cmd/myapp
```

查看支持的平台：

```bash
go tool dist list
```

---

## 4. CGO

纯 Go 程序常用：

```bash
CGO_ENABLED=0 go build -o build/myapp ./cmd/myapp
```

如果项目依赖 SQLite、某些系统库或 C 库，可能需要开启 CGO：

```bash
CGO_ENABLED=1 go build -o build/myapp ./cmd/myapp
```

CGO 会影响交叉编译和镜像基础环境，构建前要确认依赖。

---

## 5. 多阶段 Dockerfile

```dockerfile
ARG GO_VERSION=1.24

FROM golang:${GO_VERSION}-alpine AS build
WORKDIR /src

COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 go build -o /out/myapp ./cmd/myapp

FROM alpine:3.20
RUN apk --no-cache add ca-certificates
COPY --from=build /out/myapp /usr/local/bin/myapp

EXPOSE 8080
ENTRYPOINT ["myapp"]
```

`GO_VERSION` 要和项目 `go.mod` 中的版本保持一致。

---

## 6. 更小的运行镜像

纯静态二进制可以使用 distroless 或 scratch。scratch 示例：

```dockerfile
FROM scratch
COPY --from=build /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=build /out/myapp /myapp
ENTRYPOINT ["/myapp"]
```

使用 scratch 时要确认应用不依赖 shell、时区文件、系统证书之外的运行时文件。

---

## 7. 构建镜像

```bash
docker build -t myapp:dev .
```

运行：

```bash
docker run --rm -p 8080:8080 myapp:dev
```

验证：

```bash
curl -i http://localhost:8080/healthz
```

镜像构建完成后，要验证容器真的能启动并响应请求。

---

## 8. 学习检查

检查构建和镜像时，重点看：

1. 主程序入口是否明确。
2. 构建命令是否写入 Makefile。
3. 构建产物是否带版本、commit、build time。
4. 是否理解 CGO 对构建的影响。
5. Dockerfile 是否使用多阶段构建。
6. 镜像启动后是否用 `curl` 验证。
7. 基础镜像版本是否有维护策略。
