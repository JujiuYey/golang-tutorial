# 02: HTTP 服务端基础：Handler、ServeMux 与 Server

这篇用标准库把一个 HTTP 服务跑起来，并看懂 Go HTTP 世界的**唯一核心抽象——`http.Handler` 接口**。Express 里 handler、路由、中间件是三种东西；Go 里它们本质是同一个接口的不同玩法。理解了这一个接口，后面的中间件（05 篇）和测试（11 篇）都是顺水推舟。

> 本模块的示例统一监听 18001+ 这类高位端口，避免和你机器上已有的服务冲突。文中所有 curl 输出都是实际请求的结果（略去了变化的 `Date` 头）。

---

## 1. 先跑起来：最小 HTTP 服务

```go
package main

import (
	"fmt"
	"log"
	"net/http"
)

func hello(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintln(w, "hello")
}

func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/hello", hello)

	log.Fatal(http.ListenAndServe(":18001", mux))
}
```

另开一个终端请求它：

```bash
curl http://localhost:18001/hello
```

```text
hello
```

对照 Express 校准一下：

```text
Express:  app.get('/hello', (req, res) => res.send('hello'))
Go:       mux.HandleFunc("/hello", hello)   // hello 的签名是 (w, r)
```

🕳️ 坑：参数顺序和 Express **反着**。Express 是 `(req, res)`，Go 是 `(w http.ResponseWriter, r *http.Request)`——**响应在前，请求在后**。刚切过来的人几乎人人写反一次。

还有个眼熟的朋友：`fmt.Fprintln(w, "hello")`——F 系列打印函数写到任意 `io.Writer`，`ResponseWriter` 就是一个 Writer，往里写就是往 HTTP 响应体里写。

---

## 2. Handler 接口：整个体系的地基

```go
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}
```

就一个方法：**给你请求，你往 w 里写响应**。任何实现了 `ServeHTTP` 的类型都能处理 HTTP 请求——是函数、结构体还是别的什么，服务器不关心。

这带来一个重要推论：路由器（ServeMux）自己也实现了 `ServeHTTP`，所以**路由器也是一个 Handler**；后面会看到中间件同样是"包一层再吐出一个 Handler"。一个接口通吃全场。

---

## 3. HandlerFunc：普通函数变 Handler

普通函数没有方法，怎么满足接口？标准库定义了 `http.HandlerFunc` 类型——它底层就是函数类型，并且给自己实现了 `ServeHTTP`（内容就是调用函数自身）：

```go
func hello(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintln(w, "hello")
}

var h http.Handler = http.HandlerFunc(hello)
```

注意 `http.HandlerFunc(hello)` 是**类型转换**，不是函数调用——和 02-05 的 `[]byte(s)` 是同一种语法：把签名匹配的函数"倒进" HandlerFunc 这个容器，它就带上了 `ServeHTTP` 方法。

日常不用手动转，`mux.HandleFunc("/hello", hello)` 内部帮你转好了。**记住这层关系即可：HandleFunc 收函数，Handle 收 Handler，前者是后者的便民窗口。**

---

## 4. ServeMux：按路径分发

`ServeMux`（mux = multiplexer，多路复用器）就是路由器：看请求路径，转给对应 handler：

```go
package main

import (
	"fmt"
	"log"
	"net/http"
)

func main() {
	mux := http.NewServeMux()

	mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintln(w, "home")
	})

	mux.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintln(w, "ok")
	})

	log.Fatal(http.ListenAndServe(":18002", mux))
}
```

```bash
curl http://localhost:18002/health
curl http://localhost:18002/
curl http://localhost:18002/no-such-path
```

```text
ok
home
home
```

### 🕳️ 坑：`"/"` 不是"只匹配根路径"

以为会怎样：`/no-such-path` 没注册，应该 404。
实际怎样：返回了 `home`。
为什么：ServeMux 里以 `/` 结尾的 pattern 是**前缀匹配**，`"/"` 是所有路径的前缀，等于兜底路由。真正的 404 长这样（请求第 1 节那个只注册了 `/hello` 的服务）：

```bash
curl -i http://localhost:18001/nope
```

```text
HTTP/1.1 404 Not Found
Content-Type: text/plain; charset=utf-8
X-Content-Type-Options: nosniff
Content-Length: 19

404 page not found
```

想让 `"/"` 只管首页，用 Go 1.22+ 的 pattern `"/{$}"`，或在 handler 里检查 `r.URL.Path != "/"` 就返回 404。

---

## 5. 用 http.Server 启动

`http.ListenAndServe(addr, mux)` 是快捷方式。正式一点的写法是自己构造 `http.Server`：

```go
server := &http.Server{
    Addr:    ":18002",
    Handler: mux,
}

if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
    return err
}
```

两个细节：

- `ListenAndServe` 是**阻塞**的（像 Node 里 `server.listen` 之后进程不退出），只有出错或被关闭才返回。
- 别忽略它的 error——端口被占用时错误是 `listen tcp :18002: bind: address already in use`，吞掉它你会对着"没反应的服务"发呆。正常关闭时它返回 `http.ErrServerClosed`，所以上面把这个"正常错误"排除掉。

---

## 6. 配置超时

第 01 篇的铁律在服务端的落地。`http.Server` 默认**没有任何超时**——一个建了连接却慢慢发数据的客户端能无限期占住资源：

```go
server := &http.Server{
    Addr:              ":18002",
    Handler:           mux,
    ReadHeaderTimeout: 5 * time.Second,  // 读完请求头的时限
    ReadTimeout:       10 * time.Second, // 读完整个请求的时限
    WriteTimeout:      10 * time.Second, // 写完响应的时限
    IdleTimeout:       60 * time.Second, // keep-alive 空闲连接存活时限
}
```

不用背全。**最低要求记一条：生产服务至少配 `ReadHeaderTimeout`**，它挡住最经典的慢连接攻击（slowloris）。

---

## 7. ResponseWriter：写响应有顺序

写响应的三步有严格顺序，**因为 HTTP 报文在网络上就是按这个顺序发出去的**：

```text
① w.Header().Set(...)      // 攒 header（还没发送）
② w.WriteHeader(status)    // 发出状态行 + header，从此不可反悔
③ w.Write(body)            // 发 body（如果没做②，这步自动补一个 200）
```

```go
func hello(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "text/plain; charset=utf-8")
	w.WriteHeader(http.StatusOK)
	if _, err := w.Write([]byte("hello")); err != nil {
		slog.Error("write response failed", "err", err)
	}
}
```

### 🕳️ 坑：先写 body 再设状态码，静默失效

Express 里 `res.send()` 之后再 `res.status()` 也没用，Go 同样——但 Go 会在服务端日志里留个条子：

```go
func wrong(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("hello\n"))
	w.WriteHeader(http.StatusCreated) // 太晚了
}
```

```bash
curl -i http://localhost:18003/wrong
```

```text
HTTP/1.1 200 OK
Content-Length: 6
Content-Type: text/plain; charset=utf-8

hello
```

客户端拿到的是 200 不是 201。同时服务端日志出现：

```text
2026/07/16 10:14:59 http: superfluous response.WriteHeader call from main.wrong (a02_order.go:10)
```

看到 `superfluous response.WriteHeader` 就知道：某个 handler 的写响应顺序反了。

---

## 8. Request：请求信息都在这

`*http.Request` 相当于 Express 的 `req`：方法、URL、header、body、context 全在里面：

```go
func inspect(w http.ResponseWriter, r *http.Request) {
	fmt.Println(r.Method)
	fmt.Println(r.URL.Path)
	fmt.Println(r.Header.Get("User-Agent"))
	fmt.Fprintln(w, "logged")
}
```

`curl http://localhost:18005/inspect` 之后，服务端终端打印：

```text
GET
/inspect
curl/8.7.1
```

注意 `fmt.Println` 打到**服务端终端**，`fmt.Fprintln(w, ...)` 才是发给**客户端**——一个字母 F 的差别，写岔了就是"响应怎么是空的"。详细字段下一篇展开。

---

## 9. 方法限制：不支持就 405

一个 handler 默认什么方法都接（GET、POST、DELETE 全进同一个函数）。要限制就自己检查：

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

```bash
curl -i -X POST http://localhost:18004/data
```

```text
HTTP/1.1 405 Method Not Allowed
Allow: GET
Content-Type: text/plain; charset=utf-8
X-Content-Type-Options: nosniff
Content-Length: 19

method not allowed
```

规矩两条：**返回 405（不是 404），并用 `Allow` 头告诉对方支持什么**。方法名用常量 `http.MethodGet`，别手写 `"GET"`（写成 `"get"` 编译器不会救你）。Go 1.22+ 的 mux 能在注册时直接写 `"GET /data"`，见第 04 篇。

---

## 10. 两个工程习惯：优雅关闭与自建 mux

**优雅关闭**：生产服务收到退出信号时，不能拔电源式地掐断正在处理的请求。`http.Server` 提供 `Shutdown`：

```go
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

if err := server.Shutdown(ctx); err != nil {
    return err
}
```

它停止接新请求、等在途请求做完（最多等 ctx 给的 5 秒）。完整的信号处理在工程化章节展开，这里先知道有这个门。

**别用默认 mux 做正式服务**：`http.HandleFunc("/x", h)`（没有 mux 前缀）注册到的是包级全局的 `http.DefaultServeMux`。小 demo 无所谓；正式项目自己 `http.NewServeMux()` 并显式传给 `http.Server`——依赖清楚，测试时也能整个 mux 拿去测（第 11 篇）。全局变量的老毛病，哪门语言都一样。

---

## 本篇重点

- [ ] Go HTTP 的一切围绕 `http.Handler` 接口（一个 `ServeHTTP(w, r)` 方法）；mux、中间件本质都是 Handler。
- [ ] handler 参数是 `(w, r)`——和 Express 的 `(req, res)` 顺序相反；`Fprintln(w,...)` 发给客户端，`Println` 打在服务端。
- [ ] `mux.HandleFunc` 收函数、`mux.Handle` 收 Handler；`http.HandlerFunc(f)` 是类型转换不是调用。
- [ ] `"/"` 是前缀兜底路由，不是"仅根路径"；写响应顺序 header → status → body，反了就静默失效（日志报 superfluous WriteHeader）。
- [ ] `http.Server` 默认无超时，至少配 `ReadHeaderTimeout`；正式服务别用默认全局 mux。

---

## 练习

1. 用 `http.NewServeMux` 写 `/` 和 `/health`，并验证第 4 节的坑：请求一个没注册的路径看返回什么。
2. 用 `http.Server` 启动服务，并配置 `ReadHeaderTimeout`。
3. 写一个只允许 GET 的 handler，用 `curl -i -X POST` 验证 405 和 `Allow` 头。
4. 写一个自定义结构体类型实现 `ServeHTTP`（提示：方法接收者随意，方法体里能用结构体字段，比如返回固定的问候语）。
5. 说明设置 header、status、body 的顺序，以及顺序反了会发生什么。
