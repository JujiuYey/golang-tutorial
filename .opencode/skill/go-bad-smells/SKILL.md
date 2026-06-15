---
name: go-bad-smells
description: Go 代码 8 条坏味道清单。审查用户贴出的 Go 代码、或评估 AI 生成代码时自动套用。覆盖错误处理、资源、并发、context、抽象、测试。
---

# Go Bad Smells

8 条坏味道清单，源自仓库 README 第 188–196 行的 8 问。审查 Go 代码时按此扫。

## 清单（按严重度从高到低）

### 1. 忽略错误 [致命]
```go
file, _ := os.Open("x.txt")  // err 丢了
```
- 信号：`_, _ = ...`、函数返回 err 但被 `_` 吞掉
- 修复：所有 `err != nil` 路径必须有显式处理

### 2. 资源未关闭 [致命]
```go
f, _ := os.Open("x.txt")
// 缺 defer f.Close()
```
- 信号：`os.Open` / `http.Response.Body` / `sql.Rows` 后无 `defer Close()`
- 修复：`defer f.Close()` 紧跟获取；Body 必须读完

### 3. 共享状态无保护 [致命]
```go
var m map[string]int
go func() { m["a"] = 1 }()  // map 写并发不安全
```
- 信号：map / slice / struct 字段在 goroutine 间共享无锁
- 修复：`sync.Mutex` / `sync.Map` / channel 串行化

### 4. goroutine 退出条件缺失 [致命]
```go
go func() { for { work() } }()  // 永不退出
```
- 信号：`for { ... }` 内无 `ctx.Done()` / `done` channel
- 修复：`for { select { case <-ctx.Done(): return; case ...: } }`

### 5. context 未传 [建议]
```go
func DoWork() error { ... }  // 缺 ctx 参数
```
- 信号：HTTP handler / DB / 远程调用缺 ctx；超时无法生效
- 修复：第一个参数是 `ctx context.Context`，一直传到最底层

### 6. 过度抽象 [建议]
```go
type UserGetter interface { GetUser(id int) (*User, error) }  // 只有 1 个实现
```
- 信号：interface 只有 1 个实现；interface 接受具体类型而非行为
- 修复：先写具体代码，第 2 个实现出现时再抽象

### 7. 测试仅 happy path [建议]
```go
func TestAdd(t *testing.T) { if Add(1, 1) != 2 { t.Fatal() } }
```
- 信号：无错误路径、无 nil 边界、无并发竞争覆盖
- 修复：表驱动测试 + `t.Run("error case", ...)` + 并发场景用 `-race`

### 8. 命名 / 可读性 [风格]
- 单字母变量滥用（除 `i`、`j` 循环）
- 错误吞到 `panic("xxx")` 替代 `return err`
- 魔法数字（应提为 const）

## 使用方式

- 用户贴代码 → 主动扫这 8 条 → 输出 `[严重度] 位置 + 一行修复`
- 用户问"这段有没有问题" → 同样扫
- 不要替代码辩护，不要"先夸一句再说问题"
