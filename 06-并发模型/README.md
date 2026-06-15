# 06: 并发模型

本模块讲 Go 的并发模型。学完这一章，你应该能判断 goroutine 什么时候结束、channel 什么时候阻塞、谁负责关闭、如何取消任务、共享数据如何保护，以及怎样用工具发现数据竞争。

---

## 文章列表

1. [goroutine 基础与调度模型](./01-goroutine基础与调度模型.md)
2. [WaitGroup：等待一组任务完成](./02-WaitGroup等待任务完成.md)
3. [channel 基础：发送、接收与阻塞](./03-channel基础发送接收与阻塞.md)
4. [缓冲 channel 与单向 channel](./04-缓冲channel与单向channel.md)
5. [关闭 channel、range 与退出信号](./05-关闭channel-range与退出信号.md)
6. [select、多路复用与超时](./06-select多路复用与超时.md)
7. [context：取消、超时与请求边界](./07-context取消超时与请求边界.md)
8. [Mutex 与 RWMutex：保护共享状态](./08-Mutex与RWMutex保护共享状态.md)
9. [Once、sync.Map 与 atomic](./09-Once-syncMap与atomic.md)
10. [数据竞争、死锁与 goroutine 泄漏](./10-数据竞争死锁与goroutine泄漏.md)
11. [常见并发模式：worker pool、fan-out/fan-in、pipeline](./11-常见并发模式.md)
12. [并发模型综合练习](./12-并发模型综合练习.md)

---

## 学习目标

完成本模块后，你应该能够：

- 正确启动 goroutine，并用同步手段等待它结束。
- 理解 channel 的发送、接收、阻塞、关闭和遍历规则。
- 区分无缓冲 channel 和缓冲 channel 的同步语义。
- 使用 `select` 同时等待多个 channel、超时或取消信号。
- 使用 `context.Context` 传递取消、截止时间和请求范围值。
- 使用 `sync.WaitGroup`、`sync.Mutex`、`sync.RWMutex`、`sync.Once`。
- 知道普通 map 不能并发读写，知道什么时候考虑 `sync.Map`。
- 使用 `go test -race` 发现数据竞争。
- 识别死锁、goroutine 泄漏和错误的 channel 关闭方式。
- 写出简单的 worker pool、fan-out/fan-in 和 pipeline。

---

## 学习建议

并发代码要一边写一边问：

1. 每个 goroutine 的退出条件是什么。
2. 谁发送，谁接收，谁关闭 channel。
3. 共享数据有没有锁或其他同步保护。
4. 如果下游提前退出，上游会不会卡住。
5. 任务失败、超时或取消时，所有 goroutine 能不能退出。

---

## 下一步

完成本模块后，继续学习：

- **07**: 标准库
