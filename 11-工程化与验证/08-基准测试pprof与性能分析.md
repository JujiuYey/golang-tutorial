# 基准测试、pprof 与性能分析

性能优化要先测量。Go 提供了内置基准测试和 `pprof`，足够支撑大多数性能定位。

---

## 1. 编写基准测试

```go
func BenchmarkReverse(b *testing.B) {
	for i := 0; i < b.N; i++ {
		Reverse("hello world")
	}
}
```

运行：

```bash
go test -bench=. ./...
```

带内存分配：

```bash
go test -bench=. -benchmem ./...
```

`b.N` 由测试框架自动调整。不要自己写固定循环次数。

---

## 2. ResetTimer

如果基准测试里有准备数据，要在正式计时前重置计时器：

```go
func BenchmarkEncodeTasks(b *testing.B) {
	tasks := make([]Task, 1000)
	for i := range tasks {
		tasks[i] = Task{ID: strconv.Itoa(i), Title: "learn go"}
	}

	b.ResetTimer()

	for i := 0; i < b.N; i++ {
		var buf bytes.Buffer
		if err := json.NewEncoder(&buf).Encode(tasks); err != nil {
			b.Fatalf("encode: %v", err)
		}
	}
}
```

准备数据的时间不应该算进被测函数。

---

## 3. ReportAllocs

```go
func BenchmarkParsePage(b *testing.B) {
	b.ReportAllocs()

	for i := 0; i < b.N; i++ {
		_, err := ParsePage("42")
		if err != nil {
			b.Fatalf("parse page: %v", err)
		}
	}
}
```

分配次数和分配字节数常常比单次耗时更能暴露问题。

---

## 4. 比较两次基准

保存结果：

```bash
go test -bench=. -benchmem ./... > old.txt
go test -bench=. -benchmem ./... > new.txt
```

可以使用 `benchstat` 比较：

```bash
go install golang.org/x/perf/cmd/benchstat@latest
benchstat old.txt new.txt
```

性能数据有波动。不要只看一次运行结果。

---

## 5. CPU profile

生成 CPU profile：

```bash
go test -bench=BenchmarkEncodeTasks -cpuprofile=cpu.out ./internal/task
```

查看：

```bash
go tool pprof cpu.out
```

常用 pprof 命令：

```text
top
list FunctionName
web
```

`top` 看最耗 CPU 的函数，`list` 看源码行，`web` 生成调用图。

---

## 6. 内存 profile

生成内存 profile：

```bash
go test -bench=BenchmarkEncodeTasks -memprofile=mem.out ./internal/task
```

查看：

```bash
go tool pprof mem.out
```

如果分配过多，先看：

```text
top
list FunctionName
```

优化前要明确瓶颈在哪里。

---

## 7. 服务中启用 pprof

内部调试服务可以启用：

```go
package debugserver

import (
	"errors"
	"net/http"
	_ "net/http/pprof"
)

func Listen(addr string) error {
	// net/http/pprof 会把调试路由注册到 DefaultServeMux。
	mux := http.DefaultServeMux
	err := http.ListenAndServe(addr, mux)
	if err != nil && !errors.Is(err, http.ErrServerClosed) {
		return err
	}
	return nil
}
```

启动：

```go
go func() {
	if err := debugserver.Listen("127.0.0.1:6060"); err != nil {
		logger.Error("debug server failed", slog.String("error", err.Error()))
	}
}()
```

访问：

```bash
go tool pprof http://127.0.0.1:6060/debug/pprof/profile
go tool pprof http://127.0.0.1:6060/debug/pprof/heap
```

调试端口不要直接暴露到公网。

---

## 8. 性能优化顺序

1. 先写基准或采集 profile。
2. 找出最明显瓶颈。
3. 做一个小改动。
4. 重跑基准。
5. 对比结果。
6. 保留有价值的改动，丢弃无效改动。

没有测量的优化很容易把代码变复杂，却没有真实收益。

---

## 9. 学习检查

检查性能验证时，重点看：

1. 是否有基准测试。
2. 是否使用 `-benchmem`。
3. 准备数据是否排除在计时外。
4. 是否用 profile 定位瓶颈。
5. 是否比较多次基准结果。
6. pprof 调试端口是否受保护。
7. 优化后是否保留可读性。
