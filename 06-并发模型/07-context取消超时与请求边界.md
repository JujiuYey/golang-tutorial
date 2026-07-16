# 07: context:取消、超时与请求边界

前两篇你已经手搓过"叫停"机制:done channel 广播 + select 监听。这篇学它的官方成品 `context.Context`——把**取消信号、截止时间、请求级小数据**打包成一个对象,顺着调用链一路传下去。学完你能让"用户断开连接"这样的上游事件,一层层传到最底下的 goroutine,让它们全体及时收工。

对比 JS:context 的取消部分就是 `AbortController` / `AbortSignal` 的 Go 版——`cancel()` ≈ `controller.abort()`,`<-ctx.Done()` ≈ 监听 `signal`,连"往 fetch 里传 signal"的用法都对得上(Go 的 `http.NewRequestWithContext`)。Go 把它更进一步:**标准库和主流库的每个阻塞函数都接受 ctx**,取消是全生态的通用语言。

---

## 1. context 长什么样、放在哪

约定俗成:**ctx 是函数的第一个参数,名字就叫 ctx**:

```go
func Fetch(ctx context.Context, url string) ([]byte, error) {
	// ...
}
```

它管三件事:

- **取消**:调用方说"别干了",下游能收到。
- **超时/截止时间**:到点自动取消。
- **请求范围的小数据**:如 trace id(第 7 节,慎用)。

🕳️ 坑:别把 ctx 存进 struct 当长期字段。ctx 跟着**一次请求/一次操作**走,操作结束它就该失效——存进 struct 的 ctx 会比它的请求活得久,取消语义就乱了。

---

## 2. 根 context:Background 和 TODO

ctx 是树状的:后面每个"带取消的 ctx"都从某个父 ctx 派生。树根有两个:

```go
ctx := context.Background() // 正牌树根:main、测试、顶层初始化用
ctx := context.TODO()       // 占位符:还没想好该传什么时先顶着
```

两者行为完全一样(永不取消、没有截止时间)。区别只在**给人看的语义**:`TODO` 是便签"这里欠一个真正的 ctx",重构时全局搜 `context.TODO()` 就能找到欠账。

---

## 3. WithCancel:手动叫停

```go
ctx, cancel := context.WithCancel(context.Background())
//   ──┬───────────────────┬─────
//     │                   └─ 从父 ctx 派生出可取消的子 ctx
//     └─ 返回两个:ctx 给下游用,cancel 留在自己手里
```

调用 `cancel()`,`ctx.Done()` 这个 channel 就会被关闭——对,就是上一篇 close(done) 广播那一套,context 内部就是这么实现的:

```go
func main() {
	ctx, cancel := context.WithCancel(context.Background())

	go func() {
		<-ctx.Done() // 堵在这,等取消广播
		fmt.Println("worker: 收到取消,", ctx.Err())
	}()

	time.Sleep(50 * time.Millisecond)
	cancel() // 广播:所有拿着这个 ctx 的人,收工
	time.Sleep(50 * time.Millisecond)
}
```

```text
worker: 收到取消, context canceled
```

`ctx.Err()` 在取消后返回原因,这里是 `context.Canceled`。

规矩:**谁派生,谁负责调 cancel**,而且标准姿势是拿到手立刻 `defer cancel()`——就算这条路上没人主动取消,defer 也会在函数退出时释放 ctx 树上的资源(03-07 篇 defer 的老本行)。

---

## 4. WithTimeout:到点自动叫停

```go
func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 200*time.Millisecond)
	defer cancel()

	select {
	case <-time.After(1 * time.Second): // 假装任务要 1 秒
		fmt.Println("任务完成")
	case <-ctx.Done():
		fmt.Println("退出:", ctx.Err())
	}
}
```

```text
退出: context deadline exceeded
```

200ms 一到,`ctx.Done()` 自动关闭,`ctx.Err()` 返回 `context.DeadlineExceeded`。对比上一篇的 `time.After` 超时:那是"这一次 select 等多久",WithTimeout 是"**这一整个操作**(可能穿过 N 层函数、N 个 goroutine)总共多久"——超时预算跟着 ctx 走,传到哪管到哪。还有个孪生兄弟 `WithDeadline`,把"多久之后"换成"到几点"。

🕳️ 坑:"反正会自动超时,cancel 不调也行吧?"——不行。任务提前完成时,不 cancel 的话定时器和 ctx 节点要一直挂到超时才释放。**`defer cancel()` 无条件写,一行都别省。**

---

## 5. worker 的标准形态:for-select + ctx.Done

把上一篇 for-select 心跳里的 done 换成 `ctx.Done()`,就是生产级 worker:

```go
func worker(ctx context.Context, jobs <-chan int) {
	for {
		select {
		case j, ok := <-jobs:
			if !ok {
				fmt.Println("jobs 关闭,worker 正常收工")
				return
			}
			fmt.Println("处理 job", j)
		case <-ctx.Done():
			fmt.Println("被取消,worker 退出:", ctx.Err())
			return
		}
	}
}

func main() {
	ctx, cancel := context.WithCancel(context.Background())
	jobs := make(chan int)

	go worker(ctx, jobs)

	jobs <- 1
	jobs <- 2
	cancel() // 不发了,直接叫停
	time.Sleep(50 * time.Millisecond)
}
```

```text
处理 job 1
处理 job 2
被取消,worker 退出: context canceled
```

两条退出路径:jobs 正常关闭(`ok == false`)是"活干完了",ctx 取消是"别干了"——**长期运行的 goroutine 两条都要有**。

---

## 6. 发送也要监听取消

上一篇第 6 节警告过:没人收货的发送会永久卡住。生产者的每一次发送都该和 `ctx.Done()` 二选一:

```go
func produce(ctx context.Context, out chan<- int) error {
	for i := 0; ; i++ {
		select {
		case out <- i: // 交货
		case <-ctx.Done(): // 或者收到叫停
			return ctx.Err()
		}
	}
}

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 100*time.Millisecond)
	defer cancel()

	out := make(chan int)
	errCh := make(chan error, 1)
	go func() { errCh <- produce(ctx, out) }()

	fmt.Println(<-out, <-out, <-out) // 只收 3 个,之后不收了
	fmt.Println("producer 退出:", <-errCh)
}
```

```text
0 1 2
producer 退出: context deadline exceeded
```

main 收完 3 个就不收了——要是 produce 里写的是裸的 `out <- i`,它就从此卡死(goroutine 泄漏,第 10 篇的主角)。有了 ctx 这条逃生通道,超时一到它体面退出。

---

## 7. context value:能用,但管住手

ctx 还能挂少量键值对,顺着调用链隐式传递:

```go
type traceIDKey struct{} // 私有类型做 key,防撞名

func handle(ctx context.Context) {
	traceID, ok := ctx.Value(traceIDKey{}).(string) // 取出来要断言
	fmt.Println("处理请求, traceID =", traceID, ok)
}

func main() {
	ctx := context.WithValue(context.Background(), traceIDKey{}, "abc-123")
	handle(ctx)
}
```

```text
处理请求, traceID = abc-123 true
```

两个细节:key 用私有空结构体类型(不同包的 key 永不相撞);取值返回 `any`,要类型断言(05 模块的招)。

🕳️ 坑:把 ctx value 当"隐形参数通道",什么都往里塞。规则很简单——**value 只放"请求的元数据"**(trace id、认证身份这类横切信息),**不放业务参数、配置、依赖对象**。判断标准:这个值影响函数的业务结果吗?影响,就该是显式参数;不影响、只是随行日志信息,才配进 ctx。滥用的代价是参数隐身:签名上看不出依赖什么,测试和重构全靠猜。

---

## 8. 传播:一条链上共用一个 ctx

context 的威力在"传"字——同一请求链路上的函数逐层把 ctx 递下去,顶层一声令下,最底层也能听见:

```go
func A(ctx context.Context) error { return B(ctx) } // 一路把 ctx 传下去
func B(ctx context.Context) error { return C(ctx) }

func C(ctx context.Context) error {
	select {
	case <-time.After(1 * time.Second): // 假装底层慢操作
		return nil
	case <-ctx.Done():
		return fmt.Errorf("C 层被叫停: %w", ctx.Err())
	}
}

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 100*time.Millisecond)
	defer cancel()

	fmt.Println(A(ctx))
}
```

```text
C 层被叫停: context deadline exceeded
```

main 定的 100ms 预算,穿过 A、B 两层直达 C。真实世界的版本:HTTP 请求进来,`r.Context()` 在用户断开连接时自动取消 → 传给数据库查询 → 查询立即中止,不浪费一毫秒。中间任何一层把 ctx 换成 `context.Background()`,链条就断了——下游从此对上游的取消充耳不闻。

---

## 9. 取消也是 error:按 05 模块的规矩处理

取消发生后,错误顺着返回值链传回来,用 05 模块的 `errors.Is` 区分两种原因:

```go
if errors.Is(err, context.Canceled) {
	// 有人主动 cancel(比如用户点了取消)
}
if errors.Is(err, context.DeadlineExceeded) {
	// 超时了
}
```

另一个惯用法:干重活之前先查一下"是不是已经被取消了",避免白干:

```go
if err := ctx.Err(); err != nil {
	return err // 已取消/已超时,立刻交还
}
```

**一句话总结:取消不是异常,是一种普通的、预期内的 error,按值处理即可。**

---

## 10. context 使用检查清单

写带 ctx 的代码,过一遍:

1. ctx 是不是第一个参数?(不是 struct 字段、不是全局变量)
2. 每个阻塞点(收、发、IO)是否都监听了 `ctx.Done()`?
3. `WithCancel/WithTimeout` 之后有没有 `defer cancel()`?
4. ctx 有没有原样传给每个下游调用?(中途换 Background = 断链)
5. ctx value 里是不是只有元数据,没混业务参数?

---

## 本篇重点

- [ ] context = 打包版的"done channel 广播 + 截止时间 + 请求元数据",Go 生态的 AbortSignal;永远做第一个参数,顺调用链传到底。
- [ ] `WithCancel` 手动叫停,`WithTimeout/WithDeadline` 到点自动叫停;拿到 cancel 一律 `defer cancel()`,提前完成也要释放。
- [ ] 长期 goroutine 的每个阻塞点(接收、发送)都要 select 一条 `<-ctx.Done()` 逃生通道——发送尤其容易被忘。
- [ ] 取消是普通 error:`ctx.Err()` 给出 `context.Canceled` 或 `context.DeadlineExceeded`,用 `errors.Is` 区分。
- [ ] ctx value 只放 trace id 这类请求元数据,key 用私有类型;业务参数走函数签名,别隐身。

---

## 练习

1. 用 `context.WithCancel` 停止一个 worker。
2. 用 `context.WithTimeout` 给任务加 200ms 超时,分别构造"按时完成"和"超时"两种结果。
3. 写一个 producer,发送时监听 `ctx.Done()`,验证接收方提前不收时它能退出。
4. 写函数链 `A(ctx) -> B(ctx) -> C(ctx)`,让取消信号传到最底层。
5. 用 context value 存取 trace id,并说明为什么不应该用它传普通参数。

提示:练习 2 用 `time.After` 假装任务耗时,调两个时长对比;练习 3 可以照第 6 节的骨架,把 `errCh` 的缓冲去掉试试会发生什么;练习 5 的 key 记得用私有结构体类型。
