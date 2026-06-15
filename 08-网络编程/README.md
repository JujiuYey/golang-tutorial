# 08: 网络编程

本模块讲 Go 标准库中的网络编程。重点是 HTTP 服务端、HTTP 客户端、URL、请求响应、超时、context、JSON API、测试，以及 TCP/UDP 的基础用法。

---

## 文章列表

1. [网络编程基础：地址、端口、URL 与 DNS](./01-网络编程基础.md)
2. [HTTP 服务端基础：Handler、ServeMux 与 Server](./02-HTTP服务端基础.md)
3. [HTTP 请求与响应：method、header、body、status](./03-HTTP请求与响应.md)
4. [路由、路径参数与查询参数](./04-路由路径参数与查询参数.md)
5. [中间件、日志、恢复与请求 ID](./05-中间件日志恢复与请求ID.md)
6. [HTTP 客户端：请求、超时与连接复用](./06-HTTP客户端请求超时与连接复用.md)
7. [JSON API：编码、解码、错误响应与校验](./07-JSON-API编码解码与错误响应.md)
8. [表单、文件上传与静态文件](./08-表单文件上传与静态文件.md)
9. [TCP 编程：Listener、Conn 与协议边界](./09-TCP编程.md)
10. [UDP 编程：数据报与丢包边界](./10-UDP编程.md)
11. [httptest：测试 Handler 和 HTTP Client](./11-httptest测试网络代码.md)
12. [网络编程综合练习](./12-网络编程综合练习.md)

---

## 学习目标

完成本模块后，你应该能够：

- 说清楚 URL、host、port、path、query 的基本结构。
- 使用 `http.Handler`、`http.HandlerFunc`、`http.ServeMux` 组织 HTTP 服务。
- 正确读取请求 method、header、query、body，并写入 status、header、body。
- 配置 `http.Server` 的基础超时，避免裸用默认服务配置。
- 写出带 timeout 和 context 的 HTTP client 请求。
- 正确关闭响应体，复用 `http.Client`。
- 编写 JSON API，处理解码错误、业务错误和状态码。
- 使用中间件实现日志、panic 恢复、请求 ID 等横切逻辑。
- 理解 TCP 字节流和 UDP 数据报的区别。
- 用 `httptest` 测试 handler 和 HTTP client。

---

## 学习建议

网络代码要一直盯住边界：

1. 请求会不会超时。
2. body 有没有关闭。
3. handler 有没有正确返回状态码。
4. JSON 解码失败时响应是否清楚。
5. TCP 读写有没有协议边界。
6. 测试是否可以不依赖真实外部网络。

---

## 下一步

完成本模块后，继续学习：

- **09**: 数据库与持久化
