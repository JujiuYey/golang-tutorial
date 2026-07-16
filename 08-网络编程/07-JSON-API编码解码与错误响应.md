# 07: JSON API：编码、解码、错误响应与校验

这篇把前面几篇的零件组装成 Go 网络编程的高频场景——JSON API。核心不是会调 `json.Encode`（那是一行的事），而是把边界都守住：body 限大小、字段防拼错、业务校验、错误响应统一格式。学完你会有三个从此复用到底的小函数：`writeJSON`、`writeError`、`decodeJSON`。

> 本篇实跑一个 `POST /users` 接口（`localhost:18014`），所有响应都是 curl 真实结果。

---

## 1. 请求和响应结构

```go
type CreateUserRequest struct {
	Name string `json:"name"`
	Age  int    `json:"age"`
}

type UserResponse struct {
	ID   int    `json:"id"`
	Name string `json:"name"`
	Age  int    `json:"age"`
}
```

请求一个结构、响应一个结构，**分开定义**。Node 里顺手 `res.json(dbRow)` 把数据库行直接甩出去的习惯，在这要戒掉：数据库模型一改字段（或者本来就带密码哈希这种列），API 就跟着漏。请求/响应结构就是你 API 的"合同"，和存储怎么变无关。

---

## 2. 写 JSON 响应：writeJSON

第 03 篇的老朋友，加上日志成为定稿版：

```go
func writeJSON(w http.ResponseWriter, status int, v any) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	if err := json.NewEncoder(w).Encode(v); err != nil {
		slog.Error("write json failed", "err", err)
	}
}
```

为什么编码错误只记日志不返回？因为 `WriteHeader` 已经把状态码发出去了——响应"开弓没有回头箭"，中途失败没法改口成 500，只能留个日志给自己。

---

## 3. 统一错误响应：writeError

```go
type ErrorResponse struct {
	Code    string `json:"code"`
	Message string `json:"message"`
}

func writeError(w http.ResponseWriter, status int, code, message string) {
	writeJSON(w, status, ErrorResponse{Code: code, Message: message})
}
```

所有错误长一个模样：`{"code":"...","message":"..."}`。`code` 是给程序看的稳定标识（前端可以 `switch` 它），`message` 是给人看的。**API 里最忌讳一半错误是 JSON、一半是 `http.Error` 的纯文本**——调用方得写两套解析。

---

## 4. 解码 JSON 请求：decodeJSON

Express 里 `app.use(express.json())` 一行搞定的事，Go 里显式写——顺便把 Express 默认不给你的两层防护也装上：

```go
func decodeJSON(w http.ResponseWriter, r *http.Request, dst any) bool {
	r.Body = http.MaxBytesReader(w, r.Body, 1<<20) // ①限 1MB(03 篇的规矩)

	dec := json.NewDecoder(r.Body)
	dec.DisallowUnknownFields() // ②不认识的字段直接报错
	if err := dec.Decode(dst); err != nil {
		writeError(w, http.StatusBadRequest, "invalid_json", err.Error())
		return false
	}
	return true
}
```

返回 bool 是个小设计：失败时它**已经把 400 响应写好了**，调用方只需 `if !decodeJSON(...) { return }` 一行。

看它拦下三种坏请求（都是实跑）：

```bash
curl -H 'Content-Type: application/json' -d '{bad json' localhost:18014/users
```

```text
{"code":"invalid_json","message":"invalid character 'b' looking for beginning of object key string"}
```

```bash
curl -H 'Content-Type: application/json' -d '{"name":"alice","agee":18}' localhost:18014/users
```

```text
{"code":"invalid_json","message":"json: unknown field \"agee\""}
```

```bash
curl -H 'Content-Type: application/json' -d '{"name":"alice","age":"18"}' localhost:18014/users
```

```text
{"code":"invalid_json","message":"json: cannot unmarshal string into Go struct field CreateUserRequest.age of type int"}
```

第二条最值钱：客户端把 `age` 拼成 `agee`，没有 `DisallowUnknownFields` 的话解码**静默成功**、`Age` 得零值 0，然后你排查半天"为什么年龄总是 0"。第三条是 Go 强类型的福利——`"18"`（字符串）进不了 `int` 字段，JS 里这种类型混乱要到业务深处才炸。

---

## 5. 校验请求

🕳️ 坑：以为解码成功 = 请求合法。实际上 JSON 格式正确只说明"是合法的 JSON"，`{"age":18}` 没有 name 也能顺利解码（`Name` 得零值 `""`）。业务规则要自己查：

```go
func (r CreateUserRequest) Validate() error {
	if strings.TrimSpace(r.Name) == "" {
		return errors.New("name is required")
	}
	if r.Age < 0 {
		return errors.New("age must be non-negative")
	}
	return nil
}
```

```bash
curl -H 'Content-Type: application/json' -d '{"age":18}' localhost:18014/users
```

```text
{"code":"invalid_request","message":"name is required"}
```

把校验做成请求结构的方法，handler 里一句 `req.Validate()`，规则和结构定义住在一起。

**一句话总结：解码管"长得对不对"，校验管"内容行不行"，缺一不可。**

---

## 6. 组装：创建资源 handler

零件齐了，拼出完整的 `POST /users`：

```go
func createUser(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodPost { // ①方法
		w.Header().Set("Allow", http.MethodPost)
		writeError(w, http.StatusMethodNotAllowed, "method_not_allowed", "method not allowed")
		return
	}

	var req CreateUserRequest
	if !decodeJSON(w, r, &req) { // ②解码(内含限流和未知字段检查)
		return
	}
	if err := req.Validate(); err != nil { // ③校验
		writeError(w, http.StatusBadRequest, "invalid_request", err.Error())
		return
	}

	user := UserResponse{ID: 1, Name: req.Name, Age: req.Age} // ④业务
	writeJSON(w, http.StatusCreated, user)                    // ⑤响应 201
}
```

正常路径：

```bash
curl -i -H 'Content-Type: application/json' -d '{"name":"alice","age":18}' localhost:18014/users
```

```text
HTTP/1.1 201 Created
Content-Type: application/json
Content-Length: 33

{"id":1,"name":"alice","age":18}
```

方法不对（连 405 都是 JSON 格式，风格统一到底）：

```bash
curl -i localhost:18014/users
```

```text
HTTP/1.1 405 Method Not Allowed
Allow: POST
Content-Type: application/json
Content-Length: 61

{"code":"method_not_allowed","message":"method not allowed"}
```

方法 → 解码 → 校验 → 业务 → 响应，**每一步失败就 return，一条路走到黑**。这就是 03 模块"卫述句"风格在 HTTP 里的样子。

---

## 7. 状态码习惯

| 状态码 | 用在什么时候 |
|--------|-------------|
| `200 OK` | 查询、更新成功 |
| `201 Created` | 创建成功 |
| `204 No Content` | 删除成功且没有响应体 |
| `400 Bad Request` | 请求格式或参数错误 |
| `401 Unauthorized` | 没登录（未认证） |
| `403 Forbidden` | 登录了但没权限 |
| `404 Not Found` | 资源不存在 |
| `405 Method Not Allowed` | 方法不支持 |
| `500 Internal Server Error` | 服务端自己的锅 |

好记的分法：**2xx 成了、4xx 你的错（调用方）、5xx 我的错（服务方）**。401 和 403 最容易混：401 是"你是谁我不知道"，403 是"我知道你是谁,但你不行"。

---

## 8. 不要把内部错误直接返回给用户

```go
if err != nil {
	slog.Error("database failed", "err", err) // 细节进日志
	writeError(w, http.StatusInternalServerError, "internal_error", "internal server error") // 外面只给一句话
	return
}
```

数据库连接串、SQL 语句、文件路径这些都可能藏在 err 里，直接 `err.Error()` 甩给客户端等于漏内裤。**规矩：4xx 的 message 可以具体（帮调用方改对请求），5xx 的 message 一律模糊（细节只进日志，配合第 05 篇的 request id 定位）。**

---

## 9. JSON 内容类型

要求客户端声明"我发的是 JSON"：

```go
if ct := r.Header.Get("Content-Type"); !strings.Contains(ct, "application/json") {
	writeError(w, http.StatusUnsupportedMediaType, "unsupported_media_type", "content type must be application/json")
	return
}
```

```bash
curl -X POST -H 'Content-Type: text/plain' -d '{"name":"a"}' localhost:18014/users
```

```text
{"code":"unsupported_media_type","message":"content type must be application/json"}
```

用 `strings.Contains` 而不是 `==`，因为常见的实际值是 `application/json; charset=utf-8`——精确比较会把合法客户端也拒了。

---

## 10. API 设计检查

1. 请求结构和响应结构是否分离（不直接暴露存储模型）？
2. body 是否限制大小（`MaxBytesReader`）？
3. 是否 `DisallowUnknownFields` 防拼错字段？
4. 解码之后是否还有业务校验（`Validate`）？
5. 所有错误（包括 405）是否统一 JSON 格式？
6. 5xx 是否只写日志、不外泄细节？

---

## 本篇重点

- [ ] 三件套复用到底：`writeJSON`（Content-Type → 状态码 → Encode）、`writeError`（统一 `{code, message}`）、`decodeJSON`（限 1MB + 拒未知字段 + 失败自动 400）。
- [ ] `DisallowUnknownFields` 把"字段拼错静默得零值"变成明确的 400——不开它的代价是排查"为什么总是 0"。
- [ ] 解码成功 ≠ 请求合法：JSON 格式检查之后必须跟业务校验（`Validate` 方法）。
- [ ] handler 骨架固定：方法 → 解码 → 校验 → 业务 → 响应，每步失败即 return。
- [ ] 4xx 消息可以具体，5xx 消息必须模糊（细节进日志）；Content-Type 用 Contains 匹配别用相等。

---

## 练习

1. 写 `writeJSON` 和 `writeError`。
2. 写 `decodeJSON`，限制 body 为 1MB（提示：返回 bool，失败时内部写好 400）。
3. 定义 `CreateUserRequest` 并实现 `Validate`。
4. 写 `POST /users` handler，用 curl 验证成功和至少三种失败路径（坏 JSON、未知字段、缺 name）。
5. 对错误路径写 `httptest` 测试（先预习，第 11 篇细讲）。
