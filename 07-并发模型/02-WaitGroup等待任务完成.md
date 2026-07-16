# 02: WaitGroup:等待一组任务完成

上一篇留了个尾巴:main 不等 goroutine,Sleep 又是猜时间。这篇给出正解 `sync.WaitGroup`——**让主 goroutine 精确等到"一组任务全部完成"那一刻**,不早不晚。学完你能写出并发任务的标准骨架,并躲开"计数加晚了"和"复制 WaitGroup"两个坑。

对比 JS:WaitGroup 干的活最接近 `Promise.all`——"这 N 件事都完成后再继续"。区别是 `Promise.all` 直接把结果数组给你,WaitGroup **只管等待,不管结果和错误**,后两样要靠 channel 自己搭(第 5、6 节)。

---

## 1. 基本用法:计数器归零就放行

WaitGroup 本质是个计数器:开工前 `+1`,干完一个 `-1`,`Wait()` 一直堵着,直到计数归零。

```go
func worker(id int, wg *sync.WaitGroup) {
	defer wg.Done()
	fmt.Println("worker", id, "完成")
}

func main() {
	var wg sync.WaitGroup

	for i := 0; i < 3; i++ {
		wg.Add(1)
		go worker(i, &wg)
	}

	wg.Wait()
	fmt.Println("all done")
}
```

三个 worker 的打印顺序每次可能不同,下面是某一次的实际运行结果:

```text
worker 2 完成
worker 1 完成
worker 0 完成
all done
```

worker 顺序随缘,但 **"all done" 保证垫底**——这就是 WaitGroup 给的承诺。三个方法:

| 方法 | 作用 | 谁调用 |
|------|------|--------|
| `Add(n)` | 计数 +n(登记 n 个任务) | 启动方 |
| `Done()` | 计数 -1(等价 `Add(-1)`) | 每个 worker 完成时 |
| `Wait()` | 阻塞到计数归零 | 等待方 |

---

## 2. 🕳️ 坑:Add 必须在 `go` 之前

以为会怎样:`Add(1)` 写在 goroutine 里面也一样,反正都会执行到。
实际怎样:main 可能在任何 worker 跑起来之前就冲到 `Wait()`,此时计数还是 0,`Wait()` **直接放行**——等了个寂寞。

```go
func main() {
	var wg sync.WaitGroup

	for i := 0; i < 3; i++ {
		go func(id int) {
			wg.Add(1) // 错误习惯:计数加晚了
			defer wg.Done()
			fmt.Println("worker", id, "完成")
		}(i)
	}

	wg.Wait()
	fmt.Println("all done")
}
```

这段代码每次运行结果都可能不同(取决于 worker 抢不抢得到时间),下面这次运行里 worker 一个都没来得及打印:

```text
all done
```

为什么:`go` 之后新 goroutine 什么时候开跑没有任何保证。**登记必须发生在等待之前**,所以口诀是:`Add` 写在 `go` 的前一行,和它同一个 goroutine。

对比 JS 帮助记忆:`Promise.all(promises)` 也得先把 promise 数组凑齐再传进去——你不会先 `all` 一个空数组再往里塞。

---

## 3. Done 交给 defer

```go
go func() {
	defer wg.Done()

	if err := doWork(); err != nil {
		return // 提前 return,defer 照样执行
	}
}()
```

worker 中间随便哪条路 `return`(甚至 panic),`defer wg.Done()` 都会执行(06-03 篇 defer 的承诺)。忘了 Done 或某条分支跳过了 Done,计数永远不归零,`Wait()` 就永远等下去——程序卡死。

**一句话总结:进门先 `defer wg.Done()`,后面爱怎么 return 怎么 return。**

---

## 4. 🕳️ 坑:WaitGroup 必须传指针

worker 是独立函数时,参数必须是 `*sync.WaitGroup`。写成值传递,编译器不拦你,但每个 worker 拿到的是**计数器的复印件**——它们减的是复印件,main 手里的原件永远是 1:

```go
func worker(wg sync.WaitGroup) { // 错误:值传递,拿到的是副本
	defer wg.Done()
	fmt.Println("working")
}

func main() {
	var wg sync.WaitGroup
	wg.Add(1)
	go worker(wg)
	wg.Wait()
	fmt.Println("all done")
}
```

```text
working
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [sync.WaitGroup.Wait]:
sync.runtime_SemacquireWaitGroup(0x104e4fff0?, 0x18?)
	runtime/sema.go:114 +0x38
sync.(*WaitGroup).Wait(0x14000092020)
	sync/waitgroup.go:206 +0xa8
main.main()
	./e0204.go:17 +0x74
exit status 2
```

见到 `fatal error: all goroutines are asleep - deadlock!` 别慌,这是 Go 运行时在说:"所有 goroutine 都在睡觉,没人能推进程序了"——worker 打印完就退出了,main 还在等一个永远不会归零的计数。这是死锁报错在本模块的首次登场,第 10 篇专门讲它。

这条规则是 04 模块"值传递会复制"的直接推论。**`sync` 包的家族成员(`WaitGroup`、`Mutex`、`RWMutex`、`Once`)都不能复制**,一律传指针或放在指针接收者的结构体里。

---

## 5. WaitGroup 不收集错误:配一条 error channel

WaitGroup 只回答"完了没",不回答"成没成"。错误得自己收——先剧透一下下篇的 channel,看个惯用法:

```go
func main() {
	var wg sync.WaitGroup
	errCh := make(chan error, 3) // 缓冲 = 任务数,发送不会卡住

	for i := 0; i < 3; i++ {
		wg.Add(1)
		go func(i int) {
			defer wg.Done()
			if err := doWork(i); err != nil {
				errCh <- err
			}
		}(i)
	}

	wg.Wait()
	close(errCh)

	for err := range errCh {
		fmt.Println("收到错误:", err)
	}
}
```

`doWork` 里让任务 1 故意失败,运行结果:

```text
收到错误: task 1 failed
```

流程:每个失败的 worker 往 `errCh` 里丢一个 error → 全员完工后关闭 channel → 统一遍历处理。channel 的语法细节(`<-`、`close`、`range`)接下来三篇细讲,这里先记住分工:**WaitGroup 管等待,channel 管传数据**。要"一个失败就取消其他任务"得靠 context,第 07 篇见。

---

## 6. 惯用模式:等完再关结果 channel

worker 用 channel 交结果、main 用 `range` 收结果时,有个鸡生蛋问题:`range` 要等 channel 关闭才结束,而 channel 得等所有 worker 交完才能关。标准解法是**雇个"关门员" goroutine**:

```go
func main() {
	results := make(chan int)
	var wg sync.WaitGroup

	for i := 1; i <= 3; i++ {
		wg.Add(1)
		go func(i int) {
			defer wg.Done()
			results <- i * i
		}(i)
	}

	go func() {
		wg.Wait()      // 等所有生产者退出
		close(results) // 再关闭,range 才能结束
	}()

	for v := range results {
		fmt.Println(v)
	}
}
```

输出顺序每次可能不同,下面是某一次的实际运行结果:

```text
1
4
9
```

为什么 `wg.Wait()` 要挪进单独的 goroutine?main 正忙着 `range` 收货呢——要是 main 自己去 `Wait`,就没人收货,worker 全堵在发送上,谁也动不了。这个模式第 05、11 篇会反复出现,先混个脸熟。

---

## 7. 计数不能变负数

`Done` 次数超过 `Add`,当场 panic:

```go
func main() {
	var wg sync.WaitGroup
	wg.Done()
}
```

```text
panic: sync: negative WaitGroup counter

goroutine 1 [running]:
sync.(*WaitGroup).Add(0x14000092000, 0xffffffffffffffff)
	sync/waitgroup.go:118 +0x264
sync.(*WaitGroup).Done(...)
	sync/waitgroup.go:156
main.main()
	./e0203.go:7 +0x30
exit status 2
```

记账原则:**一个 `Add(1)` 配一个 `Done()`**,像括号一样成对。多 Done 会 panic(上面),少 Done 会卡死(第 3 节)——两边都不能错。

---

## 8. WaitGroup 可以复用,但别交叉

一轮 `Wait()` 返回后,同一个 WaitGroup 可以再来一轮 `Add / go / Wait`,合法。但别在上一轮还没 `Wait` 完就混着开下一轮的任务——账就乱了。清晰的做法:**每组任务用一个局部 WaitGroup**,函数结束它就消失,想乱都难。

---

## 9. 🕳️ 坑:WaitGroup 不保护共享变量

以为会怎样:都 `Wait()` 过了,1000 次 `total++` 该稳稳等于 1000。
实际怎样:

```go
func main() {
	var total int
	var wg sync.WaitGroup

	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			total++ // 没有任何保护
		}()
	}

	wg.Wait()
	fmt.Println("total =", total)
}
```

结果每次运行都可能不同,下面是某一次的实际运行结果:

```text
total = 984
```

丢了 16 次!为什么:WaitGroup 只保证"main 等到了所有 worker 结束",**不保证 worker 们同时改 `total` 时不打架**。`total++` 是"读→加→写"三步,两个 goroutine 同时读到旧值就会互相覆盖——这叫数据竞争(data race)。用竞争检测器跑同一份代码,Go 会直接点名(报告较长,截取首尾):

```text
$ go run -race e0207.go
==================
WARNING: DATA RACE
Read at 0x00c000010178 by goroutine 11:
  main.main.func1()
      e0207.go:16 +0x68

Previous write at 0x00c000010178 by goroutine 7:
  main.main.func1()
      e0207.go:16 +0x78
...(省略第二组同类报告)
total = 847
Found 2 data race(s)
exit status 66
```

JS 里从没见过这种事,因为单线程下 `total++` 永远不会被人插队——**这是 JS 直觉在 Go 里最危险的一处失灵**。怎么修(Mutex、atomic、channel 汇总)是第 08、09 篇的主题,本篇先记住:**WaitGroup 管等待,不管数据安全**。

---

## 本篇重点

- [ ] WaitGroup = 任务计数器:`Add` 登记、`Done` 销账、`Wait` 堵到归零;作用类似 `Promise.all`,但只管等待不管结果。
- [ ] `Add` 必须在 `go` 之前、和等待方同一个 goroutine;`Done` 交给 defer,保证每条退出路径都销账。
- [ ] WaitGroup(以及 sync 全家)不能复制,传参一律用指针,否则 worker 减的是副本,`Wait` 卡死。
- [ ] 多 Done 会 panic,少 Done 会死锁;一个 `Add(1)` 配一个 `Done()`,像括号一样成对。
- [ ] `Wait` 过 ≠ 数据安全:1000 次无保护的 `total++` 真实跑出了 984,数据竞争要靠第 08 篇的锁来治。

---

## 练习

1. 用 `WaitGroup` 等待 5 个 worker 完成,worker 各打印自己的编号。
2. 故意漏掉一个 `Done`,观察程序是否卡住、报什么错。
3. 故意多调用一次 `Done`,观察 panic 信息。
4. 用 `WaitGroup` 等待多个生产者结束后关闭结果 channel(照第 6 节的骨架)。
5. 写一段代码说明 `WaitGroup` 不能解决 `total++` 的数据竞争,用 `go run -race` 跑给自己看。

提示:练习 2 的现象和第 4 节的死锁报错是同一个;练习 5 多跑几次普通版,观察结果是不是每次都不一样——不稳定本身就是竞争的信号。
