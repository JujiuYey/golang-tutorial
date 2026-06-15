# 06: HTTP 客户端：请求、超时与连接复用

HTTP client 代码最容易漏掉两个东西：超时和关闭响应体。还要注意 client 应该复用，不要每次请求都随手创建复杂配置。

---

## 1. 使用 http.Client

```go
client := &http.Client{
    Timeout: 5 * time.Second,
}
```

`Timeout` 覆盖整个请求过程，包括连接、重定向、读取响应体。

不要在生产代码里长期使用没有超时的默认请求。

---

## 2. GET 请求

```go
func fetch(ctx context.Context, client *http.Client, rawURL string) ([]byte, error) {
    req, err := http.NewRequestWithContext(ctx, http.MethodGet, rawURL, nil)
    if err != nil {
        return nil, fmt.Errorf("create request: %w", err)
    }

    resp, err := client.Do(req)
    if err != nil {
        return nil, fmt.Errorf("do request: %w", err)
    }
    defer resp.Body.Close()

    if resp.StatusCode < 200 || resp.StatusCode >= 300 {
        return nil, fmt.Errorf("unexpected status: %s", resp.Status)
    }

    body, err := io.ReadAll(resp.Body)
    if err != nil {
        return nil, fmt.Errorf("read response body: %w", err)
    }
    return body, nil
}
```

即使状态码不是 2xx，也要关闭 body。

---

## 3. POST JSON

```go
func postJSON(ctx context.Context, client *http.Client, rawURL string, v any) error {
    var buf bytes.Buffer
    if err := json.NewEncoder(&buf).Encode(v); err != nil {
        return fmt.Errorf("encode json: %w", err)
    }

    req, err := http.NewRequestWithContext(ctx, http.MethodPost, rawURL, &buf)
    if err != nil {
        return fmt.Errorf("create request: %w", err)
    }
    req.Header.Set("Content-Type", "application/json")

    resp, err := client.Do(req)
    if err != nil {
        return fmt.Errorf("do request: %w", err)
    }
    defer resp.Body.Close()

    if resp.StatusCode < 200 || resp.StatusCode >= 300 {
        return fmt.Errorf("unexpected status: %s", resp.Status)
    }
    return nil
}
```

不要忽略 JSON 编码错误。

---

## 4. Header

```go
req.Header.Set("Authorization", "Bearer "+token)
req.Header.Set("Accept", "application/json")
```

Header key 大小写不敏感，但 Go 会规范化显示形式。

---

## 5. Query 参数

```go
u, err := url.Parse("https://api.example.com/search")
if err != nil {
    return err
}
q := u.Query()
q.Set("q", "go")
q.Set("page", "1")
u.RawQuery = q.Encode()
```

使用 `url.Values` 构造查询参数，避免手写转义。

---

## 6. 复用 Client

`http.Client` 内部会复用连接。通常应该长期复用一个 client。

```go
type APIClient struct {
    BaseURL string
    Client  *http.Client
}
```

不要每次请求都创建带自定义 Transport 的新 client，否则连接复用效果会变差。

---

## 7. Transport 超时

更细粒度控制可以配置 Transport：

```go
transport := &http.Transport{
    DialContext: (&net.Dialer{
        Timeout:   5 * time.Second,
        KeepAlive: 30 * time.Second,
    }).DialContext,
    TLSHandshakeTimeout:   5 * time.Second,
    ResponseHeaderTimeout: 5 * time.Second,
}

client := &http.Client{
    Transport: transport,
    Timeout:   10 * time.Second,
}
```

课程阶段先记住：至少要有整体 timeout。

---

## 8. 读取错误响应体

API 出错时，可以读取一小段响应体帮助定位：

```go
if resp.StatusCode >= 300 {
    data, _ := io.ReadAll(io.LimitReader(resp.Body, 4096))
    return fmt.Errorf("status %s: %s", resp.Status, strings.TrimSpace(string(data)))
}
```

使用 `io.LimitReader` 避免错误响应体过大。

---

## 9. context 取消

```go
ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
defer cancel()

req, err := http.NewRequestWithContext(ctx, http.MethodGet, rawURL, nil)
```

调用方取消 context 后，请求会尽快停止。

---

## 10. 客户端设计检查

1. 是否设置 timeout。
2. 是否使用 `NewRequestWithContext`。
3. 是否关闭 `resp.Body`。
4. 是否检查非 2xx 状态码。
5. 是否复用 client。
6. 错误里是否包含请求地址或操作上下文。

---

## 练习

1. 写带 timeout 的 GET 函数。
2. 写 POST JSON 函数，并处理编码错误。
3. 给请求加 Authorization header。
4. 用 `url.Values` 构造 query。
5. 对非 2xx 响应读取最多 4KB 错误 body。
