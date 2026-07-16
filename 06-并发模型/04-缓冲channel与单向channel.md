# 04: 缓冲 channel 与单向 channel

上一篇的 channel 是"手递手交接",发送方没人接就得干等。这篇给管道加一段**存货空间**(缓冲 channel),让发送方能先放下货就走;再学**单向 channel**——在函数签名上写死"这头只进"或"这头只出",让编译器帮你把关。学完你还能掌握一个高频实战招:用缓冲 channel 限制并发数量。

---

## 1. 创建缓冲 channel:make 的第二个参数

```go
func main() {
	ch := make(chan int, 3)

	ch <- 1 // 没有接收方,但不阻塞:放进缓冲区
	ch <- 2

	fmt.Println("len:", len(ch), "cap:", cap(ch))
}
```

```text
len: 2 cap: 3
```

```text
make(chan int, 3)
//             ┬
//             └─ 缓冲容量:管道里最多囤 3 件货
```

对比上一篇:同样是"发送后没人接收",无缓冲版当场死锁,这里却顺利跑完——货放进了缓冲区。`len(ch)` 是当前囤了几件,`cap(ch)` 是最多囤几件(和 slice 的 len/cap 一个手感)。

---

## 2. 缓冲满了,照样阻塞

缓冲只是缓冲,不是豁免。第 3 件塞满之后,第 4 件(这里是第 3 次发送遇上容量 2)就回到无缓冲的处境——等人来取:

```go
func main() {
	ch := make(chan int, 2)

	ch <- 1
	ch <- 2
	fmt.Println("前两次发送顺利")

	ch <- 3 // 缓冲区满了,又没有接收方
}
```

```text
前两次发送顺利
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan send]:
main.main()
	./e0402.go:12 +0x8c
exit status 2
```

反方向也对称:**缓冲区空了、又没有发送方,接收会阻塞**。

**一句话总结:缓冲 channel 的阻塞规则 = 满了堵发送,空了堵接收。**

---

## 3. 接收顺序:先进先出

```go
func main() {
	ch := make(chan int, 2)
	ch <- 1
	ch <- 2

	fmt.Println(<-ch)
	fmt.Println(<-ch)
}
```

```text
1
2
```

缓冲区是队列不是栈:先放的先取,顺序有保证。(注意:这和"多个 goroutine 抢着发/收时谁先轮到"是两码事——单条 channel 内货物有序,抢管道的人无序。)

---

## 4. 🕳️ 坑:把缓冲当无限队列

以为会怎样:容量开大点,生产者随便发,反正有缓冲兜着。
实际怎样:只要生产长期快于消费,缓冲区**迟早**满,发送方开始堆积阻塞,goroutine 越攒越多——问题没消失,只是推迟爆发,而且爆发时更难查。

写过 Node 的对这个应该有感觉:这就是 stream 的**背压**(backpressure)问题,`highWaterMark` 就是那个"容量"。Go 里的态度一样:缓冲容量不是拍脑袋的性能参数,应该来自明确的设计依据——worker 数量、批次大小、下游系统的并发上限。

---

## 5. 容量改变的是同步语义

| | 无缓冲 `make(chan T)` | 缓冲 `make(chan T, n)` |
|---|---|---|
| 发送方何时能走 | 必须等到接收方到场 | 缓冲没满就能走 |
| 表达的关系 | **同步**:两边在此刻汇合 | **解耦**:生产消费步调可以错开 |
| 收到货意味着 | 对方"此刻"正在发 | 对方"曾经"发过,现在在哪不知道 |

所以容量不是随手可调的数字——从 0 改成 1,程序的时序关系就变了,依赖"发送完成 = 对方已收到"的逻辑会悄悄失效。

**一句话总结:无缓冲传递"时机",缓冲传递"数据"。**

---

## 6. 单向 channel:在类型上写死方向

函数签名里可以声明"我只用这一头":

```go
func producer(out chan<- int) { // 只许发送
	for i := 0; i < 3; i++ {
		out <- i
	}
	close(out)
}

func consumer(in <-chan int) { // 只许接收
	for v := range in {
		fmt.Println("收到", v)
	}
}
```

```text
chan<- int        <-chan int
// ─┬─               ─┬─
//  └ 箭头指进 chan     └ 箭头从 chan 出来
//    :只能发送          :只能接收
```

记法和上一篇一致:**箭头指着数据流向**。箭头进 chan,数据只进不出(只发);箭头出 chan,数据只出不进(只收)。

在只发送的 channel 上接收?编译器直接拦下:

```go
func producer(out chan<- int) {
	v := <-out // 试图从"只发送"channel 接收
	fmt.Println(v)
}
```

```text
./e0405.go:6:9: invalid operation: cannot receive from send-only channel chan<- int out (variable of type chan<- int)
```

对比 JS:这种"方向约束"在 JS 里没有对应物,最接近的是 TypeScript 的 `readonly`——把使用规则写进类型,违规在编译期就死,不用等运行时。

---

## 7. 双向 channel 自动"降级"成单向

调用方创建的还是普通双向 channel,传参时自动收窄:

```go
func main() {
	ch := make(chan int) // 双向 channel
	go producer(ch)      // 传入后,函数内只能发
	consumer(ch)         // 传入后,函数内只能收
}
```

```text
收到 0
收到 1
收到 2
```

(单生产者按序发送,这个输出顺序是稳定的。)

注意:这只是**视角限制**,不是新建了一条管道——producer 和 consumer 操作的是同一个 `ch`,只是各自被限定了权限。方向只能收窄不能放宽:双向转单向 OK,单向转回双向不行。

另外看一眼细节:`consumer` 的参数是 `<-chan int`,所以它**不能** close 这个 channel——close 是发送方的特权,编译器连这个都管。这为下一篇"谁负责关闭"埋个伏笔。

---

## 8. 实战招:缓冲 channel 当"停车场"限流

缓冲 channel 的容量天然是个并发上限——把它当停车场用,车位满了就排队:

```go
func main() {
	sem := make(chan struct{}, 2) // 最多 2 个任务同时跑
	var wg sync.WaitGroup

	for i := 1; i <= 5; i++ {
		sem <- struct{}{} // 占坑:满了就在这排队
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			defer func() { <-sem }() // 干完让坑
			fmt.Println("task", id, "开始")
			time.Sleep(100 * time.Millisecond)
			fmt.Println("task", id, "结束")
		}(i)
	}
	wg.Wait()
}
```

输出顺序每次可能不同,下面是某一次的实际运行结果:

```text
task 2 开始
task 1 开始
task 1 结束
task 2 结束
task 4 开始
task 3 开始
task 3 结束
task 4 结束
task 5 开始
task 5 结束
```

能看出"最多两个在跑"的节奏:先两个开始,有人结束才有新人开始。原理:`sem <- struct{}{}` 在缓冲满(2 个坑被占)时阻塞,第 2 节的"满了堵发送"从坑变成了工具。这个东西的学名叫**信号量**(semaphore)。

对比 JS:相当于 `p-limit` 这类并发限制库干的活,Go 用一个缓冲 channel 就搞定了。

---

## 9. `chan struct{}`:纯信号,不带货

上面 `sem` 里传的 `struct{}{}` 长得很怪,拆一下:

```text
struct{}{}
// ──┬─── ┬
//   │    └─ 后一对 {}:构造一个实例(空的,啥也不填)
//   └─ 前一段 struct{}:零字段结构体类型,占 0 字节
```

channel 只用来传信号、不关心值是什么时,惯例用 `chan struct{}`——零内存开销,且向读者明说"这里没有数据,只有事件":

```go
func main() {
	done := make(chan struct{})

	go func() {
		fmt.Println("干活...")
		done <- struct{}{} // 发信号:值本身没意义
	}()

	<-done // 等信号
	fmt.Println("收到完成信号")
}
```

```text
干活...
收到完成信号
```

用 close 来"广播"信号是更常见的招,正好是下一篇的主角。

---

## 本篇重点

- [ ] `make(chan T, n)` 给管道加 n 个存货位:满了堵发送,空了堵接收;`len` 看现货,`cap` 看容量。
- [ ] 缓冲不是无限队列,长期生产 > 消费迟早堵死(Node stream 背压同款问题);容量要有设计依据。
- [ ] 无缓冲传"时机"(同步汇合),缓冲传"数据"(解耦步调);改容量 = 改时序语义,不能随手调。
- [ ] `chan<- T` 只发、`<-chan T` 只收,箭头指数据流向;双向传参自动收窄,违规编译期报错,close 只归发送方。
- [ ] 缓冲 channel + 占坑/让坑 = 信号量限流;纯信号用 `chan struct{}`。

---

## 练习

1. 创建容量为 2 的 channel,发送 2 个值,再接收它们。
2. 写代码验证第三次发送会阻塞,看清死锁报错。
3. 写 `producer(out chan<- int)` 和 `consumer(in <-chan int)`,再试试在 producer 里接收,读一读编译错误。
4. 用容量为 3 的 channel 限制同时运行的任务数量,打印开始/结束验证节奏。
5. 用 `chan struct{}` 表示完成信号。

提示:练习 4 照第 8 节的骨架,把容量和任务数改一改,多跑几次观察"最多 3 个在跑"的规律是否始终成立(顺序会变,规律不变)。
