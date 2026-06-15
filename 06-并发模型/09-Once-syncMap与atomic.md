# 09: Once、sync.Map 与 atomic

除了 WaitGroup 和锁，`sync` 与 `sync/atomic` 还提供了一些专门工具。它们适合特定场景，不应该替代清晰的数据结构设计。

---

## 1. sync.Once

`sync.Once` 保证某个函数只执行一次。

```go
var once sync.Once
var config Config

func LoadConfig() Config {
    once.Do(func() {
        config = readConfig()
    })
    return config
}
```

即使多个 goroutine 同时调用 `LoadConfig`，`readConfig` 也只会执行一次。

---

## 2. Once 的错误处理

```go
var (
    once sync.Once
    cfg  Config
    err  error
)

func LoadConfig() (Config, error) {
    once.Do(func() {
        cfg, err = readConfig()
    })
    return cfg, err
}
```

如果第一次执行失败，`Once` 也会认为已经执行过。是否接受这种语义，要根据业务判断。

---

## 3. sync.Map

`sync.Map` 是并发安全的 map。

```go
var m sync.Map

m.Store("name", "alice")

v, ok := m.Load("name")
if ok {
    fmt.Println(v)
}

m.Delete("name")
```

`sync.Map` 的 key 和 value 类型都是 `any`，读取后通常需要类型断言。

---

## 4. sync.Map 的适用场景

`sync.Map` 适合一些特殊场景，比如：

- key 集合相对稳定，读多写少。
- 多 goroutine 访问不同 key，冲突较少。
- 缓存类数据，类型断言成本可以接受。

普通业务 map 通常优先用 `map + Mutex`，类型更清楚。

---

## 5. sync.Map Range

```go
m.Range(func(key, value any) bool {
    fmt.Println(key, value)
    return true
})
```

返回 `false` 可以停止遍历。

`Range` 期间看到的内容不一定是某个瞬间的完整快照。并发写入时，不要依赖它产生强一致视图。

---

## 6. atomic 计数

```go
var n atomic.Int64

n.Add(1)
fmt.Println(n.Load())
```

现代 Go 提供了类型化 atomic，例如 `atomic.Int64`、`atomic.Bool`、`atomic.Pointer[T]`。

如果使用旧式函数：

```go
var n int64
atomic.AddInt64(&n, 1)
fmt.Println(atomic.LoadInt64(&n))
```

---

## 7. atomic 适合简单状态

适合 atomic：

- 计数器。
- 开关标志。
- 简单统计值。
- 非常小的状态更新。

不适合 atomic：

- 多字段不变量。
- map、slice 这类复合结构。
- 需要和其他状态一起保持一致的值。

多字段一致性通常用 Mutex 更清晰。

---

## 8. atomic 和内存可见性

atomic 操作不仅是“不会被打断”，还提供并发读写的同步语义。

初学阶段不需要展开内存模型细节，但要记住：同一个变量如果用 atomic 写，就也应该用 atomic 读。不要混用普通读写。

---

## 9. Pool 简介

`sync.Pool` 用于复用临时对象，减少 GC 压力。

```go
var pool = sync.Pool{
    New: func() any {
        return new(bytes.Buffer)
    },
}

buf := pool.Get().(*bytes.Buffer)
buf.Reset()
defer pool.Put(buf)
```

`sync.Pool` 中的对象可能随时被 GC 清掉。它适合临时缓存，不适合保存必须存在的数据。

---

## 10. 选择建议

- 初始化一次：`sync.Once`。
- 普通共享 map：`map + Mutex`。
- 特殊并发缓存：考虑 `sync.Map`。
- 简单计数或标志：atomic。
- 多字段状态：Mutex。
- 临时对象复用：`sync.Pool`。

工具越底层，越要能说清楚为什么需要它。

---

## 练习

1. 用 `sync.Once` 实现只初始化一次的配置加载。
2. 处理 `Once` 初始化失败，说明第一次失败后是否允许重试。
3. 用 `sync.Map` 实现简单并发缓存。
4. 用 `atomic.Int64` 实现并发计数器。
5. 判断一个包含两个字段的不变量为什么不适合只用 atomic。
