# 08: Mutex 与 RWMutex：保护共享状态

goroutine 可以共享内存。只要多个 goroutine 同时访问同一份数据，并且至少一个是写操作，就需要同步。

---

## 1. 数据竞争示例

```go
var counter int

for i := 0; i < 1000; i++ {
    go func() {
        counter++
    }()
}
```

`counter++` 不是原子操作，它包含读、加、写。多个 goroutine 同时执行会产生数据竞争。

---

## 2. 使用 Mutex

```go
var (
    counter int
    mu      sync.Mutex
)

func Inc() {
    mu.Lock()
    defer mu.Unlock()
    counter++
}
```

`Lock` 和 `Unlock` 之间是临界区。同一时刻只有一个 goroutine 能进入。

---

## 3. defer Unlock

```go
mu.Lock()
defer mu.Unlock()
```

这样可以确保函数提前返回时也会释放锁。

如果临界区非常短且函数很热，也可以手动 Unlock，但初学阶段优先用 defer 保证正确。

---

## 4. 锁保护的是不变量

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

锁保护的是 `balance` 的一致性。所有读写 `balance` 的方法都应该使用同一把锁。

---

## 5. 读取也需要锁

```go
func (a *Account) Balance() int {
    a.mu.Lock()
    defer a.mu.Unlock()
    return a.balance
}
```

如果一个 goroutine 写，另一个 goroutine 无锁读，也仍然是数据竞争。

---

## 6. 不要复制 Mutex

```go
type Counter struct {
    mu sync.Mutex
    n  int
}
```

包含 Mutex 的类型通常应该用指针接收者：

```go
func (c *Counter) Inc() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.n++
}
```

复制含锁结构体可能导致锁和被保护数据关系混乱。

---

## 7. RWMutex

`sync.RWMutex` 区分读锁和写锁。

```go
type Cache struct {
    mu    sync.RWMutex
    items map[string]string
}

func (c *Cache) Get(key string) (string, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    v, ok := c.items[key]
    return v, ok
}

func (c *Cache) Set(key, value string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.items[key] = value
}
```

多个读锁可以同时持有。写锁会排斥读锁和写锁。

---

## 8. RWMutex 不一定更快

读非常多、写很少、临界区较长时，`RWMutex` 可能有帮助。

如果读写都很频繁，或者临界区很短，普通 `Mutex` 可能更简单、更快。

先追求正确，再用基准测试判断是否需要换锁。

---

## 9. 避免持锁做慢操作

```go
mu.Lock()
data := snapshot()
mu.Unlock()

err := callRemote(data)
```

不要在持锁期间做网络请求、磁盘 IO、长时间计算。持锁时间越长，其他 goroutine 等待越久，也越容易形成复杂死锁。

---

## 10. channel 和 Mutex 怎么选

常见判断：

- 保护共享状态，用 Mutex 更直接。
- 表达任务流、结果流、取消信号，用 channel 更自然。
- 一个 goroutine 独占状态，其他 goroutine 通过 channel 发请求，也是一种好模式。

不要为了避开锁，把简单状态更新强行改成复杂 channel 流程。

---

## 练习

1. 写一个有数据竞争的计数器，用 `go test -race` 验证。
2. 用 `sync.Mutex` 修复计数器。
3. 实现 `Account`，支持 `Deposit`、`Withdraw`、`Balance`。
4. 实现 `Cache`，用 `RWMutex` 保护 map。
5. 写一段代码说明无锁读取共享变量也会产生数据竞争。
