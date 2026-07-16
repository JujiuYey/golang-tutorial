# 08: Mutex 与 RWMutex:保护共享状态

02 篇埋的雷该拆了:1000 次 `total++` 只加出 984。这篇学 `sync.Mutex`(互斥锁)——**给共享数据配一把锁,改数据前先排队拿锁,同一时刻只放一个人进去**。学完你能写出并发安全的计数器、账户、缓存,并明白一条 JS 里不存在的铁律:**共享数据,连"读"都要锁**。

先说清为什么 JS 没这东西:JS 单线程,一段同步代码跑完之前没人能插队,`total++` 天然安全。Go 的 goroutine 可能真的在两个 CPU 核上**同时**执行同一行代码——"没人能插队"的免费午餐没有了,得自己买锁。

---

## 1. 病灶:counter++ 不是一步

判断标准一句话:**多个 goroutine 访问同一份数据,其中至少一个在写 → 必须同步**。看病例:

```go
var counter int

func main() {
	var wg sync.WaitGroup
	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			counter++ // 读、加、写,三步,可能被插队
		}()
	}
	wg.Wait()
	fmt.Println("counter =", counter)
}
```

结果每次运行都可能不同,下面是某一次的实际运行结果:

```text
counter = 926
```

为什么丢了 74 次?`counter++` 在机器层面是三步:

```text
counter++
// ① 读出当前值        (读到 41)
// ② 加一              (算出 42)
// ③ 写回              (写入 42)
// 两个 goroutine 都在 ① 读到 41 → 都写回 42 → 两次 ++ 只涨了 1
```

这就是**数据竞争**(data race):结果取决于谁快谁慢,每次都不一样。

---

## 2. 药方:Mutex 圈出临界区

```go
var (
	counter int
	mu      sync.Mutex
)

func inc() {
	mu.Lock()
	defer mu.Unlock()
	counter++ // 临界区:同一时刻只有一人能进
}
```

1000 个 goroutine 跑 `inc()`:

```text
counter = 1000
```

分毫不差。`Lock()` 到 `Unlock()` 之间叫**临界区**:一个 goroutine 拿到锁,其他人到 `Lock()` 就排队,直到锁被还回来。三步"读加写"从此不会被人插在中间。

```text
mu.Lock()          ← 拿锁(别人拿着就排队等)
defer mu.Unlock()  ← 预约还锁(哪条路退出都还)
counter++          ← 现在,这段路上只有我
```

对比 JS:完全没有对应物——这是从 JS 到 Go 必须新建的心智模块。硬要找感觉的话,它像"单人卫生间的门锁":进去反锁,后来的在门口排队。

---

## 3. Unlock 交给 defer

```go
mu.Lock()
defer mu.Unlock()
```

和 `defer file.Close()`、`defer wg.Done()` 一个道理:中间随便哪条路 return,锁都保证归还。忘了还锁比忘了关文件严重得多——**所有等这把锁的 goroutine 永远堵死**。临界区极短、函数极热时手动 Unlock 能省一点开销,但现阶段一律 defer,先保证正确。

---

## 4. 锁和数据住一起:保护的是"一致性"

实战里锁很少是全局变量,而是和它保护的数据**同住一个 struct**:

```go
type Account struct {
	mu      sync.Mutex
	balance int
}

func (a *Account) Deposit(n int) {
	a.mu.Lock()
	defer a.mu.Unlock()
	a.balance += n
}
```

100 个 goroutine 并发各存 10 块:

```text
余额: 1000
```

要点:锁保护的不是"某一行代码",而是 **`balance` 这个数据的一致性**——所以所有读写 `balance` 的方法必须用**同一把** `a.mu`。字段声明挨着写(`mu` 紧贴它罩着的字段)是 Go 的惯例,读代码的人一眼看懂"这把锁管谁"。

---

## 5. 🕳️ 坑:读取也要锁

以为会怎样:我只是读一下余额,又不改,不锁没事吧。
实际怎样:一边有人在写、一边无锁地读,照样是数据竞争。race detector 抓个现行:

```go
var (
	mu      sync.Mutex
	balance int
)

func main() {
	var wg sync.WaitGroup
	wg.Add(1)
	go func() {
		defer wg.Done()
		mu.Lock()
		balance = 100 // 写:老老实实拿了锁
		mu.Unlock()
	}()

	fmt.Println(balance) // 读:没拿锁,自以为"只是看看"
	wg.Wait()
}
```

```text
$ go run -race e0805.go
0
==================
WARNING: DATA RACE
Write at 0x000100f530f8 by goroutine 7:
  main.main.func1()
      e0805.go:19 +0x74

Previous read at 0x000100f530f8 by main goroutine:
  main.main()
      e0805.go:23 +0xb8

Goroutine 7 (running) created at:
  main.main()
      e0805.go:16 +0xa8
==================
Found 1 data race(s)
exit status 66
```

为什么读也危险:无锁的读可能撞上写到一半的状态,而且编译器/CPU 对无同步的内存访问有重排自由,读到的东西没有任何保证。所以 `Balance()` 这样的读方法也要 `Lock`(第 4 节的 Account 就是这么写的)。**口诀:同一份数据,读写全走锁,一个都不豁免。**

---

## 6. 不要复制含锁的结构体

02 篇的教训在这里复现:`Mutex` 和 `WaitGroup` 一样**不可复制**——复制后就有两把独立的锁,各锁各的门,等于没锁。推论:含锁 struct 的方法一律用**指针接收者**:

```go
func (c *Counter) Inc() { // *Counter,不是 Counter
	c.mu.Lock()
	defer c.mu.Unlock()
	c.n++
}
```

值接收者每次调用复制整个 struct(连锁一起复制),锁形同虚设。`go vet` 能查出大部分"锁被复制"的写法,养成跑 vet 的习惯。

---

## 7. RWMutex:读多写少的优化

`sync.RWMutex` 把锁拆成两种:**读锁可以很多人同时拿,写锁独占**:

```go
type Cache struct {
	mu    sync.RWMutex
	items map[string]string
}

func (c *Cache) Get(key string) (string, bool) {
	c.mu.RLock() // 读锁:读者之间不互斥
	defer c.mu.RUnlock()
	v, ok := c.items[key]
	return v, ok
}

func (c *Cache) Set(key, value string) {
	c.mu.Lock() // 写锁:独占
	defer c.mu.Unlock()
	c.items[key] = value
}
```

三个 goroutine 并发读,输出顺序每次可能不同,下面是某一次的实际运行结果:

```text
reader 3 读到 go
reader 1 读到 go
reader 2 读到 go
```

规则表:

| 想进门的人 | 门里有读者 | 门里有写者 |
|-----------|-----------|-----------|
| 读者(RLock) | 一起进 | 等 |
| 写者(Lock) | 等 | 等 |

顺带敲个重点:**map 就是最典型的"必须加锁"数据**——多 goroutine 无保护地读写同一个 map,运行时会直接把程序杀掉(第 10 篇有真实案发现场)。`map + RWMutex` 打包成 struct,是 Go 缓存的标准件。

---

## 8. RWMutex 不一定更快

直觉是"区分读写肯定快",实际:RWMutex 自身的记账开销比 Mutex 大,只有**读远多于写、且临界区不算太短**时才划算。读写都频繁或临界区就一两行时,普通 Mutex 往往更简单也更快。

**决策顺序:先用 Mutex 保证正确 → 性能真有问题 → 基准测试证明 → 再考虑换 RWMutex。**

---

## 9. 🕳️ 坑:持锁做慢操作

```go
// 错:网络请求也圈进临界区,锁被占住几百毫秒
mu.Lock()
data := snapshot()
err := callRemote(data) // 所有人等你的网络往返
mu.Unlock()

// 对:临界区只干"拿数据"这一件事
mu.Lock()
data := snapshot()
mu.Unlock()

err := callRemote(data) // 慢活在锁外干
```

临界区里只放"必须原子完成的数据操作",网络、磁盘、长计算统统挪到锁外。持锁时间越长,排队越长,还容易和其他锁纠缠出死锁(第 10 篇)。**拿锁像借公共卫生间:速战速决,别在里面打电话。**

---

## 10. channel 和 Mutex 怎么选

到这里你手上有两套并发工具了,分工其实清楚:

| 场景 | 用什么 |
|------|--------|
| 保护一份共享状态(计数器、缓存、余额) | Mutex 直接了当 |
| 传递数据流、任务流、完成/取消信号 | channel 自然贴合 |
| 一个 goroutine 独占状态,别人发消息请求它读写 | channel(actor 风格,也很 Go) |

社区名言"不要通过共享内存来通信,要通过通信来共享内存"说的是**架构倾向**,不是"锁是二等公民"。就地更新一个计数器,加锁两行搞定;硬改成 channel 流程反而绕。**怎么简单怎么来。**

---

## 本篇重点

- [ ] 判断标准:多 goroutine 访问同一数据且至少一个写 → 必须同步;JS 单线程的"天然安全"在 Go 不存在。
- [ ] `counter++` 是读-加-写三步,会被插队(1000 次真实跑出 926);`Lock/defer Unlock` 圈出临界区,一次只放一人。
- [ ] 读也要锁:有人在写时无锁读同样是数据竞争,race detector 会点名;同一份数据读写全走同一把锁。
- [ ] Mutex 不可复制:和数据同住 struct、方法用指针接收者;`go vet` 能兜底。
- [ ] RWMutex 读共享写独占,只在读多写少时划算;临界区速战速决,慢操作放锁外;状态用锁、流程用 channel,怎么简单怎么来。

---

## 练习

1. 写一个有数据竞争的计数器,用 `go run -race`(或 `go test -race`)验证。
2. 用 `sync.Mutex` 修复计数器,确认结果稳定为预期值。
3. 实现 `Account`,支持 `Deposit`、`Withdraw`、`Balance`,并发调用后账目正确。
4. 实现 `Cache`,用 `RWMutex` 保护 map,支持 `Get`、`Set`。
5. 写一段代码说明无锁读取共享变量也会产生数据竞争。

提示:练习 3 注意 `Withdraw` 里"查余额 + 扣款"必须在同一次持锁内完成——分两次拿锁,中间就可能被人插队;练习 5 照第 5 节的思路,让读和写分属两个 goroutine。
