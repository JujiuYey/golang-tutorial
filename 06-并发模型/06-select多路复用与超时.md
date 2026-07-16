# 06: select、多路复用与超时

到目前为止,一个 goroutine 一次只能等**一个** channel。真实场景经常要"同时等好几件事,谁先来处理谁":等结果、等超时、等取消信号。这篇学 `select`——**同时盯着多个 channel 操作,哪个先就绪走哪个**。学完你能写出 Go 里最常见的两个句式:带超时的等待,和"干活 + 随时可叫停"的 for-select 循环。

对比 JS 找感觉:`select` 最像 `Promise.race`——几路一起等,先到的赢。但有两个升级:select 输了的那几路**不会被丢弃**(下轮循环接着等),而且发送也能参赛(JS 里没有"等着把值塞给别人"这回事)。

---

## 1. 基本 select:谁先就绪走谁

长得像 switch,但每个 case 是一个 **channel 操作**:

```go
func main() {
	ch1 := make(chan string)
	ch2 := make(chan string)

	go func() { time.Sleep(50 * time.Millisecond); ch1 <- "ch1 的货" }()
	go func() { time.Sleep(100 * time.Millisecond); ch2 <- "ch2 的货" }()

	select {
	case v := <-ch1:
		fmt.Println("先到:", v)
	case v := <-ch2:
		fmt.Println("先到:", v)
	}
}
```

```text
先到: ch1 的货
```

ch1 睡 50ms、ch2 睡 100ms,select 等到 ch1 就绪就走了,**只执行一个 case**,然后整个 select 语句结束(没有 switch 那种从上到下的判断顺序,更没有贯穿)。

```text
select {
case v := <-ch1:   ← 每个 case 是一个 channel 收/发操作
case v := <-ch2:   ← 不是布尔条件!
}
// 全都没就绪 → 阻塞等;有一个就绪 → 执行它;多个就绪 → 随机挑一个
```

---

## 2. 多个同时就绪:随机挑,不看先后

case 的书写顺序**不代表优先级**。两个都就绪时,Go 故意伪随机选:

```go
func main() {
	for i := 0; i < 5; i++ {
		ch1 := make(chan int, 1)
		ch2 := make(chan int, 1)
		ch1 <- 1
		ch2 <- 2

		select { // 两个 case 都准备好了
		case v := <-ch1:
			fmt.Print(v, " ")
		case v := <-ch2:
			fmt.Print(v, " ")
		}
	}
	fmt.Println()
}
```

每次运行结果都可能不同,下面是某一次的实际运行结果:

```text
2 1 1 2 1
```

为什么要随机?防止你无意中依赖 case 顺序,造成某条 channel 永远抢不到机会(饥饿)。**别写依赖 case 顺序的逻辑。**

---

## 3. default:不等,看一眼就走

没有 default 时,select 阻塞到某个 case 就绪;加了 default,所有 case 都没就绪就**立刻**走 default:

```go
func main() {
	ch := make(chan int)

	select {
	case v := <-ch:
		fmt.Println(v)
	default:
		fmt.Println("没货,不等了")
	}
}
```

```text
没货,不等了
```

这就是"非阻塞收发":试一下,不行拉倒。

🕳️ 坑:把 `for { select { ... default: } }` 当轮询用。default 让每轮循环都瞬间完成,没货时这个循环就变成全速空转,一颗 CPU 核直接拉满。JS 里 `while(!ready){}` 会卡死页面所以你不会写;Go 里它不卡别人,但烧 CPU 一样是事故。要"等",就让 select 阻塞着等,别拿 default 硬轮。

---

## 4. 超时:select + time.After

Go 名场面句式——"等结果,但最多等 500ms":

```go
func main() {
	ch := make(chan int)

	go func() {
		time.Sleep(2 * time.Second) // 慢任务
		ch <- 42
	}()

	select {
	case v := <-ch:
		fmt.Println("value:", v)
	case <-time.After(500 * time.Millisecond):
		fmt.Println("timeout: 等不下去了")
	}
}
```

```text
timeout: 等不下去了
```

`time.After(d)` 返回一个 channel,d 时间后会送来一个值——**定时器也伪装成 channel**,于是"超时"和"等结果"变成了 select 里平等的两路赛跑。JS 里对应的正是那个经典技巧:`Promise.race([task, timeoutPromise])`,一模一样的思路。

细节先记一句:高频循环里反复 `time.After` 会攒出很多定时器,那种场景用 `time.NewTimer` 复用;偶尔一次的超时用 `time.After` 完全没问题。

---

## 5. 等取消:done 信号入列

上一篇的 done channel 广播,配上 select 就成了"干活的同时竖着耳朵":

```go
select {
case v := <-work:
	fmt.Println(v)
case <-done:
	return // 收到退出广播
}
```

`done` 换成下一篇的 `ctx.Done()` 就是生产级写法——形式完全一样。

---

## 6. 发送也能当 case

select 的 case 不限于接收,发送也行。经典场景:往外交货,但下游可能已经不收了,不能吊死在发送上:

```go
func main() {
	out := make(chan int) // 始终没人接收
	done := make(chan struct{})

	go func() {
		time.Sleep(100 * time.Millisecond)
		close(done) // 100ms 后广播放弃信号
	}()

	select {
	case out <- 1:
		fmt.Println("sent")
	case <-done:
		fmt.Println("没人收货,放弃发送")
	}
}
```

```text
没人收货,放弃发送
```

没有 select 兜底,`out <- 1` 会永久卡住(03 篇的 `[chan send]`)——这正是 goroutine 泄漏的头号来源,第 10 篇会算总账。**在"可能没人收"的地方发送,发送本身就该放进 select。**

---

## 7. 🕳️ 坑:select 里没处理关闭的 channel

上一篇学过:**关闭的 channel 接收永远立即就绪**。放进 select 循环里,这个 case 会每轮都中签,零值刷屏:

```go
func main() {
	ch := make(chan int)
	close(ch)

	for i := 0; i < 3; i++ {
		select {
		case v := <-ch:
			fmt.Println("收到", v)
		}
	}
}
```

```text
收到 0
收到 0
收到 0
```

这里循环 3 次就停了;真实代码里的 `for { select { ... } }` 会无限刷。解法是老朋友 `v, ok`:

```go
case v, ok := <-ch:
	if !ok {
		return // 或看下一节,把 ch 禁用掉
	}
```

---

## 8. 冷招:nil channel 禁用 case

03 篇说过 nil channel 上的操作**永远阻塞**——在 select 里,"永远阻塞"意味着"这个 case 永远不中签"。反过来用:把收完的 channel 置为 nil,等于**动态关掉那一路**,合并多路输入时特别顺手:

```go
func main() {
	ch1 := gen(1, 2) // gen 返回一个发完就 close 的 channel
	ch2 := gen(10)

	for ch1 != nil || ch2 != nil {
		select {
		case v, ok := <-ch1:
			if !ok {
				ch1 = nil // 禁用这个 case
				continue
			}
			fmt.Println("ch1:", v)
		case v, ok := <-ch2:
			if !ok {
				ch2 = nil
				continue
			}
			fmt.Println("ch2:", v)
		}
	}
	fmt.Println("两路都收完了")
}
```

两路交错的顺序每次可能不同,下面是某一次的实际运行结果:

```text
ch2: 10
ch1: 1
ch1: 2
两路都收完了
```

不置 nil 的话,收完的那路会掉进第 7 节的零值刷屏;置了 nil,它安静退赛,循环条件 `ch1 != nil || ch2 != nil` 顺便充当"全收完"的判断。

---

## 9. for-select:常驻 goroutine 的标准心跳

长期干活的 goroutine,九成长这样:

```go
func main() {
	jobs := make(chan int)
	done := make(chan struct{})

	go func() {
		for { // 常驻循环
			select {
			case j := <-jobs:
				fmt.Println("处理 job", j)
			case <-done:
				fmt.Println("worker 退出")
				return // 唯一出口
			}
		}
	}()

	jobs <- 1
	jobs <- 2
	close(done)
	time.Sleep(50 * time.Millisecond)
}
```

```text
处理 job 1
处理 job 2
worker 退出
```

一轮循环 = 等一件事发生(来活或叫停),处理完回来继续等。这个"心跳"结构配上 context 就是第 07 篇的 worker 标准形态。**每个 for-select 必须有一条能 return 的 case**——没有出口的常驻循环就是泄漏预定。

---

## 10. select 设计检查清单

写 select 时过一遍:

1. 没 case 就绪时,该**等**(不写 default)还是该**走**(写 default)?
2. 有 channel 会被关闭吗?关闭后这一路是 return、还是置 nil 退赛?
3. 要不要加一条取消/超时 case?
4. 发送 case 会不会因为没人收而永久卡住?(卡住的发送要不要也 select 化?)
5. default 有没有把循环变成 CPU 空转?

---

## 本篇重点

- [ ] select 同时等多个 channel 收/发操作,谁先就绪走谁,只走一个;像 `Promise.race`,但输家下轮还能再赛、发送也能参赛。
- [ ] 多 case 同时就绪是随机挑——别依赖 case 书写顺序;default = 不等直接走,循环里滥用会 CPU 空转。
- [ ] 超时句式:`case <-time.After(d)` 和结果赛跑;取消句式:`case <-done`(下一篇升级为 `ctx.Done()`)。
- [ ] 关闭的 channel 在 select 里永远中签刷零值:用 `v, ok` 检查,要么 return 要么置 nil 禁用那一路。
- [ ] for-select 是常驻 goroutine 的标准心跳,必须留一条 return 出口;可能没人收的发送也要包进 select。

---

## 练习

1. 同时等待两个 channel,打印先到的值。
2. 用 `time.After` 给接收操作加 500ms 超时。
3. 写一个 `for select`,同时处理 jobs 和 done。
4. 合并两个输入 channel,关闭后把对应 channel 设为 nil。
5. 写一个发送 case,同时监听退出信号(done channel 或 ctx)。

提示:练习 2 让发送方故意睡 1 秒制造超时;练习 4 照第 8 节骨架,重点体会循环条件为什么写 `!= nil`;练习 5 先构造一个"永远没人收"的 channel 逼出第二条 case。
