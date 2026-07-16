# 03: channel 基础:发送、接收与阻塞

上一篇说 Go 没有 Promise,goroutine 的结果要"自己搭路送回来"——这篇就是修路:channel(通道)。学完你能让两个 goroutine 传值、汇合,还能看懂 Go 并发第一名场面报错:`fatal error: all goroutines are asleep - deadlock!`。

先给 JS 用户校准坐标:JS 里 async 任务用 Promise 交结果、用 `await` 等;Go 里这两件事合并成一根"管道"——**一头把值塞进去(发送),另一头把值取出来(接收),没对上就原地等**。传值和等待是同一个动作,这是 channel 和 Promise 最大的不同:Promise 是"一次性的结果盒子",channel 是"可以持续过货的传送带"。

---

## 1. 创建 channel

```go
ch := make(chan int)
```

```text
ch := make(chan int)
//         ─┬── ─┬─
//          │    └─ 管道里跑什么类型的货(只能是 int)
//          └─ channel 类型,用 make 创建(和 map、slice 一样)
```

这样创建的是**无缓冲** channel——管道粗细为零,货必须手递手交接(缓冲版下一篇讲)。

channel 的零值是 nil(map 那篇的老朋友:声明不 make,得到 nil):

```go
func main() {
	var ch chan int
	fmt.Println(ch == nil)
}
```

```text
true
```

🕳️ 坑:nil channel 上的发送和接收会**永久阻塞**,不报错、不 panic,就是无声地卡住。忘了 `make` 的 channel 比忘了 make 的 map 更阴险(map 起码写入会 panic)。

---

## 2. 发送和接收:一个箭头两个方向

```go
ch <- 100   // 发送:值进管道(箭头指向 ch,货流进去)
v := <-ch   // 接收:值出管道(箭头从 ch 出来,货流出来)
```

记法:**`<-` 永远指着数据流动的方向**。跑一个最小例子:

```go
func main() {
	ch := make(chan int)

	go func() {
		ch <- 100 // 发送:把 100 塞进管道
	}()

	v := <-ch // 接收:从管道取出来
	fmt.Println(v)
}
```

```text
100
```

注意发送方在另一个 goroutine 里。为什么必须这样?看第 4 节。

---

## 3. 无缓冲 channel 是同步点

无缓冲 channel 的交接规则:**发送方和接收方必须同时到场,一手交货一手收货**。谁先到,谁就站在原地等对方——所以它天然是两个 goroutine 的"汇合点":

```go
func main() {
	ch := make(chan string)

	go func() {
		fmt.Println("goroutine: 开始干活")
		time.Sleep(50 * time.Millisecond)
		ch <- "done"
	}()

	fmt.Println("main: 等待中...")
	msg := <-ch // 卡在这里,直到对面送来值
	fmt.Println("main: 收到", msg)
}
```

前两行的先后每次可能不同,下面是某一次的实际运行结果:

```text
main: 等待中...
goroutine: 开始干活
main: 收到 done
```

main 在 `<-ch` 上睡了 50 毫秒,直到对面交货才醒——**这就取代了上一篇的 `time.Sleep` 猜时间**:不用猜,货到了自然醒。"main: 收到 done" 保证最后出现,这是 channel 给的顺序承诺。

对比 JS:`msg := <-ch` 的体感非常像 `const msg = await promise`。但 `await` 是把控制权还给事件循环,Go 是这个 goroutine 真的停住,其他 goroutine 照跑——不需要 async 标记,任何函数里都能等。

---

## 4. 发送会阻塞:死锁初体验

发送方到了场,接收方永远不来,会怎样?

```go
func main() {
	ch := make(chan int)
	ch <- 1 // 没有任何接收方
}
```

```text
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan send]:
main.main()
	./e0303.go:5 +0x34
exit status 2
```

拆一下这份报错,以后你会经常见到它:

- `all goroutines are asleep - deadlock!`:运行时发现**所有** goroutine 都在等,没有任何人能推进程序,确定性卡死,干脆自杀报错。
- `goroutine 1 [chan send]`:1 号(main)卡在"channel 发送"上——方括号里的状态是破案关键。

为什么第 2 节同样是 `ch <- 100` 却没事?因为它在**另一个** goroutine 里,main 随后就来接货了。这段代码里发送写在 main 自己身上,main 卡住 = 全员卡住。

---

## 5. 接收也会阻塞

反过来一样,只有接收没有发送:

```go
func main() {
	ch := make(chan int)
	v := <-ch // 没有任何发送方
	fmt.Println(v)
}
```

```text
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan receive]:
main.main()
	./e0309.go:7 +0x34
exit status 2
```

状态从 `[chan send]` 变成了 `[chan receive]`,其余一样。

要纠正一个直觉:**阻塞不是 bug,阻塞是 channel 的工作方式**——第 3 节的"等待汇合"靠的就是阻塞。真正的问题只有一个:卡住之后,还有没有别的 goroutine 能来解开这个结。有,叫同步;没有,叫死锁。

---

## 6. channel 传递的是值(复制)

```go
func main() {
	ch := make(chan int)

	go func() {
		x := 10
		ch <- x // 发送的是 x 此刻的值(复制)
		x = 20  // 改晚了,channel 里已经是复制品
	}()

	fmt.Println(<-ch)
}
```

```text
10
```

发送那一刻,值被**复制**进 channel,之后改原变量不影响已发出的货——和函数传参、`go` 启动参数、defer 参数是同一条"当场求值、复制定格"规则,Go 里已经是第三次见了。

但复制的深浅遵循 04 模块的规则:slice、map、指针被发送时,复制的是描述符/地址,**底层数据仍然共享**。通过 channel 发一个 slice 给别的 goroutine,两边同时改元素照样是数据竞争。安全习惯:货交出去之后,发送方就别再碰它。

---

## 7. 用 channel 返回结果:Promise 平替成型

上一篇的遗留问题"goroutine 没有返回值"正式解决:

```go
func square(n int, out chan<- int) {
	out <- n * n
}

func main() {
	out := make(chan int)
	go square(4, out)

	result := <-out // 既拿结果,也等计算完成
	fmt.Println(result)
}
```

```text
16
```

一行 `<-out` 同时干了 `await` 的两件事:等任务完成 + 拿到结果。参数里的 `chan<- int` 是"只能发送"的单向声明,下一篇细讲,先眼熟。

对照表:

| JS | Go |
|----|----|
| `const p = task()` | `out := make(chan int)` + `go task(out)` |
| `await p` | `<-out` |
| Promise 只能 resolve 一次 | channel 可以持续发多个值 |
| 不 await,结果也在 p 里 | 没人接收,发送方会一直卡住 |

---

## 8. 多个发送方

多个 goroutine 可以往同一个 channel 发货,接收方来者不拒:

```go
func main() {
	ch := make(chan int)

	for i := 0; i < 3; i++ {
		go func(i int) {
			ch <- i
		}(i)
	}

	for i := 0; i < 3; i++ {
		fmt.Println(<-ch)
	}
}
```

接收顺序取决于谁先抢到交货机会,和启动顺序无关。输出顺序每次可能不同,下面是某一次的实际运行结果:

```text
2
0
1
```

注意 main 精确接收 3 次——发几个收几个,账要对上。少收一次,就有一个发送方永远卡在交货上(这叫 goroutine 泄漏,第 10 篇细讲)。

---

## 9. 多个接收方:每件货只有一个买主

反过来,多个 goroutine 从同一个 channel 收货时,**每个值只会被其中一个接收方拿到**(不是广播):

```go
func main() {
	jobs := make(chan int)
	var wg sync.WaitGroup

	for w := 1; w <= 3; w++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			for job := range jobs {
				fmt.Println("worker", id, "拿到 job", job)
			}
		}(w)
	}

	for j := 1; j <= 5; j++ {
		jobs <- j
	}
	close(jobs)
	wg.Wait()
}
```

哪个 worker 抢到哪个 job 每次都可能不同,下面是某一次的实际运行结果:

```text
worker 3 拿到 job 1
worker 3 拿到 job 4
worker 3 拿到 job 5
worker 2 拿到 job 3
worker 1 拿到 job 2
```

5 个 job 每个恰好被处理一次,分配完全看调度。这就是"任务分发"的雏形——worker pool 模式(第 11 篇)的地基。`range jobs` 和 `close` 的细节在第 05 篇展开,这里先看个整体。

---

## 10. channel 不是万能锤

channel 很适合表达"goroutine 之间传数据、对时机"。但如果代码本来就是同步的——普通函数调用、遍历 slice、查 map——别为了"并发感"硬套 channel。**先写清晰的同步代码,确实需要并发时再引入 channel**,这是 Go 社区的共识,不是保守。

---

## 本篇重点

- [ ] channel 是 goroutine 间的管道:`ch <- v` 发送、`<-ch` 接收,箭头永远指向数据流向;零值 nil,操作 nil channel 永久卡住。
- [ ] 无缓冲 channel 手递手交接:先到的一方阻塞等对方——阻塞是同步机制,不是 bug。
- [ ] `<-ch` ≈ `await`:一次搞定"等完成 + 拿结果";它是 Go 里 Promise 的平替,但能持续传多个值。
- [ ] 卡住且没有任何 goroutine 能解围 = `fatal error: all goroutines are asleep - deadlock!`,方括号里的 `[chan send]` / `[chan receive]` 指明卡点。
- [ ] 发送即复制(当场求值),但 slice/map/指针的底层数据仍共享;多接收方之间是分货不是广播。

---

## 练习

1. 用无缓冲 channel 从 goroutine 返回一个字符串。
2. 写代码验证无接收方发送会阻塞,看清报错里的 `[chan send]`。
3. 启动 3 个 goroutine 向同一个 channel 发送结果,主 goroutine 接收 3 次;多跑几次观察顺序。
4. 启动 2 个 worker 从同一个 jobs channel 读取任务,确认每个任务只被处理一次。
5. 解释 channel 传递 slice 时,哪些部分被复制,哪些数据可能共享。

提示:练习 4 可以照第 9 节的骨架改;练习 5 回顾 04 模块 slice 头部(指针+长度+容量)的结构,再套第 6 节的复制规则。
