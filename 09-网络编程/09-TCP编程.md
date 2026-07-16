# 09: TCP 编程：Listener、Conn 与协议边界

TCP 是可靠的字节流协议。Go 用 `net.Listener` 接受连接，用 `net.Conn` 读写连接。

---

## 1. TCP 服务器

```go
listener, err := net.Listen("tcp", ":8080")
if err != nil {
    return err
}
defer listener.Close()

for {
    conn, err := listener.Accept()
    if err != nil {
        return err
    }
    go handleConn(conn)
}
```

每个连接通常交给一个 goroutine 处理。

---

## 2. 处理连接

```go
func handleConn(conn net.Conn) {
    defer conn.Close()

    scanner := bufio.NewScanner(conn)
    for scanner.Scan() {
        line := scanner.Text()
        fmt.Fprintln(conn, "echo:", line)
    }
    if err := scanner.Err(); err != nil {
        slog.Error("connection error", "err", err)
    }
}
```

这个例子用换行作为消息边界。

---

## 3. TCP 是字节流

TCP 没有消息边界。一次 `Write` 不一定对应一次 `Read`。

如果要发送一条条消息，需要自己定义协议边界：

- 用换行分隔。
- 用固定长度。
- 用长度前缀。
- 用 JSON 流。

不要假设 `Read(buffer)` 一次就读到完整消息。

---

## 4. TCP 客户端

```go
conn, err := net.DialTimeout("tcp", "localhost:8080", 3*time.Second)
if err != nil {
    return err
}
defer conn.Close()

if _, err := fmt.Fprintln(conn, "hello"); err != nil {
    return err
}
```

使用 `DialTimeout` 或 `net.Dialer` 设置超时。

---

## 5. 设置读写 deadline

```go
if err := conn.SetDeadline(time.Now().Add(10 * time.Second)); err != nil {
    return err
}
```

也可以分别设置：

```go
conn.SetReadDeadline(time.Now().Add(5 * time.Second))
conn.SetWriteDeadline(time.Now().Add(5 * time.Second))
```

deadline 可以避免连接长期卡住。

---

## 6. 半关闭

TCP 支持半关闭，但 `net.Conn` 接口本身没有直接暴露。具体类型 `*net.TCPConn` 有：

```go
tcpConn.CloseWrite()
tcpConn.CloseRead()
```

一般业务里先掌握完整关闭 `Close()` 即可。

---

## 7. 并发连接控制

服务器不能无限制创建 goroutine。可以用信号量限制并发连接数：

```go
sem := make(chan struct{}, 100)

for {
    conn, err := listener.Accept()
    if err != nil {
        return err
    }

    sem <- struct{}{}
    go func() {
        defer func() { <-sem }()
        handleConn(conn)
    }()
}
```

---

## 8. 优雅退出

真实服务器需要在退出时关闭 listener，让 `Accept` 返回错误，再等待正在处理的连接结束。

可以组合：

- context
- WaitGroup
- listener.Close()
- connection deadline

本章先掌握连接生命周期。

---

## 9. 错误处理

读写网络连接时，错误很常见：

- 客户端断开。
- 超时。
- 协议格式错误。
- 连接被重置。

不要把每个断开都当成严重错误。根据场景区分正常断开和异常。

---

## 10. TCP 设计检查

1. 协议边界是什么。
2. 连接是否关闭。
3. 是否设置超时或 deadline。
4. goroutine 数量是否受控。
5. 错误日志是否带远端地址。

---

## 练习

1. 写一个按行 echo 的 TCP server。
2. 写一个 TCP client，发送一行并读取响应。
3. 给连接设置 5 秒 deadline。
4. 解释为什么一次 Write 不等于一次 Read。
5. 用信号量限制最多 10 个并发连接。
