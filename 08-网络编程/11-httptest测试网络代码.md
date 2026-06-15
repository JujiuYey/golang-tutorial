# 11: httptest：测试 Handler 和 HTTP Client

网络代码不应该依赖真实外部服务才能测试。标准库 `net/http/httptest` 可以在内存里测试 handler，也可以启动临时测试服务器。

---

## 1. 测试 Handler

```go
func TestHealth(t *testing.T) {
    req := httptest.NewRequest(http.MethodGet, "/health", nil)
    rr := httptest.NewRecorder()

    health(rr, req)

    if rr.Code != http.StatusOK {
        t.Fatalf("status = %d, want %d", rr.Code, http.StatusOK)
    }
}
```

`NewRecorder` 记录 handler 写出的响应。

---

## 2. 检查响应体

```go
got := strings.TrimSpace(rr.Body.String())
if got != `{"status":"ok"}` {
    t.Fatalf("body = %q", got)
}
```

JSON 响应更稳的做法是解码后比较结构体。

---

## 3. 测试 JSON handler

```go
body := strings.NewReader(`{"name":"alice","age":18}`)
req := httptest.NewRequest(http.MethodPost, "/users", body)
req.Header.Set("Content-Type", "application/json")
rr := httptest.NewRecorder()

createUser(rr, req)

if rr.Code != http.StatusCreated {
    t.Fatalf("status = %d", rr.Code)
}
```

---

## 4. 测试错误路径

```go
req := httptest.NewRequest(http.MethodPost, "/users", strings.NewReader(`{bad json`))
rr := httptest.NewRecorder()

createUser(rr, req)

if rr.Code != http.StatusBadRequest {
    t.Fatalf("status = %d", rr.Code)
}
```

网络代码的错误路径很重要，不能只测 happy path。

---

## 5. httptest.Server

```go
server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintln(w, `{"status":"ok"}`)
}))
defer server.Close()

resp, err := http.Get(server.URL)
if err != nil {
    t.Fatal(err)
}
defer resp.Body.Close()
```

`httptest.Server` 会启动一个真实 HTTP server，只是地址和端口由测试管理。

---

## 6. 测试 HTTP client

```go
func TestClient(t *testing.T) {
    server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if r.URL.Path != "/users" {
            t.Fatalf("path = %s", r.URL.Path)
        }
        fmt.Fprintln(w, `{"id":1,"name":"alice"}`)
    }))
    defer server.Close()

    client := &APIClient{
        BaseURL: server.URL,
        Client:  server.Client(),
    }

    user, err := client.GetUser(context.Background(), 1)
    if err != nil {
        t.Fatal(err)
    }
    if user.Name != "alice" {
        t.Fatalf("user = %+v", user)
    }
}
```

测试 client 时，把 base URL 指向测试服务器。

---

## 7. 模拟超时

```go
server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    time.Sleep(200 * time.Millisecond)
}))
defer server.Close()

client := &http.Client{Timeout: 50 * time.Millisecond}
_, err := client.Get(server.URL)
if err == nil {
    t.Fatal("expected timeout")
}
```

超时测试要留出合理时间，避免测试不稳定。

---

## 8. 测试中间件

```go
handler := Logger(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusNoContent)
}))

req := httptest.NewRequest(http.MethodGet, "/", nil)
rr := httptest.NewRecorder()

handler.ServeHTTP(rr, req)
```

中间件本质也是 handler，所以测试方式一样。

---

## 9. 不依赖外部网络

单元测试不要请求真实的 `https://api.example.com`。外部网络会带来：

- 慢。
- 不稳定。
- 需要密钥。
- 受第三方服务状态影响。

使用 `httptest.Server` 可以把测试控制在本地。

---

## 10. 测试检查

1. 正常响应是否测试。
2. 错误响应是否测试。
3. method 不匹配是否测试。
4. JSON 解码失败是否测试。
5. client 超时是否测试。
6. 是否避免真实外部网络。

---

## 练习

1. 用 `httptest.NewRecorder` 测试 `/health`。
2. 测试 JSON 创建接口的成功路径。
3. 测试 JSON 解码失败路径。
4. 用 `httptest.Server` 测试 HTTP client。
5. 写一个超时测试。
