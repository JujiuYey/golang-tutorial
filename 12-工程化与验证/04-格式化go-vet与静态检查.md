# 格式化、go vet 与静态检查

Go 的工程化从统一格式开始。格式统一后，代码评审才会集中到行为、边界和设计上。

---

## 1. gofmt

格式化单个文件：

```bash
gofmt -w main.go
```

检查文件格式差异：

```bash
gofmt -d main.go
```

格式化整个项目常用：

```bash
go fmt ./...
```

`go fmt` 会对包运行格式化。它适合日常使用。`gofmt` 更适合脚本中处理具体文件。

---

## 2. gofmt 会做什么

原始代码：

```go
func Add(a int,b int)int{
return a+b
}
```

格式化后：

```go
func Add(a int, b int) int {
	return a + b
}
```

Go 的风格由工具决定。你不需要在缩进、空格、括号位置上花时间争论。

---

## 3. go vet

`go vet` 检查可疑代码：

```bash
go vet ./...
```

常见能发现的问题：

- `fmt.Printf` 参数和占位符不匹配。
- 复制了包含锁的值。
- unreachable code。
- 错误使用 struct tag。
- 测试里的错误格式。

例子：

```go
package main

import "fmt"

func main() {
	fmt.Printf("count=%d\n", "three")
}
```

`go vet` 会指出 `%d` 需要数字，实际传入字符串。

---

## 4. 静态检查工具

`go vet` 是标准工具，覆盖面有限。项目可以引入更强的静态检查工具，例如 `staticcheck`。

安装：

```bash
go install honnef.co/go/tools/cmd/staticcheck@latest
```

运行：

```bash
staticcheck ./...
```

引入第三方工具时，要把版本固定到团队文档或 CI 配置中，避免不同机器得到不同结果。

---

## 5. 格式检查脚本

CI 中可以检查格式是否干净：

```bash
gofmt -w .
git diff --exit-code
```

更精确的写法是只处理 Go 文件：

```bash
files=$(find . -name '*.go' -not -path './vendor/*')
gofmt -w $files
git diff --exit-code
```

如果 CI 里 `git diff --exit-code` 失败，说明有人提交了未格式化代码。

---

## 6. gofmt 和 gofumpt

`gofumpt` 是更严格的格式化工具。普通项目先掌握标准 `gofmt` 和 `go fmt`。团队真的需要更严格规则时，再统一引入 `gofumpt`。

关键不是工具越多越好，关键是团队使用同一套规则，并且 CI 能重复检查。

---

## 7. 常用本地检查组合

```bash
go fmt ./...
go vet ./...
go test ./...
```

如果项目引入了 staticcheck：

```bash
staticcheck ./...
```

这组命令应该成为提交前的肌肉记忆。

---

## 8. 学习检查

检查格式和静态分析时，重点看：

1. 是否使用 `go fmt ./...`。
2. CI 是否检查格式 diff。
3. 是否运行 `go vet ./...`。
4. 是否理解 vet 和测试的区别。
5. 第三方静态检查工具是否固定版本。
6. 工具失败时是否能定位到具体文件和行。
