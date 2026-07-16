# 测试替身、临时资源与 httptest

真实项目会依赖数据库、HTTP 服务、文件系统、时间和随机数。测试要能替换这些依赖，才能稳定、快速、可复现。

---

## 1. 测试替身

定义业务需要的接口：

```go
package task

import "context"

type Repository interface {
	Create(ctx context.Context, task Task) error
	FindByID(ctx context.Context, id string) (Task, error)
}
```

测试中实现假的 repository：

```go
type fakeRepository struct {
	task task.Task
	err  error
}

func (f fakeRepository) Create(ctx context.Context, item task.Task) error {
	return f.err
}

func (f fakeRepository) FindByID(ctx context.Context, id string) (task.Task, error) {
	if f.err != nil {
		return task.Task{}, f.err
	}
	return f.task, nil
}
```

接口小，测试替身就容易写。

---

## 2. 固定时间和 ID

业务代码：

```go
type Service struct {
	repo  Repository
	now   func() time.Time
	newID func() string
}
```

测试代码：

```go
fixedTime := func() time.Time {
	return time.Date(2026, 1, 2, 3, 4, 5, 0, time.UTC)
}

fixedID := func() string {
	return "task-1"
}
```

这样测试不受真实时间和随机 ID 影响。

---

## 3. 临时目录

测试文件读写时使用 `t.TempDir()`：

```go
func TestSaveFile(t *testing.T) {
	dir := t.TempDir()
	path := filepath.Join(dir, "data.json")

	err := Save(path, []byte(`{"name":"go"}`))
	if err != nil {
		t.Fatalf("save: %v", err)
	}

	data, err := os.ReadFile(path)
	if err != nil {
		t.Fatalf("read file: %v", err)
	}
	if string(data) != `{"name":"go"}` {
		t.Fatalf("data = %s", data)
	}
}
```

`t.TempDir()` 会在测试结束后自动清理目录。

---

## 4. t.Cleanup

需要清理资源时使用 `t.Cleanup`：

```go
func newTestServer(t *testing.T) *httptest.Server {
	t.Helper()

	server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusNoContent)
	}))

	t.Cleanup(server.Close)
	return server
}
```

`t.Helper()` 会让失败行指向调用者，更容易定位。

---

## 5. httptest 测 Handler

```go
func TestHealthz(t *testing.T) {
	handler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		w.WriteHeader(http.StatusOK)
		if _, err := w.Write([]byte(`{"status":"ok"}`)); err != nil {
			t.Fatalf("write response: %v", err)
		}
	})

	req := httptest.NewRequest(http.MethodGet, "/healthz", nil)
	rec := httptest.NewRecorder()

	handler.ServeHTTP(rec, req)

	if rec.Code != http.StatusOK {
		t.Fatalf("status = %d", rec.Code)
	}
}
```

`httptest.NewRecorder` 适合在进程内测试 handler。

---

## 6. httptest.NewServer 测 Client

如果要测试 HTTP client，使用真实本地测试服务：

```go
func TestClientFetchTask(t *testing.T) {
	server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		if r.URL.Path != "/tasks/1" {
			http.NotFound(w, r)
			return
		}
		w.Header().Set("Content-Type", "application/json")
		if _, err := w.Write([]byte(`{"id":"1","title":"learn go"}`)); err != nil {
			t.Fatalf("write response: %v", err)
		}
	}))
	t.Cleanup(server.Close)

	client := NewClient(server.URL, server.Client())
	task, err := client.FetchTask(context.Background(), "1")
	if err != nil {
		t.Fatalf("fetch task: %v", err)
	}
	if task.Title != "learn go" {
		t.Fatalf("title = %q", task.Title)
	}
}
```

这样测试不会依赖外部网络。

---

## 7. 环境变量测试

使用 `t.Setenv`：

```go
func TestLoadConfig(t *testing.T) {
	t.Setenv("ADDR", ":9090")

	cfg, err := Load()
	if err != nil {
		t.Fatalf("load config: %v", err)
	}
	if cfg.Addr != ":9090" {
		t.Fatalf("addr = %q", cfg.Addr)
	}
}
```

`t.Setenv` 会在测试结束后恢复环境。

---

## 8. 学习检查

检查测试替身和资源管理时，重点看：

1. 是否能替换数据库、HTTP client、时间、ID。
2. 测试是否使用 `t.TempDir()`。
3. 资源是否用 `t.Cleanup` 清理。
4. 测试 helper 是否调用 `t.Helper()`。
5. HTTP client 测试是否使用 `httptest.NewServer`。
6. 环境变量是否使用 `t.Setenv`。
7. 测试是否避免依赖外部网络。
