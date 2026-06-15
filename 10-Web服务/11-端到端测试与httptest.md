# 端到端测试与 httptest

Web 服务测试的目标是确认真实 HTTP 入口能按预期工作。`httptest` 可以在进程内构造请求、记录响应，也可以启动一个本地测试服务器。

---

## 1. Handler 级测试

如果应用暴露 `http.Handler`，测试可以直接调用：

```go
package app_test

import (
	"net/http"
	"net/http/httptest"
	"testing"
)

func TestHealthz(t *testing.T) {
	application := newTestApp(t)

	req := httptest.NewRequest(http.MethodGet, "/healthz", nil)
	rec := httptest.NewRecorder()

	application.Handler().ServeHTTP(rec, req)

	if rec.Code != http.StatusOK {
		t.Fatalf("status = %d, want %d", rec.Code, http.StatusOK)
	}
	if got := rec.Header().Get("Content-Type"); got != "application/json; charset=utf-8" {
		t.Fatalf("content type = %q", got)
	}
}
```

这种测试速度快，不需要监听真实端口。

---

## 2. JSON 请求测试

```go
func TestCreateTask(t *testing.T) {
	application := newTestApp(t)

	body := strings.NewReader(`{"title":"learn go web"}`)
	req := httptest.NewRequest(http.MethodPost, "/tasks", body)
	req.Header.Set("Content-Type", "application/json")
	rec := httptest.NewRecorder()

	application.Handler().ServeHTTP(rec, req)

	if rec.Code != http.StatusCreated {
		t.Fatalf("status = %d, body = %s", rec.Code, rec.Body.String())
	}

	var resp struct {
		ID     string `json:"id"`
		Title  string `json:"title"`
		Status string `json:"status"`
	}
	if err := json.NewDecoder(rec.Body).Decode(&resp); err != nil {
		t.Fatalf("decode response: %v", err)
	}
	if resp.ID == "" {
		t.Fatalf("id is empty")
	}
	if resp.Title != "learn go web" {
		t.Fatalf("title = %q", resp.Title)
	}
	if resp.Status != "open" {
		t.Fatalf("status = %q", resp.Status)
	}
}
```

测试 HTTP 响应时，状态码、响应头和响应体都值得检查。

---

## 3. 错误路径测试

```go
func TestCreateTaskRejectsEmptyTitle(t *testing.T) {
	application := newTestApp(t)

	req := httptest.NewRequest(http.MethodPost, "/tasks", strings.NewReader(`{"title":"   "}`))
	req.Header.Set("Content-Type", "application/json")
	rec := httptest.NewRecorder()

	application.Handler().ServeHTTP(rec, req)

	if rec.Code != http.StatusUnprocessableEntity {
		t.Fatalf("status = %d, body = %s", rec.Code, rec.Body.String())
	}

	var resp httpx.ErrorResponse
	if err := json.NewDecoder(rec.Body).Decode(&resp); err != nil {
		t.Fatalf("decode error response: %v", err)
	}
	if resp.Error.Code != "invalid_request" {
		t.Fatalf("error code = %q", resp.Error.Code)
	}
}
```

只测成功路径会让服务看起来没问题。Web 服务更常见的缺陷在无效输入、未认证、权限不足、资源不存在这些路径里。

---

## 4. 用 httptest.NewServer 测完整客户端流程

`httptest.NewServer` 会启动一个真实 HTTP server，并返回可请求的 URL：

```go
func TestCreateAndGetTaskOverHTTP(t *testing.T) {
	application := newTestApp(t)
	server := httptest.NewServer(application.Handler())
	defer server.Close()

	client := server.Client()

	createBody := strings.NewReader(`{"title":"write tests"}`)
	createReq, err := http.NewRequest(http.MethodPost, server.URL+"/tasks", createBody)
	if err != nil {
		t.Fatalf("new create request: %v", err)
	}
	createReq.Header.Set("Content-Type", "application/json")

	createResp, err := client.Do(createReq)
	if err != nil {
		t.Fatalf("create request: %v", err)
	}
	defer createResp.Body.Close()

	if createResp.StatusCode != http.StatusCreated {
		data, readErr := io.ReadAll(createResp.Body)
		if readErr != nil {
			t.Fatalf("read create response: %v", readErr)
		}
		t.Fatalf("create status = %d, body = %s", createResp.StatusCode, string(data))
	}
}
```

这种测试比直接调用 handler 更接近真实网络流程，但仍然运行在本地进程内。

---

## 5. 测试中间件

请求 ID 中间件可以这样测试：

```go
func TestRequestIDMiddlewareUsesIncomingID(t *testing.T) {
	next := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		id := httpx.RequestIDFromContext(r.Context())
		if id != "req-123" {
			t.Fatalf("request id = %q", id)
		}
		w.WriteHeader(http.StatusNoContent)
	})

	handler := httpx.RequestID()(next)

	req := httptest.NewRequest(http.MethodGet, "/", nil)
	req.Header.Set("X-Request-ID", "req-123")
	rec := httptest.NewRecorder()

	handler.ServeHTTP(rec, req)

	if rec.Code != http.StatusNoContent {
		t.Fatalf("status = %d", rec.Code)
	}
	if got := rec.Header().Get("X-Request-ID"); got != "req-123" {
		t.Fatalf("response request id = %q", got)
	}
}
```

中间件测试通常不需要启动完整应用，只要构造一个假的 next handler。

---

## 6. 测试替身

service 测试可以使用内存 repository：

```go
func TestServiceCreateTrimsTitle(t *testing.T) {
	repo := task.NewMemoryRepository()
	service, err := task.NewService(repo, func() string {
		return "task-1"
	}, func() time.Time {
		return time.Date(2026, 1, 2, 3, 4, 5, 0, time.UTC)
	})
	if err != nil {
		t.Fatalf("new service: %v", err)
	}

	created, err := service.Create(context.Background(), task.CreateInput{
		Title: "  learn testing  ",
	})
	if err != nil {
		t.Fatalf("create task: %v", err)
	}
	if created.Title != "learn testing" {
		t.Fatalf("title = %q", created.Title)
	}
}
```

固定 ID 和固定时间可以让断言稳定。

---

## 7. 推荐测试命令

```bash
go test ./...
go test -race ./...
```

如果服务有数据库测试，可以单独给它们加 build tag 或使用临时数据库，避免普通单元测试依赖本机环境。

---

## 8. 学习检查

检查 Web 服务测试时，重点看：

1. 是否测试 handler 成功路径。
2. 是否测试 JSON 格式错误。
3. 是否测试字段校验错误。
4. 是否测试资源不存在。
5. 是否测试未认证和权限不足。
6. 是否测试中间件行为。
7. 是否能运行 `go test -race ./...`。
