# 02: HTTP 服务端基础：Handler、ServeMux 与 Server

Go 的 HTTP 服务端核心是 `http.Handler` 接口。理解这个接口，就能理解 handler、mux、中间件和测试。

---

## 1. Handler 接口

```go
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}
```

任何实现 `ServeHTTP` 方法的类型，都可以处理 HTTP 请求。

---

## 2. HandlerFunc

普通函数可以通过 `http.HandlerFunc` 变成 handler。

```go
func hello(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintln(w, "hello")
}

var h http.Handler = http.HandlerFunc(hello)
```

常见写法：

```go
mux.HandleFunc("/hello", hello)
```

---

## 3. ServeMux

```go
mux := http.NewServeMux()

mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintln(w, "home")
})

mux.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintln(w, "ok")
})
```

`ServeMux` 根据请求路径把请求分发给不同 handler。

---

## 4. 启动服务

```go
server := &http.Server{
    Addr:    ":8080",
    Handler: mux,
}

if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
    return err
}
```

不要忽略 `ListenAndServe` 返回的 error。

---

## 5. 配置超时

```go
server := &http.Server{
    Addr:              ":8080",
    Handler:           mux,
    ReadHeaderTimeout: 5 * time.Second,
    ReadTimeout:       10 * time.Second,
    WriteTimeout:      10 * time.Second,
    IdleTimeout:       60 * time.Second,
}
```

服务端超时可以降低慢连接拖垮服务的风险。

---

## 6. ResponseWriter

```go
func hello(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "text/plain; charset=utf-8")
    w.WriteHeader(http.StatusOK)
    if _, err := w.Write([]byte("hello")); err != nil {
        slog.Error("write response failed", "err", err)
    }
}
```

写响应时注意顺序：

1. 设置 header。
2. 写 status。
3. 写 body。

一旦开始写 body，状态码和部分 header 就可能已经发送。

---

## 7. Request

```go
func inspect(w http.ResponseWriter, r *http.Request) {
    fmt.Println(r.Method)
    fmt.Println(r.URL.Path)
    fmt.Println(r.Header.Get("User-Agent"))
}
```

`*http.Request` 保存请求方法、URL、header、body、context 等信息。

---

## 8. 方法限制

```go
func onlyGet(w http.ResponseWriter, r *http.Request) {
    if r.Method != http.MethodGet {
        w.Header().Set("Allow", http.MethodGet)
        http.Error(w, "method not allowed", http.StatusMethodNotAllowed)
        return
    }
    fmt.Fprintln(w, "ok")
}
```

handler 要明确接受哪些方法。不支持的方法返回 405。

---

## 9. 优雅关闭预告

生产服务通常需要优雅关闭：

```go
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

if err := server.Shutdown(ctx); err != nil {
    return err
}
```

完整信号处理会在工程化章节展开。这里先知道 `http.Server` 支持 `Shutdown`。

---

## 10. 不要使用默认 mux 做复杂服务

`http.HandleFunc` 使用包级默认 mux。小示例可以用，真实项目建议自己创建 `http.NewServeMux()`，显式传给 `http.Server`。

这样依赖更清楚，也更容易测试。

---

## 练习

1. 用 `http.NewServeMux` 写 `/` 和 `/health`。
2. 用 `http.Server` 启动服务，并配置 `ReadHeaderTimeout`。
3. 写一个只允许 GET 的 handler。
4. 写一个自定义类型实现 `ServeHTTP`。
5. 说明设置 header、status、body 的顺序。
