# 10: UDP 编程：数据报与丢包边界

UDP 是无连接的数据报协议。它保留消息边界，但不保证送达、顺序和去重。

---

## 1. 监听 UDP

```go
addr, err := net.ResolveUDPAddr("udp", ":8080")
if err != nil {
    return err
}

conn, err := net.ListenUDP("udp", addr)
if err != nil {
    return err
}
defer conn.Close()
```

`ListenUDP` 返回 `*net.UDPConn`。

---

## 2. 接收数据报

```go
buf := make([]byte, 2048)

for {
    n, remote, err := conn.ReadFromUDP(buf)
    if err != nil {
        return err
    }
    fmt.Printf("from %s: %s\n", remote, string(buf[:n]))
}
```

一次 `ReadFromUDP` 读取一个数据报。如果 buffer 太小，超出的数据会被截断。

---

## 3. 发送响应

```go
_, err := conn.WriteToUDP([]byte("ok"), remote)
if err != nil {
    return err
}
```

UDP 服务端通常通过 `remote` 地址回复发送方。

---

## 4. UDP 客户端

```go
remote, err := net.ResolveUDPAddr("udp", "localhost:8080")
if err != nil {
    return err
}

conn, err := net.DialUDP("udp", nil, remote)
if err != nil {
    return err
}
defer conn.Close()

if _, err := conn.Write([]byte("ping")); err != nil {
    return err
}
```

`DialUDP` 不建立 TCP 那种连接，它只是绑定默认远端地址，方便后续 `Read`、`Write`。

---

## 5. 设置 deadline

```go
if err := conn.SetReadDeadline(time.Now().Add(2 * time.Second)); err != nil {
    return err
}
```

UDP 读操作也可能一直等不到数据，所以要考虑 deadline。

---

## 6. UDP 的消息边界

UDP 保留数据报边界。一次 `Write` 对应一个数据报，一次 `ReadFromUDP` 读一个数据报。

这和 TCP 字节流不同。

---

## 7. 丢包和乱序

UDP 不保证：

- 一定送达。
- 按顺序送达。
- 只送达一次。

如果业务需要可靠性，要在应用层自己设计确认、重传、序号、超时等机制，或者直接使用 TCP。

---

## 8. 数据报大小

UDP 数据报不适合太大。过大的数据可能被分片，丢包风险更高。

课程阶段可以记住：UDP 适合小消息、低延迟、允许丢失或应用层能处理丢失的场景。

---

## 9. 错误处理

UDP 没有连接状态，很多错误只能在读写时发现。

服务端循环中，如果某个数据报处理失败，通常记录错误后继续处理下一个数据报，而不是直接退出整个服务。

---

## 10. UDP 设计检查

1. 数据报最大大小是否可控。
2. 丢包是否可接受。
3. 是否需要序号和重传。
4. 是否设置读写 deadline。
5. buffer 是否足够。

---

## 练习

1. 写 UDP echo server。
2. 写 UDP client 发送 `ping` 并读取响应。
3. 给 UDP client 设置 2 秒读 deadline。
4. 故意使用较小 buffer，观察数据截断。
5. 总结 TCP 和 UDP 在消息边界上的差异。
