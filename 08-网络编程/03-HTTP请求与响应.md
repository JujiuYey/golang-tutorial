# 03: HTTP 请求与响应：method、header、body、status

上一篇把服务跑起来了，这篇解决 handler 里的日常活：把请求的每个部分（method、查询参数、header、body）读出来，把响应的每个部分（状态码、header、body）正确写回去。Express 里 `req.query`、`req.body`、`res.json()` 帮你做掉的事，这里要自己动手——好处是每一步的边界（body 多大算大？状态码什么时候发出？）都在你手里。

---

## 1. 读取 Method 和 Path

```go
func handler(w http.ResponseWriter, r *http.Request) {
	fmt.Println(r.Method)   // "GET"
	fmt.Println(r.URL.Path) // "/users/42"
}
```

方法名一律用标准库常量，别手写字符串：

```go
http.MethodGet    // "GET"
http.MethodPost   // "POST"
http.MethodPut    // "PUT"
http.MethodPatch  // "PATCH"
http.MethodDelete // "DELETE"
```

写错常量名是编译错误，写错字符串 `"GTE"` 只能等运行时诡异 bug。老规矩：**能让编译器帮忙的就让编译器帮忙。**

---

## 2. 查询参数

Express 里是 `req.query.q`；Go 里从 `r.URL.Query()` 拿（就是第 01 篇的 `url.Values`）：

```go
func search(w http.ResponseWriter, r *http.Request) {
	q := r.URL.Query()
	keyword := q.Get("q")
	page := q.Get("page")
	tags := q["tag"] // 同名 key 的所有值
	fmt.Fprintf(w, "keyword=%q page=%q tags=%v\n", keyword, page, tags)
}
```

```bash
curl 'http://localhost:18010/search?q=go&page=1&tag=web&tag=http'
```

```text
keyword="go" page="1" tags=[web http]
```

两个注意点：

- `Get` 只返回第一个值；同名多值（`tag=web&tag=http`）要用 map 索引 `q["tag"]` 拿切片。
- **查询参数永远是字符串**。`page` 要当数字用就得 `strconv.Atoi(page)` 并处理 error——用户传 `?page=abc` 是常态，不是意外。参数不存在时 `Get` 返回空字符串（不是 JS 的 `undefined`），"没传"和"传了空值"分不清，介意就用 `q.Has("page")`。

---

## 3. Header

读请求头（对应 Express 的 `req.get(...)`）：

```go
token := r.Header.Get("Authorization")
contentType := r.Header.Get("Content-Type")
```

带上和不带 header 各请求一次：

```bash
curl -i http://localhost:18010/auth
```

```text
HTTP/1.1 401 Unauthorized
Content-Type: text/plain; charset=utf-8
X-Content-Type-Options: nosniff
Content-Length: 14

missing token
```

```bash
curl -H 'Authorization: Bearer abc123' http://localhost:18010/auth
```

```text
token=Bearer abc123
```

`Header.Get` 对大小写不敏感（`authorization` 也能取到），不存在时同样返回空字符串。写响应头是 `w.Header().Set(...)`——记住上一篇的顺序规则：**必须在写状态码和 body 之前**。

---

## 4. 读取 Body

Express 有 body-parser 中间件；Go 里 `r.Body` 是一条原始的字节流（`io.Reader`），自己读：

```go
func body(w http.ResponseWriter, r *http.Request) {
	r.Body = http.MaxBytesReader(w, r.Body, 16) // 演示用,只准 16 字节

	data, err := io.ReadAll(r.Body)
	if err != nil {
		http.Error(w, err.Error(), http.StatusBadRequest)
		return
	}
	fmt.Fprintf(w, "got %d bytes: %s\n", len(data), data)
}
```

```bash
curl -d 'hello' http://localhost:18010/body
```

```text
got 5 bytes: hello
```

超限的请求会在读取时直接报错：

```bash
curl -i -d 'this body is definitely longer than sixteen bytes' http://localhost:18010/body
```

```text
HTTP/1.1 400 Bad Request
Connection: close
Content-Type: text/plain; charset=utf-8
X-Content-Type-Options: nosniff
Content-Length: 29

http: request body too large
```

🕳️ 坑：不加 `MaxBytesReader` 的 `io.ReadAll(r.Body)` 是"来多少吃多少"——对方发个 2GB 的 body，你的内存就替他买单。**所有会读 body 的接口都要设上限**（正常业务给 1MB：`1<<20`）。另外服务端 handler 里请求体的生命周期由服务器管理，不像客户端那样必须手动 Close（第 06 篇的 `resp.Body.Close()` 才是必须的）。

---

## 5. 写状态码

```go
w.WriteHeader(http.StatusCreated) // 201
```

规则就两条，都在上一篇第 7 节验证过：

- 不显式调用 `WriteHeader`，第一次写 body 时自动发送 `200 OK`——所以纯 200 响应直接写 body 即可。
- 状态码只能有效发送**一次**，后续调用只会在日志里留 `superfluous response.WriteHeader` 警告，客户端看到的还是第一个。

**一句话总结：非 200 才需要 `WriteHeader`，并且要赶在任何 body 之前。**

---

## 6. http.Error

错误响应的快捷方式，一行搞定"设状态码 + 写一行文本"：

```go
http.Error(w, "not found", http.StatusNotFound)
```

```bash
curl -i http://localhost:18010/missing
```

```text
HTTP/1.1 404 Not Found
Content-Type: text/plain; charset=utf-8
X-Content-Type-Options: nosniff
Content-Length: 10

not found
```

注意它发的是 `text/plain`。写 JSON API 时前端要的是 JSON 格式的错误——所以实际项目会自己封装一个 JSON 版的错误函数（第 07 篇），`http.Error` 用于简单场景。**别忘了它不会替你 `return`**——调用完必须自己 `return`，否则代码继续往下跑，又往响应里追加内容。

---

## 7. 写 JSON 响应

Express 的 `res.status(200).json(user)`，Go 里自己拼三步——正好按第 5 节的顺序：

```go
func writeJSON(w http.ResponseWriter, status int, v any) error {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	return json.NewEncoder(w).Encode(v)
}
```

```bash
curl -i http://localhost:18010/user
```

```text
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 24

{"id":1,"name":"alice"}
```

`json.NewEncoder(w).Encode(v)` 直接把 JSON 编码进响应流，不需要先攒成字符串。handler 里这样用：

```go
if err := writeJSON(w, http.StatusOK, user); err != nil {
    // 状态码和部分数据可能已经发出去了,没法改口,只能记日志
    slog.Error("write json failed", "err", err)
}
```

这个 `writeJSON` 会一路用到第 07 篇，先把它焊进肌肉记忆。

---

## 8. 请求 Context

每个请求自带一个 context：

```go
ctx := r.Context()
```

客户端断开、请求超时、服务端关闭，这个 ctx 都会被取消——相当于 Node 里 `req.on('close')` / `AbortSignal` 的统一版。做个实验：handler 要干 5 秒的活，客户端 1 秒就跑路：

```go
func slow(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()
	select {
	case <-time.After(5 * time.Second):
		fmt.Fprintln(w, "done")
	case <-ctx.Done():
		log.Println("client gone:", ctx.Err())
	}
}
```

```bash
curl --max-time 1 http://localhost:18010/slow
```

```text
curl: (28) Operation timed out after 1002 milliseconds with 0 bytes received
```

服务端日志同一时刻打出：

```text
2026/07/16 10:18:16 client gone: context canceled
```

客户端一挂断，服务端立刻知道，不用傻等 5 秒。**实践规则：handler 里发起的数据库查询、下游 HTTP 请求、耗时任务，都把 `r.Context()` 传下去**——客户端都不要结果了，下游也别白干。

---

## 9. RemoteAddr 和 Host

```go
func addr(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintln(w, "RemoteAddr:", r.RemoteAddr)
	fmt.Fprintln(w, "Host:", r.Host)
}
```

```bash
curl http://localhost:18010/addr
```

```text
RemoteAddr: [::1]:61334
Host: localhost:18010
```

`RemoteAddr` 是对端的"IP:端口"（这里 `[::1]` 是 IPv6 的本机地址，端口每次连接都不同）。

🕳️ 坑：服务前面有 Nginx 等反向代理时，`RemoteAddr` 拿到的是**代理的 IP**，不是真实用户的。真实 IP 可能在 `X-Forwarded-For` header 里——但这个 header 任何客户端都能伪造，**只有确认代理配置可信时才能用它**。不了解部署拓扑就盲信这类 header，等于让人随便冒充 IP。

---

## 10. 响应设计检查

写完一个 handler，对着清单过一遍：

1. 方法不匹配时是否返回 405（带 `Allow` 头）？
2. 会读 body 的接口是否用 `MaxBytesReader` 限了大小？
3. JSON 解码失败是否返回 400？
4. 成功创建资源是否返回 201？
5. 错误响应格式是否统一（别一半 text/plain 一半 JSON）？

这五条会在第 07 篇 JSON API 里逐条落地。

---

## 本篇重点

- [ ] 查询参数、header 取不到时是**空字符串**不是 undefined；参数全是字符串，转数字要 `strconv.Atoi` + 处理 error。
- [ ] 读 body 必配 `http.MaxBytesReader` 设上限，超限报 `http: request body too large`。
- [ ] 状态码只发一次：非 200 用 `WriteHeader`，且必须在 body 之前；`http.Error` 之后记得 `return`。
- [ ] `writeJSON` 三步：设 Content-Type → 写状态码 → `json.NewEncoder(w).Encode(v)`。
- [ ] `r.Context()` 在客户端断开时自动取消，凡是下游调用都把它传下去；代理后面的 `RemoteAddr` 不是真实用户 IP。

---

## 练习

1. 写 handler 读取 `?q=go&page=1`，page 转成整数并处理非法输入。
2. 写 handler 检查 `Authorization` header，缺失时返回 401。
3. 写 handler 限制 body 最大 1MB，用 curl 验证正常和超限两种情况（提示：造大文件可以用 `head -c 2000000 /dev/zero > big.bin` 再 `curl --data-binary @big.bin`）。
4. 写 `writeJSON(w, status, v)` 辅助函数并用 `curl -i` 检查 Content-Type。
5. 写一个统一 JSON 错误响应函数（提示：复用你的 `writeJSON`，包一个含 code 和 message 字段的结构体）。
