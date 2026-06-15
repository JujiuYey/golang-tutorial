# 路由注册与 Handler 组织

路由层负责把 HTTP 方法和路径分配给 handler。它应该清楚、集中、容易搜索。

---

## 1. 使用 ServeMux 注册路由

Go 1.22 起，`http.ServeMux` 支持方法和路径一起匹配：

```go
package app

import (
	"fmt"

	"example.com/task-api/internal/task"
)

func (a *App) registerRoutes() error {
	taskHandler, err := task.NewHTTPHandler(a.taskService, a.logger)
	if err != nil {
		return fmt.Errorf("new task handler: %w", err)
	}

	a.mux.HandleFunc("GET /healthz", a.healthz)
	a.mux.HandleFunc("GET /tasks", taskHandler.List)
	a.mux.HandleFunc("POST /tasks", taskHandler.Create)
	a.mux.HandleFunc("GET /tasks/{id}", taskHandler.Get)
	a.mux.HandleFunc("PATCH /tasks/{id}", taskHandler.Update)
	a.mux.HandleFunc("DELETE /tasks/{id}", taskHandler.Delete)
	return nil
}
```

`GET /tasks/{id}` 里的 `{id}` 可以通过 `r.PathValue("id")` 读取。

---

## 2. App 暴露 Handler

应用结构体可以暴露一个 `Handler` 方法，方便测试和启动复用：

```go
package app

import "net/http"

func (a *App) Handler() http.Handler {
	return a.mux
}
```

启动服务时使用它：

```go
server := &http.Server{
	Addr:              a.config.Addr,
	Handler:           a.Handler(),
	ReadHeaderTimeout: a.config.ReadHeaderTimeout,
	WriteTimeout:      a.config.WriteTimeout,
}
```

测试时也可以直接使用它：

```go
request := httptest.NewRequest(http.MethodGet, "/healthz", nil)
response := httptest.NewRecorder()

app.Handler().ServeHTTP(response, request)
```

---

## 3. Handler 方法保持薄

handler 方法的形状通常很稳定：

```go
package task

import (
	"log/slog"
	"net/http"

	"example.com/task-api/internal/httpx"
)

func (h *HTTPHandler) Get(w http.ResponseWriter, r *http.Request) {
	id := r.PathValue("id")
	if id == "" {
		httpx.WriteErrorWithCode(w, http.StatusBadRequest, "missing_id", "missing task id")
		return
	}

	task, err := h.service.Get(r.Context(), id)
	if err != nil {
		writeTaskError(w, err)
		return
	}

	if err := httpx.WriteJSON(w, http.StatusOK, taskResponseFromModel(task)); err != nil {
		h.logger.Error("write response failed", slog.String("error", err.Error()))
	}
}
```

这个方法只做四件事：

1. 从 HTTP 请求中取参数。
2. 调用 service。
3. 把错误转换成 HTTP 响应。
4. 把业务结果转换成响应 DTO。

---

## 4. 路由命名要统一

推荐使用资源名复数：

| 方法 | 路径 | 含义 |
| --- | --- | --- |
| `GET` | `/tasks` | 列表 |
| `POST` | `/tasks` | 创建 |
| `GET` | `/tasks/{id}` | 详情 |
| `PATCH` | `/tasks/{id}` | 局部更新 |
| `DELETE` | `/tasks/{id}` | 删除 |

少量动作类接口可以使用子资源：

| 方法 | 路径 | 含义 |
| --- | --- | --- |
| `POST` | `/tasks/{id}/complete` | 完成任务 |
| `POST` | `/sessions` | 登录创建会话 |
| `DELETE` | `/sessions/current` | 退出当前会话 |

路径是服务对外契约，要稳定、可读、可预测。

---

## 5. Query 参数解析

列表接口经常需要分页和过滤：

```go
package task

import (
	"fmt"
	"net/http"
	"strconv"
)

type ListParams struct {
	Page     int
	PageSize int
	Status   Status
}

func parseListParams(r *http.Request) (ListParams, error) {
	query := r.URL.Query()

	page, err := parsePositiveInt(query.Get("page"), 1)
	if err != nil {
		return ListParams{}, fmt.Errorf("invalid page: %w", err)
	}

	pageSize, err := parsePositiveInt(query.Get("page_size"), 20)
	if err != nil {
		return ListParams{}, fmt.Errorf("invalid page_size: %w", err)
	}
	if pageSize > 100 {
		return ListParams{}, fmt.Errorf("page_size must be <= 100")
	}

	status := Status(query.Get("status"))
	if status != "" && !status.Valid() {
		return ListParams{}, fmt.Errorf("invalid status")
	}

	return ListParams{
		Page:     page,
		PageSize: pageSize,
		Status:   status,
	}, nil
}

func parsePositiveInt(raw string, fallback int) (int, error) {
	if raw == "" {
		return fallback, nil
	}
	value, err := strconv.Atoi(raw)
	if err != nil {
		return 0, err
	}
	if value <= 0 {
		return 0, fmt.Errorf("must be positive")
	}
	return value, nil
}
```

解析 query 时要区分默认值、格式错误、范围错误。

---

## 6. 健康检查接口

健康检查适合放在 `app` 层：

```go
package app

import (
	"log/slog"
	"net/http"
)

func (a *App) healthz(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodGet {
		http.Error(w, "method not allowed", http.StatusMethodNotAllowed)
		return
	}
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusOK)
	if _, err := w.Write([]byte(`{"status":"ok"}`)); err != nil {
		a.logger.Error("write health response failed", slog.String("error", err.Error()))
	}
}
```

如果使用了方法匹配路由，额外检查 method 的需求会减少。这个例子保留检查，是为了展示普通 handler 中如何明确拒绝方法。

---

## 7. 学习检查

检查路由和 handler 时，重点看：

1. 路由是否集中注册。
2. 路径命名是否统一。
3. handler 是否能快速读懂。
4. handler 是否直接访问数据库。
5. 路径参数和 query 参数是否有校验。
6. 写响应失败是否至少记录日志。
7. 测试是否能直接调用 `app.Handler()`。
