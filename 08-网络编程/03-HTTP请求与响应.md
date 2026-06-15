# 03: HTTP 请求与响应：method、header、body、status

HTTP handler 的核心工作是读取请求、执行业务逻辑、写出响应。每一步都要处理边界。

---

## 1. 读取 Method 和 Path

```go
func handler(w http.ResponseWriter, r *http.Request) {
    fmt.Println(r.Method)
    fmt.Println(r.URL.Path)
}
```

常见 method：

- `GET`
- `POST`
- `PUT`
- `PATCH`
- `DELETE`

使用标准库常量：

```go
http.MethodGet
http.MethodPost
```

---

## 2. 查询参数

```go
q := r.URL.Query()
keyword := q.Get("q")
page := q.Get("page")
```

`Get` 只返回第一个值。如果同一个 key 有多个值：

```go
values := q["tag"]
```

查询参数都是字符串。需要数字时要用 `strconv.Atoi` 并处理错误。

---

## 3. Header

```go
token := r.Header.Get("Authorization")
contentType := r.Header.Get("Content-Type")
```

写响应 header：

```go
w.Header().Set("Content-Type", "application/json")
```

响应 header 要在写状态码和 body 之前设置。

---

## 4. 读取 Body

```go
defer r.Body.Close()

data, err := io.ReadAll(r.Body)
if err != nil {
    http.Error(w, "read body failed", http.StatusBadRequest)
    return
}
```

服务端 handler 中，服务器会管理请求体生命周期。显式关闭也可以，但更重要的是不要无限制读取大 body。

可以用 `http.MaxBytesReader` 限制大小：

```go
r.Body = http.MaxBytesReader(w, r.Body, 1<<20)
```

---

## 5. 写状态码

```go
w.WriteHeader(http.StatusCreated)
```

如果没有显式调用 `WriteHeader`，第一次写 body 时默认发送 `200 OK`。

同一个响应只能有效写一次状态码。后续再调用 `WriteHeader` 不会改变已发送状态。

---

## 6. http.Error

```go
http.Error(w, "not found", http.StatusNotFound)
```

`http.Error` 会设置状态码和 text/plain 响应体。

JSON API 中通常使用自己的 JSON 错误响应函数，而不是直接用 `http.Error`。

---

## 7. 写 JSON 响应

```go
func writeJSON(w http.ResponseWriter, status int, v any) error {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    return json.NewEncoder(w).Encode(v)
}
```

handler 里可以这样用：

```go
if err := writeJSON(w, http.StatusOK, user); err != nil {
    // 响应可能已经开始写出，通常只能记录日志
}
```

---

## 8. 请求 Context

```go
ctx := r.Context()
```

客户端断开连接、请求超时、服务端关闭时，context 可能被取消。下游数据库、HTTP 请求、耗时任务应该继续传递这个 context。

---

## 9. RemoteAddr 和 Host

```go
fmt.Println(r.RemoteAddr)
fmt.Println(r.Host)
```

如果服务前面有反向代理，真实客户端 IP 可能在 `X-Forwarded-For` 等 header 中。不要在不了解代理配置时盲目信任这些 header。

---

## 10. 响应设计检查

写 handler 时问：

1. 方法不匹配时是否返回 405。
2. 请求体过大时是否有限制。
3. JSON 解码失败时是否返回 400。
4. 成功创建资源是否返回 201。
5. 错误响应格式是否统一。

---

## 练习

1. 写 handler 读取 `?q=go&page=1`。
2. 写 handler 检查 `Authorization` header。
3. 写 handler 限制 body 最大 1MB。
4. 写 `writeJSON(w, status, v)` 辅助函数。
5. 写一个统一 JSON 错误响应函数。
