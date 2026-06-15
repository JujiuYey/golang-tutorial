# 07: JSON API：编码、解码、错误响应与校验

JSON API 是 Go 网络编程的高频场景。核心不是只会 `json.NewEncoder(w).Encode`，还要处理方法、body 限制、未知字段、错误响应和状态码。

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

请求结构和响应结构可以分开。不要为了省事把数据库模型直接暴露成 API 响应。

---

## 2. 解码 JSON 请求

```go
func decodeJSON(w http.ResponseWriter, r *http.Request, dst any) bool {
    r.Body = http.MaxBytesReader(w, r.Body, 1<<20)

    dec := json.NewDecoder(r.Body)
    dec.DisallowUnknownFields()
    if err := dec.Decode(dst); err != nil {
        writeError(w, http.StatusBadRequest, "invalid_json", err.Error())
        return false
    }
    return true
}
```

`MaxBytesReader` 限制 body 大小。`DisallowUnknownFields` 可以发现拼错字段。

---

## 3. 写 JSON 响应

```go
func writeJSON(w http.ResponseWriter, status int, v any) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    if err := json.NewEncoder(w).Encode(v); err != nil {
        slog.Error("write json failed", "err", err)
    }
}
```

响应开始写出后，编码失败通常只能记录日志。

---

## 4. 统一错误响应

```go
type ErrorResponse struct {
    Code    string `json:"code"`
    Message string `json:"message"`
}

func writeError(w http.ResponseWriter, status int, code, message string) {
    writeJSON(w, status, ErrorResponse{
        Code:    code,
        Message: message,
    })
}
```

统一格式方便前端、CLI 或其他调用方处理错误。

---

## 5. 校验请求

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

解码成功只代表 JSON 格式正确，业务字段仍然要校验。

---

## 6. 创建资源

```go
func createUser(w http.ResponseWriter, r *http.Request) {
    if r.Method != http.MethodPost {
        w.Header().Set("Allow", http.MethodPost)
        writeError(w, http.StatusMethodNotAllowed, "method_not_allowed", "method not allowed")
        return
    }

    var req CreateUserRequest
    if !decodeJSON(w, r, &req) {
        return
    }
    if err := req.Validate(); err != nil {
        writeError(w, http.StatusBadRequest, "invalid_request", err.Error())
        return
    }

    user := UserResponse{ID: 1, Name: req.Name, Age: req.Age}
    writeJSON(w, http.StatusCreated, user)
}
```

创建成功通常返回 201。

---

## 7. 状态码习惯

常见状态码：

- `200 OK`: 查询或更新成功。
- `201 Created`: 创建成功。
- `204 No Content`: 删除成功且没有响应体。
- `400 Bad Request`: 请求格式或参数错误。
- `401 Unauthorized`: 未认证。
- `403 Forbidden`: 无权限。
- `404 Not Found`: 资源不存在。
- `405 Method Not Allowed`: 方法不支持。
- `500 Internal Server Error`: 服务端未知错误。

---

## 8. 不要把内部错误直接返回给用户

```go
if err != nil {
    slog.Error("database failed", "err", err)
    writeError(w, http.StatusInternalServerError, "internal_error", "internal server error")
    return
}
```

内部错误写日志，外部响应保持稳定、简洁，避免泄漏敏感细节。

---

## 9. JSON 内容类型

对于 JSON 请求，可以检查：

```go
if ct := r.Header.Get("Content-Type"); !strings.Contains(ct, "application/json") {
    writeError(w, http.StatusUnsupportedMediaType, "unsupported_media_type", "content type must be application/json")
    return
}
```

实际项目中还要考虑 `application/json; charset=utf-8` 这类形式，所以不要只做简单相等。

---

## 10. API 设计检查

1. 请求和响应结构是否分离。
2. body 是否限制大小。
3. JSON 是否禁止未知字段。
4. 字段是否做业务校验。
5. 错误响应是否统一。
6. 内部错误是否只写日志。

---

## 练习

1. 写 `writeJSON` 和 `writeError`。
2. 写 `decodeJSON`，限制 body 为 1MB。
3. 定义 `CreateUserRequest` 并实现 `Validate`。
4. 写 `POST /users` handler。
5. 对错误路径写 `httptest` 测试。
