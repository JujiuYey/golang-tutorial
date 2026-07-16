# Web 服务综合练习

本练习把本模块的知识串起来：使用标准库实现一个任务管理 API，要求具备清楚分层、统一错误、请求校验、中间件、优雅关闭和测试。

---

## 1. 目标

实现一个 `task-api` 服务，支持：

- 创建任务。
- 查询任务列表。
- 查询任务详情。
- 更新任务标题。
- 完成任务。
- 删除任务。
- 健康检查。

第一版使用内存 repository。完成后可以扩展成数据库 repository。

---

## 2. 推荐目录

```text
task-api/
  go.mod
  cmd/
    task-api/
      main.go
  internal/
    app/
      app.go
      routes.go
      lifecycle.go
    config/
      config.go
    httpx/
      json.go
      errors.go
      middleware.go
      request_id.go
    task/
      model.go
      service.go
      repository.go
      memory_repository.go
      handler.go
      dto.go
```

保持结构清楚比追求目录数量更重要。每个文件要能说明自己的职责。

---

## 3. API 契约

### 3.1 健康检查

```http
GET /healthz
```

成功响应：

```json
{
  "status": "ok"
}
```

### 3.2 创建任务

```http
POST /tasks
Content-Type: application/json

{
  "title": "learn go web"
}
```

成功响应：

```json
{
  "id": "task_123",
  "title": "learn go web",
  "status": "open",
  "created_at": "2026-01-02T03:04:05Z",
  "updated_at": "2026-01-02T03:04:05Z"
}
```

### 3.3 查询列表

```http
GET /tasks?status=open&page=1&page_size=20
```

响应：

```json
{
  "items": [
    {
      "id": "task_123",
      "title": "learn go web",
      "status": "open",
      "created_at": "2026-01-02T03:04:05Z",
      "updated_at": "2026-01-02T03:04:05Z"
    }
  ],
  "page": 1,
  "page_size": 20
}
```

### 3.4 查询详情

```http
GET /tasks/{id}
```

不存在时返回：

```json
{
  "error": {
    "code": "task_not_found",
    "message": "task not found"
  }
}
```

状态码为 `404`。

### 3.5 更新标题

```http
PATCH /tasks/{id}
Content-Type: application/json

{
  "title": "learn go web deeply"
}
```

成功时返回更新后的任务。

### 3.6 完成任务

```http
POST /tasks/{id}/complete
```

成功时返回更新后的任务。已经完成的任务再次完成时返回 `409 Conflict`。

### 3.7 删除任务

```http
DELETE /tasks/{id}
```

成功时返回 `204 No Content`。

---

## 4. 业务规则

任务模型：

```go
type Status string

const (
	StatusOpen Status = "open"
	StatusDone Status = "done"
)

type Task struct {
	ID        string
	Title     string
	Status    Status
	CreatedAt time.Time
	UpdatedAt time.Time
}
```

规则：

1. 标题 trim 后不能为空。
2. 标题最多 120 个字符。
3. 新任务状态为 `open`。
4. 完成任务后状态变为 `done`。
5. `done` 状态的任务不能再次完成。
6. 更新标题会刷新 `UpdatedAt`。
7. 删除不存在的任务返回 `task_not_found`。

---

## 5. 错误响应

所有错误使用统一结构：

```json
{
  "error": {
    "code": "invalid_request",
    "message": "title is required"
  }
}
```

推荐错误映射：

| 场景 | 状态码 | code |
| --- | --- | --- |
| JSON 格式错误 | `400` | `invalid_json` |
| 字段校验失败 | `422` | `invalid_request` |
| 资源不存在 | `404` | `task_not_found` |
| 重复完成 | `409` | `task_already_done` |
| 未知错误 | `500` | `internal_error` |

---

## 6. 中间件要求

至少实现：

1. Request ID：读取或生成 `X-Request-ID`，写回响应头。
2. Access Log：记录 method、path、status、duration、request_id。
3. Recover：捕获 panic，记录堆栈，返回统一 `500`。

进阶要求：

1. Bearer Token 认证。
2. 基于角色的 `admin` 路由保护。
3. 每个请求设置超时 context。

---

## 7. 启动和关闭

`cmd/task-api/main.go` 应满足：

1. 从环境变量加载 `ADDR`、`LOG_LEVEL`、`SHUTDOWN_TIMEOUT`。
2. 显式创建 `http.Server`。
3. 设置 `ReadHeaderTimeout`。
4. 监听 `os.Interrupt` 和 `SIGTERM`。
5. 收到信号后调用 `Shutdown`。
6. 关闭 repository 或数据库资源。

---

## 8. 测试要求

至少覆盖：

1. `GET /healthz` 返回成功。
2. 创建任务成功。
3. 空标题被拒绝。
4. 查询不存在任务返回 `404`。
5. 完成任务成功。
6. 重复完成返回 `409`。
7. 删除任务后再次查询返回 `404`。
8. Request ID 中间件写回响应头。
9. Recover 中间件能捕获 panic。

推荐命令：

```bash
go test ./...
go test -race ./...
```

---

## 9. curl 验证

启动服务：

```bash
ADDR=:8080 go run ./cmd/task-api
```

健康检查：

```bash
curl -i http://localhost:8080/healthz
```

创建任务：

```bash
curl -i \
  -H 'Content-Type: application/json' \
  -d '{"title":"learn go web"}' \
  http://localhost:8080/tasks
```

查询列表：

```bash
curl -i 'http://localhost:8080/tasks?status=open&page=1&page_size=20'
```

完成任务：

```bash
curl -i -X POST http://localhost:8080/tasks/task_123/complete
```

---

## 10. 自查清单

完成后逐项检查：

1. handler 没有直接访问 map 或数据库。
2. service 不依赖 `net/http`。
3. repository 所有方法都接收 context。
4. JSON body 有大小限制。
5. 错误响应结构统一。
6. 状态码和错误码稳定。
7. map 被锁保护。
8. 启动错误会返回到 `main`。
9. 关闭流程有超时。
10. 测试覆盖了错误路径。

这个练习完成后，你应该能独立搭出一个小型 Go Web 服务，并能判断代码结构是否清楚、边界是否稳定、验证是否充分。
