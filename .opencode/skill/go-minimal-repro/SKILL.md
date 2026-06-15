---
name: go-minimal-repro
description: Go 最小可复现示例约束。涉及 Go 性能、并发、错误处理、内存、资源、context 行为时强制要求：≤ 20 行可 go run 代码 + go test 或 go test -race 验证。用于"我怀疑 X"、"证明 Y"、"为什么 Z"。
---

# Go Minimal Reproduction

任何关于 Go 行为、性能、并发、资源的判断，**必须**配最小可复现示例 + 验证命令。

## 触发场景

- 用户问"为什么 channel 会阻塞"、"slice 到底共享了什么"、"这段有没有 race"
- 用户贴一段代码说"这里有问题吗"
- 用户想证明某个 Go 机制的行为

## 最小复现规则

- 代码 ≤ 20 行（含 import）
- 单文件，可直接 `go run file.go`
- 命名贴主题：`race_demo.go` / `channel_block.go` / `slice_share.go`
- 故意暴露问题点（不要写"看起来正确"的代码）

## 必跑的验证命令

按行为类型二选一：

- **并发 / 共享状态 / 内存**：`go run -race file.go` 或 `go test -race ./...`
- **API / 标准库行为**：`go test -run TestX` + 看输出
- **静态检查**：`go vet ./...`

## 反例（不要这么示范）

```go
// 80 行的 web server 演示 context 取消
```

## 正例

```go
// race_demo.go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var x int
	var wg sync.WaitGroup
	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go func() { x++; wg.Done() }()
	}
	wg.Wait()
	fmt.Println(x)
}
```

跑法：`go run -race race_demo.go`

## 收尾

示范完代码后，**强制**要求用户：
1. 自己跑一遍命令
2. 贴回输出
3. 用一句话解释输出为什么是这样
