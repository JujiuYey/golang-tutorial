# 05: 关闭 channel、range 与退出信号

前两篇一直有个悬念:接收方怎么知道"货发完了,别等了"?答案是 `close`。这篇讲清 close 的确切含义、`for range` 收货的标准姿势、"谁负责关门"的规矩,以及一个漂亮用法——**用关闭动作本身当广播信号**。学完你能躲开 close 家的三个 panic。

先纠正一个望文生义:`close(ch)` 不是"销毁管道",更不是"叫停 goroutine"。它只是在管道上挂一块牌子:**"本店不再进货"**——存货照卖,只是不会再有新的了。

---

## 1. close 之后会发生什么

```go
close(ch)
```

挂牌之后的三条规则:

- 不能再发送(违者 panic,第 5 节演示)。
- 缓冲区里囤的货**照常可以接收**——close 不清空存货。
- 存货取完后,继续接收**不阻塞**,立刻返回元素类型的零值。

第三条和 map 的手感很像:从 map 取不存在的 key 得到零值,分不清"存的就是零值"还是"没有"——所以 channel 也配了同款 `v, ok` 双返回值。

---

## 2. `v, ok` 判断关闭状态

```go
func main() {
	ch := make(chan int, 2)
	ch <- 1
	close(ch)

	v, ok := <-ch
	fmt.Println(v, ok) // 缓冲区里还有货

	v, ok = <-ch
	fmt.Println(v, ok) // 货取完了

	v, ok = <-ch
	fmt.Println(v, ok) // 再取还是一样
}
```

```text
1 true
0 false
0 false
```

`ok == false` 的确切含义:**已关闭,且存货清空**。注意第一次接收 `ok` 还是 `true`——close 了不代表立刻收不到货。`0 false` 可以无限取,关闭后的 channel 永远"立即有响应",这个特性第 8 节要变成招数。

---

## 3. for range:自动收货到关门

手写 `v, ok` 循环太啰嗦,`range` 帮你包了:

```go
func producer(out chan<- int) {
	defer close(out) // 发送方负责关门
	for i := 1; i <= 3; i++ {
		out <- i
	}
}

func main() {
	ch := make(chan int)
	go producer(ch)

	for v := range ch { // 一直收,直到 ch 关闭且取空
		fmt.Println(v)
	}
	fmt.Println("range 结束")
}
```

```text
1
2
3
range 结束
```

(单生产者按序发,这个输出顺序是稳定的。)

`range ch` = "有货收货,没货等着,关门且清空就散会"。和 02 模块 range slice/map 不同:这里只有一个循环变量(值本身,没有索引),而且**循环会阻塞等待**。

🕳️ 坑:channel 永远不关,`range` 就永远不散会。所有发送方都退出了却没人 close,接收方卡死——这是 goroutine 泄漏的经典成因(第 10 篇细讲)。**用 range 收货,就必须有人负责关门。**

---

## 4. 规矩:发送方关门,接收方绝不关

谁 close?**发送方**——只有它知道"我不会再发了"。上面 `producer` 里的 `defer close(out)` 就是标准姿势(发完必关,哪条路退出都关,06-03 篇 defer 的老本行)。

接收方为什么绝对不能关?它无法知道是否还有发送方正在路上——它一关门,正在发货的人就撞墙 panic(下一节)。回忆上一篇:`<-chan T` 类型上根本不让调 close,编译器已经把这条规矩刻进了类型系统。

**一句话总结:close 是发送方的收尾动作,不是接收方的拒收手段。**

---

## 5. 🕳️ 坑一:关门后发送 = panic

```go
func main() {
	ch := make(chan int)
	close(ch)
	ch <- 1 // 向已关闭的 channel 发送
}
```

```text
panic: send on closed channel

goroutine 1 [running]:
main.main()
	./e0503.go:6 +0x40
exit status 2
```

注意这是 panic 不是 error——没有 `v, ok` 式的温柔试探,撞上就崩。所以多发送方场景的关闭时机要格外小心(第 7 节给方案)。

---

## 6. 🕳️ 坑二、坑三:close 的另外两种 panic

close 一个 nil channel(声明了忘 make,上上篇的老朋友):

```go
func main() {
	var ch chan int // nil channel
	close(ch)
}
```

```text
panic: close of nil channel

goroutine 1 [running]:
main.main()
	./e0504.go:5 +0x20
exit status 2
```

close 已经关过的 channel:

```text
panic: close of closed channel
```

三个 panic 凑齐了:**关 nil、关两次、关了还发**。共同解法只有一条纪律——**每个 channel 有且只有一个明确的关闭者**,它 close 恰好一次,且在所有发送结束之后。

---

## 7. 多发送方怎么关:请个协调者

多个 goroutine 往同一个 channel 发货时,任何一个发送方都不敢关门(别人可能还在发)。标准解法在 02 篇见过,现在能完全看懂了:**WaitGroup 确认全员收工,协调者统一关门**:

```go
func main() {
	out := make(chan int)
	var wg sync.WaitGroup

	for i := 0; i < 3; i++ {
		wg.Add(1)
		go func(i int) {
			defer wg.Done()
			out <- i // 发送方只发,不关
		}(i)
	}

	go func() {
		wg.Wait()  // 确认所有发送方收工
		close(out) // 由协调者统一关门
	}()

	for v := range out {
		fmt.Println(v)
	}
}
```

输出顺序每次可能不同,下面是某一次的实际运行结果:

```text
0
1
2
```

三个角色分工:发送方只发、协调者只关、接收方 range 收到散会。谁也不越界,三个 panic 一个都碰不上。

---

## 8. done channel:把"关门"用作广播

第 2 节说过:关闭后的 channel 上,接收**永远立即返回**。反过来利用它——一个从来不发货的 channel,专门靠"关门"这个动作通知所有等待者:

```go
func main() {
	done := make(chan struct{})
	var wg sync.WaitGroup

	for i := 1; i <= 3; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			<-done // 都堵在这等信号
			fmt.Println("worker", id, "收到退出信号")
		}(i)
	}

	close(done) // 关门 = 对所有人广播
	wg.Wait()
}
```

输出顺序每次可能不同,下面是某一次的实际运行结果:

```text
worker 1 收到退出信号
worker 2 收到退出信号
worker 3 收到退出信号
```

妙处在"一对多":普通发送一次只能叫醒一个接收方(03 篇第 9 节:每件货只有一个买主),想通知 3 个人得发 3 次;而 **close 一次,所有堵在 `<-done` 上的人同时放行**——这是 channel 世界唯一的广播机制。

对比 JS:这就是 `AbortController` 的味道——`close(done)` 相当于 `controller.abort()`,`<-done` 相当于监听 `signal`。Go 标准库确实把这个模式包装成了正式设施:`context.Context`,第 07 篇的主角。

---

## 9. close 只能说"没了",不能说"为什么"

关闭携带的信息量只有一个 bit:没有更多值了。它说不清是正常收工还是出错中断。要传错误,老老实实用数据说话:

```go
type Result struct {
	Value int
	Err   error
}
```

发 `Result` 而不是裸值,接收方对每件货检查 `Err`——06 模块"错误是值"的思路搬到并发世界。或者配一条独立的 error channel(02 篇第 5 节),或者用 context 的取消原因(下一篇)。

---

## 10. channel 关闭检查清单

写完 channel 代码,过一遍:

1. 谁是**唯一**关闭者?(发送方本人,或协调者)
2. 关闭是否发生在**所有发送之后**?(多发送方 → WaitGroup + 协调者)
3. 接收方能在关闭后退出吗?(range 会自动散会;手写循环记得查 `ok`)
4. 有没有人可能向已关闭的 channel 发送?(panic 三兄弟之首)
5. 需要传递错误或取消原因吗?(close 说不了"为什么",用 Result / error channel / context)

---

## 本篇重点

- [ ] close = 挂牌"不再进货":存货照收,取空后接收立即返回零值 + `ok=false`;它不清空、不停 goroutine。
- [ ] `for v := range ch` 自动收货到"关闭且取空";没人关门 range 就永远等——用 range 必须配一个关门人。
- [ ] 规矩:发送方(或协调者)关门,接收方绝不关;多发送方用 WaitGroup + 协调者统一 close。
- [ ] close 三 panic:向已关闭 channel 发送、close nil、close 两次——纪律是"每个 channel 恰好一个关闭者,关恰好一次"。
- [ ] close 是一对多广播(done channel ≈ AbortController),但只能说"没了"不能说"为什么";传错误用 Result 类型或 error channel。

---

## 练习

1. 写一个 producer,发送 1 到 5 后关闭 channel。
2. 用 `for range` 接收 producer 输出。
3. 写代码验证关闭后继续接收会得到零值和 `ok=false`。
4. 写多发送方示例,用 `WaitGroup` 统一关闭结果 channel。
5. 用关闭 `done` channel 的方式通知多个 goroutine 退出。

提示:练习 3 用带缓冲的 channel 先囤一件货再 close,观察 `ok` 从 true 变 false 的分界点;练习 5 注意让 main 等到所有 worker 打印完再退出(01 篇的教训)。
