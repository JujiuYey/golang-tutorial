# 01: goroutine 基础与调度模型

这篇解决一个心智模型的切换:从 JS 的"单线程 + 事件循环"切到 Go 的"真·多任务同时跑"。学完你能用 `go` 关键字启动并发任务,并且躲开三个新手必踩的坑:main 跑完全家陪葬、goroutine 里 panic 炸掉整个程序、拿不到返回值。

先给 JS 用户校准坐标:JS 的"并发"是一个线程在事件循环里轮流处理回调——同一时刻只有一段 JS 代码在执行。Go 的 goroutine 是**真并发**:成千上万个 goroutine 被 Go 运行时调度到多个 CPU 核上,可能真的同时执行。这是本模块所有内容的地基,也是和 JS 差异最大的地方——JS 里"不加锁也没事"的直觉,在 Go 里会直接翻车(第 08 篇细讲)。

**先说结论:并发输出天然不稳定。本模块凡是多个 goroutine 各自打印的示例,输出顺序每次运行都可能不同,文中贴的都是某一次的实际运行结果。**

---

## 1. 启动 goroutine:一个 `go` 字

在任何函数调用前面加 `go`,这个调用就会被扔到一个新的 goroutine(Go 运行时管理的轻量执行单元)里去跑,当前代码**不等它**,直接往下走:

```go
package main

import "fmt"

func printMessage(msg string) {
	fmt.Println(msg)
}

func main() {
	go printMessage("hello")
	fmt.Println("main 结束")
}
```

```text
main 结束
```

### 🕳️ 坑:hello 没打印出来

以为会怎样:启动了 goroutine,"hello" 总会打印吧。
实际怎样:多数情况下只打印 "main 结束","hello" 消失了(上面就是一次真实运行结果)。
为什么:**main 函数一返回,整个程序立刻退出**,不管其他 goroutine 干到哪了。`go printMessage("hello")` 只是"预约了一个任务",main 不等它开跑就先结束了。

对比 JS:Node 里 `setTimeout(fn, 0)` 后脚本跑完,进程会等定时器执行完才退出;Go 的 main 可没这么客气——**main 是老大,老大走了全家散伙**。

```text
go printMessage("hello")
// ┬  ──────────┬────────
// │            └─ 普通的函数调用
// └─ 加上 go:这个调用放进新 goroutine 执行,当前代码不等它
```

---

## 2. 用 Sleep 等 goroutine:能跑,但别当真

想看到 "hello",最粗暴的办法是让 main 睡一会儿:

```go
func main() {
	go printMessage("hello")
	time.Sleep(100 * time.Millisecond)
	fmt.Println("main 结束")
}
```

```text
hello
main 结束
```

这次 "hello" 出来了。但 Sleep 是**猜时间**,不是同步:任务可能 1 毫秒就完了(白等 99 毫秒),也可能 200 毫秒还没完(照样丢输出)。本篇后面的演示代码为了简单会继续用 Sleep,但记住:**正经代码用 `WaitGroup`(下一篇)、channel(第 03 篇)或 context(第 07 篇)表达"等谁、等到什么时候"。**

对比 JS:这相当于用 `setTimeout` 猜一个毫秒数来"等" Promise 完成,而不是 `await` 它——你在 JS 里不会这么干,在 Go 里也不该。

---

## 3. 匿名函数 goroutine

不想专门定义一个函数,就地写匿名函数(03 模块学过的闭包)也行:

```go
go func() {
	fmt.Println("run in goroutine")
}()
```

注意最后那个 `()`——没有它只是定义了函数,加上才是调用,只不过这次调用发生在新 goroutine 里。可以传参数:

```go
func main() {
	name := "worker-1"

	go func(n string) {
		fmt.Println("我是", n)
	}(name)

	time.Sleep(100 * time.Millisecond)
}
```

```text
我是 worker-1
```

```text
go func(n string) { ... }(name)
//      ────┬───           ──┬─
//     ①声明参数        ②启动时把 name 的值传进去(当场求值)
```

把需要的值作为参数传进去,比闭包直接捕获外部变量更清晰——参数在 `go` 那一刻就复制定格了(和 defer 参数的"当场求值"是同一个规则,06-03 篇学过)。

---

## 4. goroutine 调用和普通调用的区别

同一个函数,两种调法,行为完全不同:

```go
func doWork() {
	fmt.Println("working...")
}

func main() {
	doWork() // 普通调用:等它做完才往下走
	fmt.Println("A: doWork 返回了")

	go doWork() // goroutine:不等,立刻往下走
	fmt.Println("B: 已启动,不知道做没做完")

	time.Sleep(100 * time.Millisecond)
}
```

输出顺序每次可能不同(B 和第二个 working... 谁先出现看调度),下面是某一次的实际运行结果:

```text
working...
A: doWork 返回了
B: 已启动,不知道做没做完
working...
```

普通调用是"打电话等对方办完";`go` 调用是"发个消息就挂断"。挂断意味着三件事从此没了着落:**结果怎么拿回来、出错了谁知道、它什么时候结束**——这三个问题贯穿整个模块。

对比 JS:`go doWork()` 很像调用 async 函数却不 `await`(fire-and-forget)。区别在于 JS 里那个"任务"还是要排队等你的同步代码让出线程;Go 里它可能**立刻在另一个 CPU 核上跑起来**。

---

## 5. 调度模型的直觉:很多 goroutine,少量线程

Go 运行时把成千上万个 goroutine 调度到少量操作系统线程上跑(所以 goroutine 便宜:初始栈只有几 KB,开几万个不心疼;线程一个就要 MB 级)。

你不需要管线程,需要管的是这四件事:

- goroutine 的**数量**是否失控(第 04、11 篇讲怎么限)。
- goroutine 能不能**退出**(第 10 篇讲泄漏)。
- goroutine 之间怎么**通信**(第 03-06 篇的 channel)。
- 共享数据是否**安全**(第 08-09 篇的锁和 atomic)。

**一句话总结:`go` 关键字只负责"启动",启动之后的生老病死全要你自己设计。**

---

## 6. GOMAXPROCS:并发 ≠ 并行

`GOMAXPROCS` 是"同时执行 Go 代码的操作系统线程数上限",默认等于 CPU 核数,绝大多数业务代码不用碰:

```go
runtime.GOMAXPROCS(4) // 知道有这回事即可,一般不手动设
```

顺便掰清两个词:

| 概念 | 意思 | 类比 |
|------|------|------|
| 并发(concurrency) | 多个任务在时间上**交错**推进 | 一个厨师轮流照看 4 口锅 |
| 并行(parallelism) | 多个任务**同一时刻**真的在跑 | 4 个厨师各管一口锅 |

JS 的事件循环是纯并发(一个厨师);goroutine 支持并发,而是否并行取决于 CPU 核数和调度。写代码时按"可能并行"的最坏情况防护——这正是需要锁的原因。

---

## 7. 循环里启动 goroutine:参数何时定格

```go
func main() {
	for i := 0; i < 3; i++ {
		go func(v int) {
			fmt.Println(v)
		}(i)
	}
	time.Sleep(100 * time.Millisecond)
}
```

输出顺序每次可能不同,下面是某一次的实际运行结果:

```text
2
0
1
```

`go func(v int){...}(i)` 在启动那一刻把 `i` 的值复制给 `v`,每个 goroutine 拿到自己那份——0、1、2 一个不少,只是打印顺序看调度心情。

那闭包直接捕获 `i` 呢?

```go
for i := 0; i < 3; i++ {
	go func() {
		fmt.Println(i) // 闭包直接捕获循环变量
	}()
}
time.Sleep(100 * time.Millisecond)
```

输出顺序每次可能不同,某一次的实际运行结果:

```text
1
0
2
```

也是 0、1、2 各一个。**但这是 Go 1.22 之后的行为**:从 1.22 起,`for` 的循环变量每轮迭代都是一个新变量。在更老的 Go 上,三个闭包共享同一个 `i`,循环跑完 `i` 已经是 3,很可能打印 `3 3 3`——老代码里常见的 `i := i`(遮蔽一份副本)就是为了绕这个坑。你读旧代码会遇到它,自己写新代码推荐传参写法:意图更明显,还能防住"捕获的变量在别处继续被改"的变体问题。

---

## 8. goroutine 没有返回值

```go
// result := go compute() // 编译不过:go 语句没有值
```

`go` 是一条语句,不是表达式——这可能是 JS 转 Go 最大的落差:JS 里 `const p = asyncFn()` 拿到一个 Promise,回头 `await` 就行;**Go 里没有 Promise 这个"装未来结果的盒子"**,想拿结果得自己搭路:

- 用 channel 送回来(第 03 篇,最常用,channel 就是你要找的"Promise 平替")。
- 写进加锁保护的共享变量(第 08 篇)。
- 更高层用 errgroup 之类的库(本模块先用标准库把地基打牢)。

---

## 9. goroutine 里的 panic 会炸掉整个程序

### 🕳️ 坑:一个 goroutine panic,全员陪葬

```go
func main() {
	go func() {
		panic("boom")
	}()

	time.Sleep(100 * time.Millisecond)
	fmt.Println("main 还活着吗?")
}
```

```text
panic: boom

goroutine 19 [running]:
main.main.func1()
	./e0107.go:10 +0x2c
created by main.main in goroutine 1
	./e0107.go:9 +0x24
exit status 2
```

"main 还活着吗?" 永远没机会打印——**任何一个 goroutine panic 且没 recover,整个进程直接崩**。对比 JS:一个 Promise reject 了,顶多控制台报个 unhandledRejection,其他代码照跑;Go 没有这种隔离。

顺便读一下这份报错:`goroutine 19 [running]` 是肇事者,`created by main.main in goroutine 1` 告诉你它是谁启动的——排查并发 panic 时这行最有用。

想挡住 panic,必须**在那个 goroutine 内部** defer recover(06 模块的知识,跨 goroutine 接不住):

```go
func main() {
	go func() {
		defer func() {
			if r := recover(); r != nil {
				fmt.Println("recovered:", r)
			}
		}()
		panic("boom")
	}()

	time.Sleep(100 * time.Millisecond)
	fmt.Println("main 还活着")
}
```

```text
recovered: boom
main 还活着
```

老规矩(06 模块讲过):recover 只用来兜底意外崩溃,普通失败该返回 error 或通过 channel 传出去,别用 recover 掩盖。

---

## 10. 启动前的灵魂五问

`go` 只要一个字,但每次敲下它之前,把这五个问题过一遍:

1. 这个 goroutine **什么时候退出**?
2. 它出错了,**错误传给谁**?
3. 调用方想取消,它**怎么知道**?
4. 它碰**共享数据**了吗?
5. 循环里启动时,**数量有没有上限**?

现在答不上来没关系——这五个问题就是本模块后面 11 篇的目录。但从今天起,启动 goroutine 前先想一遍,是并发代码不出事故的第一习惯。

---

## 本篇重点

- [ ] `go f()` 把调用扔进新 goroutine,不等它;JS 是单线程事件循环,Go 是真并发,可能多核同时执行。
- [ ] main 返回则全程序退出,不管其他 goroutine;Sleep 等待是猜时间,正经同步用 WaitGroup/channel。
- [ ] `go` 启动时参数当场求值(同 defer);Go 1.22+ 循环变量每轮独立,老代码的 `i := i` 是历史遗留。
- [ ] `go` 语句没有返回值,Go 没有 Promise——结果要靠 channel 等机制自己送回来。
- [ ] 任意 goroutine 的未 recover panic 会炸掉整个进程;recover 必须写在该 goroutine 内部。

---

## 练习

1. 启动 3 个 goroutine 打印不同名称,用 `WaitGroup` 等待它们完成(先预习下一篇的基本用法,或暂时用 Sleep 顶替)。
2. 把循环变量作为参数传给 goroutine,观察输出;多跑几次,确认顺序是否每次相同。
3. 故意在 goroutine 中 panic,观察程序行为和报错里的 `created by` 行。
4. 给 goroutine 加上 defer recover,观察程序是否继续运行。
5. 写一个函数,用注释说明它启动的 goroutine 如何退出(对照第 10 节的五问)。

提示:练习 2 多跑几次的意义在于亲眼看到"顺序不稳定";练习 3、4 记得让 main 睡一小会儿,否则 main 可能先退出,panic 都来不及发生。
