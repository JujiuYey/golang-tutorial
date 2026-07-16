# 06: HTTP 客户端：请求、超时与连接复用

换个方向：这篇解决"Go 里怎么当调用方"，也就是 JS 里 `fetch` 干的活。Go 客户端代码最常见的两个事故是**没设超时**和**忘关响应体**——前者让程序卡死，后者让连接泄漏。学完这篇你能写出一个"有超时、会关 body、能查状态码、可复用"的标准请求函数。

> 本篇示例请求的是本地测试服务 `localhost:18013`（上面挂了 `/user`、`/slow`、`/fail` 几个接口），所有输出都是实跑结果。

---

## 1. 使用 http.Client：先把超时焊死

```go
client := &http.Client{
    Timeout: 5 * time.Second,
}
```

`Timeout` 罩住**整个请求过程**：建连接、跟重定向、读完响应体，全算在内。

🕳️ 坑：`http.Get(url)` 一行党用的是 `http.DefaultClient`——它**没有任何超时**，和浏览器 `fetch` 一样能无限等下去。JS 里你要靠 `AbortController` 补救，Go 里就一句话：**自己 new client，第一件事设 Timeout。**

看超时真的生效：请求一个要睡 1 秒的接口，client 只给 100ms：

```go
client := &http.Client{Timeout: 100 * time.Millisecond}

req, _ := http.NewRequestWithContext(context.Background(),
    http.MethodGet, "http://localhost:18013/slow", nil)

_, err := client.Do(req)
fmt.Println(err)
```

```text
Get "http://localhost:18013/slow": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
```

错误信息把"哪个 URL、超在哪个阶段"都告诉你了。

---

## 2. GET 请求：标准模板

这是本篇的核心函数，五个动作一个不能少：

```go
func fetch(ctx context.Context, client *http.Client, rawURL string) ([]byte, error) {
	req, err := http.NewRequestWithContext(ctx, http.MethodGet, rawURL, nil) // ①带 ctx 建请求
	if err != nil {
		return nil, fmt.Errorf("create request: %w", err)
	}

	resp, err := client.Do(req) // ②发出去
	if err != nil {
		return nil, fmt.Errorf("do request: %w", err)
	}
	defer resp.Body.Close() // ③预约关 body(03-07 的套路)

	if resp.StatusCode < 200 || resp.StatusCode >= 300 { // ④检查状态码
		return nil, fmt.Errorf("unexpected status: %s", resp.Status)
	}

	body, err := io.ReadAll(resp.Body) // ⑤读 body
	if err != nil {
		return nil, fmt.Errorf("read response body: %w", err)
	}
	return body, nil
}
```

```go
body, err := fetch(context.Background(), client, "http://localhost:18013/user")
fmt.Print(string(body))
```

```text
{"id":1,"name":"alice"}
```

两个和 `fetch` 的关键对照：

- **404/500 不是 error**——和 JS 的 `fetch` 一样（它也只在网络层面失败时 reject）。`client.Do` 只要收到响应就算成功，状态码要自己查，等价于 JS 的 `if (!resp.ok)`。
- **`resp.Body` 必须 Close**——JS 里没这回事（GC 收拾），Go 里不关的后果是底层 TCP 连接没法回池复用，泄漏攒多了连接耗尽。`defer resp.Body.Close()` 紧跟错误检查之后，肌肉记忆。

🕳️ 坑：`defer resp.Body.Close()` 不能写在 `if err != nil` 之前——`Do` 失败时 `resp` 是 nil，Close 一个 nil 直接 panic。先确认资源到手，再预约清理（03-07 原话）。另外**非 2xx 也要关 body**，上面的写法 defer 在检查状态码之前就注册了，正好覆盖。

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
	fmt.Println("状态:", resp.Status)
	return nil
}
```

```go
user := map[string]any{"name": "bob", "age": 30}
postJSON(context.Background(), client, "http://localhost:18013/users", user)
```

```text
状态: 201 Created
```

服务端那头确实收到了：

```text
2026/07/16 10:25:06 server got: map[age:30 name:bob] (Content-Type: application/json)
```

对比 JS 的 `fetch(url, {method:'POST', body: JSON.stringify(v), headers:{...}})`，Go 多两件事：JSON 编码错误要接住（别 `_` 吞掉），`Content-Type` 要手动设——**忘设它是 POST JSON 打不通的第一嫌疑人**。

---

## 4. Header

```go
req.Header.Set("Authorization", "Bearer "+token)
req.Header.Set("Accept", "application/json")
```

Header key 大小写不敏感，Go 存储时会规范化成 `Authorization` 这种形式（每段首字母大写）。设 header 必须在 `client.Do` 之前——请求发出去就晚了。

---

## 5. Query 参数

第 01 篇的 `url.Values` 在这派上用场：

```go
u, err := url.Parse("http://localhost:18013/search")
if err != nil {
    return err
}
q := u.Query()
q.Set("q", "go")
q.Set("page", "1")
u.RawQuery = q.Encode()
// u.String() 作为请求 URL
```

规矩不变：**转义交给库，别手拼 `"?q=" + keyword`**——用户输入里一个 `&` 就把你的参数结构拆了。

---

## 6. 复用 Client

`http.Client` 内部有连接池，同一个 host 的请求会复用 TCP 连接（省掉握手）。所以：

```go
type APIClient struct {
    BaseURL string
    Client  *http.Client // 整个程序生命周期就这一个
}
```

**client 造一次、用到死**；每次请求都 new 一个（尤其是带自定义 Transport 的），等于每次都开新连接池，复用完全泡汤。JS 里没有这个心智负担（浏览器/Node 管连接池），Go 里连接池就挂在 client 身上，你得自己别把它扔了。

**一句话总结：`http.Client` 是可以并发安全共享的重资产，不是一次性筷子。**

---

## 7. Transport 超时

想更细地控制每个阶段，配 Transport：

```go
transport := &http.Transport{
    DialContext: (&net.Dialer{
        Timeout:   5 * time.Second,  // 建连接
        KeepAlive: 30 * time.Second,
    }).DialContext,
    TLSHandshakeTimeout:   5 * time.Second, // TLS 握手
    ResponseHeaderTimeout: 5 * time.Second, // 等响应头
}

client := &http.Client{
    Transport: transport,
    Timeout:   10 * time.Second, // 整体兜底
}
```

分阶段超时用于精细排障和高要求服务。课程阶段的底线只有一条：**至少要有整体 Timeout**，其余以后按需加。

---

## 8. 读取错误响应体

API 报错时 body 里常有有用信息（错误码、消息）。读一小段放进 error，排障效率翻倍：

```go
if resp.StatusCode >= 300 {
    data, _ := io.ReadAll(io.LimitReader(resp.Body, 4096)) // 最多读 4KB
    return fmt.Errorf("status %s: %s", resp.Status, strings.TrimSpace(string(data)))
}
```

对一个返回 500 + JSON 错误体的接口实跑：

```text
status 500 Internal Server Error: {"code":"db_down","message":"database unavailable"}
```

比光秃秃的 `unexpected status: 500` 有用多了。`io.LimitReader` 是安全阀：错误页有时是整个 HTML 报错页，不设限等于又开了一个"来多少吃多少"的口子。

---

## 9. context 取消

`Timeout` 是 client 级的一刀切；context 允许**每次调用**单独给期限，还能被上游（比如第 03 篇 handler 的 `r.Context()`）联动取消：

```go
ctx, cancel := context.WithTimeout(context.Background(), 200*time.Millisecond)
defer cancel()

req, _ := http.NewRequestWithContext(ctx,
    http.MethodGet, "http://localhost:18013/slow", nil)

_, err := http.DefaultClient.Do(req)
fmt.Println(err)
```

```text
Get "http://localhost:18013/slow": context deadline exceeded
```

这就是"链路取消"的闭环：用户关了浏览器 → 你的 handler ctx 取消 → 你发给下游的请求跟着取消，谁都不白干。所以模板里坚持用 `NewRequestWithContext` 而不是 `NewRequest`。

---

## 10. 客户端设计检查

1. client 是否设置了 Timeout？
2. 是否用 `NewRequestWithContext` 把 ctx 传进去？
3. `resp.Body` 是否在所有路径上都会 Close（包括非 2xx）？
4. 是否检查了状态码（记住 404 不是 error）？
5. client 是否长期复用（不是每次请求 new 一个）？
6. 错误里是否带了 URL/状态码/响应体片段这些上下文？

---

## 本篇重点

- [ ] `http.Get`/DefaultClient 没有超时；自己 new client 并设 `Timeout` 是第一动作。
- [ ] 和 JS fetch 一样，404/500 **不是 error**——状态码必须自己检查。
- [ ] `defer resp.Body.Close()` 写在 `Do` 的错误检查之后、状态码检查之前；不关 body = 连接泄漏。
- [ ] client 是带连接池的重资产，造一次全程复用；POST JSON 别忘了 `Content-Type`。
- [ ] 用 `NewRequestWithContext` 接入取消链路；出错时用 `LimitReader` 读一小段错误体放进 error。

---

## 练习

1. 写带 timeout 的 GET 函数，用一个故意睡 1 秒的本地接口验证超时错误。
2. 写 POST JSON 函数，并处理编码错误。
3. 给请求加 Authorization header。
4. 用 `url.Values` 构造 query。
5. 对非 2xx 响应读取最多 4KB 错误 body 并包进 error（提示：`io.LimitReader(resp.Body, 4096)`）。
