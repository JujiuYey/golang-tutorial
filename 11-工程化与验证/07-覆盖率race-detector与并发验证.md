# 覆盖率、race detector 与并发验证

测试通过只说明已有测试通过。覆盖率和 race detector 能进一步提醒你：哪些路径还没测，并发访问是否有数据竞争。

---

## 1. 查看覆盖率

```bash
go test -cover ./...
```

生成覆盖率文件：

```bash
go test -coverprofile=coverage.out ./...
```

生成 HTML 报告：

```bash
go tool cover -html=coverage.out -o coverage.html
```

覆盖率报告适合找盲区，不适合当作唯一质量指标。

---

## 2. 覆盖率盲区

示例函数：

```go
func ParsePage(raw string) (int, error) {
	if raw == "" {
		return 1, nil
	}
	page, err := strconv.Atoi(raw)
	if err != nil {
		return 0, err
	}
	if page <= 0 {
		return 0, fmt.Errorf("page must be positive")
	}
	return page, nil
}
```

测试要覆盖：

- 空字符串默认值。
- 正常数字。
- 非数字。
- 小于等于 0。

覆盖率可以提醒你某些分支没有执行。

---

## 3. race detector

运行：

```bash
go test -race ./...
```

数据竞争示例：

```go
type Counter struct {
	value int
}

func (c *Counter) Inc() {
	c.value++
}

func (c *Counter) Value() int {
	return c.value
}
```

如果多个 goroutine 同时调用 `Inc`，这里有数据竞争。修复方式之一：

```go
type Counter struct {
	mu    sync.Mutex
	value int
}

func (c *Counter) Inc() {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.value++
}

func (c *Counter) Value() int {
	c.mu.Lock()
	defer c.mu.Unlock()
	return c.value
}
```

---

## 4. race 测试要真的并发

如果测试没有触发并发访问，`-race` 也查不到问题：

```go
func TestCounterConcurrent(t *testing.T) {
	var counter Counter
	var wg sync.WaitGroup

	for i := 0; i < 100; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			counter.Inc()
		}()
	}

	wg.Wait()

	if got := counter.Value(); got != 100 {
		t.Fatalf("value = %d, want 100", got)
	}
}
```

并发代码要写并发测试，再配合 `go test -race`。

---

## 5. race detector 的成本

`-race` 会让测试变慢，内存占用也会上升。常见策略：

- 本地开发时针对相关包运行。
- 提交前跑全量。
- CI 中至少对核心包或全项目运行。

命令：

```bash
go test -race ./internal/...
```

---

## 6. 覆盖率阈值要谨慎

CI 可以设置覆盖率阈值，但不要把数字当成目标本身。一个 90% 覆盖率的项目仍然可能漏掉关键错误路径。

更好的检查方式：

1. 新功能是否有测试。
2. 错误路径是否有测试。
3. 并发路径是否有 race 验证。
4. 覆盖率报告是否有明显空白区域。

---

## 7. 学习检查

检查覆盖率和并发验证时，重点看：

1. 是否能生成覆盖率报告。
2. 是否阅读 HTML 报告找盲区。
3. 错误分支是否被测试。
4. 并发代码是否有并发测试。
5. 是否运行 `go test -race ./...`。
6. race 失败时是否能定位到共享变量。
7. 覆盖率阈值是否服务于质量目标。
