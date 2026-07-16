# 04: 路由注册与 Handler 组织

这篇解决的问题是:**几十条路由怎么注册得清楚、handler 怎么写得薄**。学完你能用标准库 `ServeMux` 搞定"方法 + 路径 + 路径参数"的匹配(不需要第三方路由库),并且知道 query 参数该怎么解析才不埋雷。

对比 Express:`app.get('/tasks/:id', handler)` 你闭着眼都会写。Go 1.22 之后标准库终于有了对等物——`mux.HandleFunc("GET /tasks/{id}", handler)`,`:id` 变成 `{id}`,`req.params.id` 变成 `r.PathValue("id")`。

---

## 1. 使用 ServeMux 注册路由

Go 1.22 起,`http.ServeMux` 的路由模式支持方法和路径一起匹配:

```go
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

拆一下路由模式:

```text
"GET /tasks/{id}"
 ─┬─ ────┬─── ─┬─
  ①方法   ②路径  ③路径参数,handler 里用 r.PathValue("id") 取
```

方法不匹配时,mux 自动回 405——不用自己写 `if r.Method != ...` 了。实测(服务跑在 19080 端口):

```bash
curl -si -X PUT http://localhost:19080/tasks | head -1
```

```text
HTTP/1.1 405 Method Not Allowed
```

🕳️ 坑:方法匹配是 Go **1.22** 才有的。读老项目会看到 `mux.HandleFunc("/tasks", ...)` 加手动 method 判断,或者 gorilla/mux、chi 这些第三方路由库——它们存在的历史原因之一就是老 ServeMux 太弱。

**一句话总结:路由集中注册在一个函数里,想知道服务有哪些接口,看这一屏就够。**

---

## 2. App 暴露 Handler

给 `App` 加一个 `Handler()` 方法,把整棵路由树作为一个 `http.Handler` 交出去:

```go
func (a *App) Handler() http.Handler {
    return a.mux
}
```

启动时用它:

```go
server := &http.Server{
    Addr:              a.config.Addr,
    Handler:           a.Handler(),
    ReadHeaderTimeout: a.config.ReadHeaderTimeout,
    WriteTimeout:      a.config.WriteTimeout,
}
```

测试时也用它——**不经过网络**,直接进程内调用:

```go
request := httptest.NewRequest(http.MethodGet, "/healthz", nil)
response := httptest.NewRecorder()

app.Handler().ServeHTTP(response, request)
```

对比 Node:supertest 的 `request(app)` 干的就是这件事。`Handler()` 是启动和测试共用的同一个入口,保证"测的就是跑的"。

---

## 3. Handler 方法保持薄

handler 的形状非常固定,四步流水线:

```go
func (h *HTTPHandler) Get(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")                          // ① 取参数
    if id == "" {
        httpx.WriteErrorWithCode(w, http.StatusBadRequest, "missing_id", "missing task id")
        return
    }

    task, err := h.service.Get(r.Context(), id)      // ② 调 service
    if err != nil {
        writeTaskError(w, err)                       // ③ 错误 → HTTP 响应
        return
    }

    if err := httpx.WriteJSON(w, http.StatusOK, taskResponseFromModel(task)); err != nil {
        h.logger.Error("write response failed", slog.String("error", err.Error()))
    }                                                // ④ 结果 → 响应 DTO
}
```

实测:

```bash
curl -s http://localhost:19080/tasks/task_1
curl -si http://localhost:19080/tasks/nope | head -1
```

```text
{"id":"task_1","title":"learn go web","status":"open","created_at":"2026-07-16T17:14:04.298555Z","updated_at":"2026-07-16T17:14:04.298555Z"}
HTTP/1.1 404 Not Found
```

`writeTaskError`、`WriteJSON`、`taskResponseFromModel` 这些配角分别是第 06、05 篇的主角。handler 里看不到 SQL、看不到业务规则——它只是 HTTP 和业务之间的**翻译**。

**一句话总结:handler 只做四件事——取参数、调 service、翻译错误、翻译结果。**

---

## 4. 路由命名要统一

推荐资源名复数 + 标准方法:

| 方法 | 路径 | 含义 |
| --- | --- | --- |
| `GET` | `/tasks` | 列表 |
| `POST` | `/tasks` | 创建 |
| `GET` | `/tasks/{id}` | 详情 |
| `PATCH` | `/tasks/{id}` | 局部更新 |
| `DELETE` | `/tasks/{id}` | 删除 |

少量"动作"类接口用子资源表达:

| 方法 | 路径 | 含义 |
| --- | --- | --- |
| `POST` | `/tasks/{id}/complete` | 完成任务 |
| `POST` | `/sessions` | 登录创建会话 |
| `DELETE` | `/sessions/current` | 退出当前会话 |

这套约定和 Express/Rails 的 RESTful 习惯完全一致。路径是服务的**对外契约**——客户端代码、文档、监控都挂在上面,改一次伤一片,所以从第一天就要稳定、可预测。

---

## 5. Query 参数解析

### 🕳️ 坑:query 参数全是 string,而且可能是任何东西

以为会怎样:`?page=2` 里的 2 是数字。
实际怎样:`r.URL.Query().Get("page")` 返回 `"2"`(string),也可能返回 `""`(没传)或 `"abc"`(乱传)。
为什么:URL 就是文本,和 Express 的 `req.query` 一样,类型转换和校验都得自己来——但 Go 不会帮你隐式转换,`"2" * 1` 这种 JS 小聪明不存在。

解析时要区分三种情况:**没传(用默认值)、格式错(报错)、超范围(报错)**:

```go
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

    return ListParams{Page: page, PageSize: pageSize, Status: status}, nil
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

实测乱传的效果:

```bash
curl -si 'http://localhost:19080/tasks?page=abc' | head -1
curl -s 'http://localhost:19080/tasks?page=abc'
curl -s 'http://localhost:19080/tasks?status=archived'
```

```text
HTTP/1.1 400 Bad Request
{"error":{"code":"invalid_params","message":"invalid page: strconv.Atoi: parsing \"abc\": invalid syntax"}}
{"error":{"code":"invalid_params","message":"invalid status"}}
```

正常请求:

```bash
curl -s 'http://localhost:19080/tasks?status=open&page=1&page_size=20'
```

```text
{"items":[{"id":"task_1","title":"learn go web","status":"open","created_at":"2026-07-16T17:14:04.298555Z","updated_at":"2026-07-16T17:14:04.298555Z"}],"page":1,"page_size":20}
```

---

## 6. 健康检查接口

健康检查不属于任何业务域,适合放在 `app` 层:

```go
func (a *App) healthz(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json; charset=utf-8")
    w.WriteHeader(http.StatusOK)
    if _, err := w.Write([]byte(`{"status":"ok"}` + "\n")); err != nil {
        a.logger.Error("write health response failed", slog.String("error", err.Error()))
    }
}
```

```bash
curl -si http://localhost:19080/healthz
```

```text
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
X-Request-Id: 9ad96b7dc219a2667134c7795517559a
Date: Thu, 16 Jul 2026 17:13:36 GMT
Content-Length: 16

{"status":"ok"}
```

两个细节:路由模式写了 `GET /healthz`,方法拦截交给 mux(第 1 节),handler 里不用再查;`w.Write` 也可能失败(客户端断开),至少记条日志,别让 linter 和事故都找上门。

---

## 本篇重点

- [ ] Go 1.22+ 的 `ServeMux` 支持 `"GET /tasks/{id}"` 方法+路径匹配,`r.PathValue("id")` 取路径参数(= Express 的 `req.params.id`),方法不符自动 405。
- [ ] `App.Handler()` 是启动和测试共用的入口,测试用 `httptest` 进程内调用,不走网络。
- [ ] handler 四步流水线:取参数 → 调 service → 翻译错误 → 翻译结果;别塞业务逻辑。
- [ ] 路径命名用资源复数 + 标准方法,动作用子资源;路径是对外契约。
- [ ] query 参数是 string,解析要区分"没传用默认 / 格式错 / 超范围"三种情况。

---

## 练习

给 `book-api` 注册路由并实现列表接口:

要求:

1. 注册 `GET /healthz`、`GET /books`、`POST /books`、`GET /books/{id}`、`DELETE /books/{id}`。
2. `GET /books` 支持 `?page=`、`?page_size=`(默认 1 / 20,上限 100)、`?status=`,非法值返回 400 和具体的错误信息。
3. 用 curl 验证:405(错误方法)、400(`page=0`)、200 各一次。
4. `App` 暴露 `Handler() http.Handler`,并用 `httptest.NewRequest` + `NewRecorder` 给 `/healthz` 写一个不走网络的测试。

提示:分页解析可以照抄第 5 节的 `parsePositiveInt` 思路;`{id}` 路由和 `/books` 路由是两条,不会互相冲突。
