# 04: panic、recover 与程序边界

这篇解决两个问题：程序崩溃时打印的那一大坨 `panic: ...` 到底是什么、怎么读；以及 `panic` 和 `error` 的分工——**什么时候该崩，什么时候该返回错误**。先说结论：普通业务出错一律返回 `error`，`panic` 留给"程序没法正常继续"的场面。

---

## 1. panic：拉响警报，程序崩溃

`panic` 表示程序进入了无法正常继续的状态——执行立刻中断，沿调用栈向上炸，最后整个程序退出：

```go
package main

import "fmt"

func main() {
    fmt.Println("开始")
    panic("发生严重错误")
    // 这之后的代码永远不会执行
}
```

```text
开始
panic: 发生严重错误

goroutine 1 [running]:
main.main()
	demo/main.go:7 +0x60
exit status 2
```

学会读这坨输出：第一行是 panic 携带的信息；下面是**调用栈**——出事时程序走到了哪（`main.main()`、文件名和行号 `main.go:7`）；`exit status 2` 表示程序非正常退出。行号能直接定位炸点，这是排查 panic 的第一线索（`+0x60` 之类的偏移量因机器而异，不用管）。

其实你可能已经见过 panic 了——数组越界就是运行时自己拉的警报：

```go
nums := []int{1, 2, 3}
fmt.Println(nums[5])
```

```text
panic: runtime error: index out of range [5] with length 3

goroutine 1 [running]:
main.main()
	demo/main.go:7 +0x24
exit status 2
```

对比 JS：`panic` ≈ `throw`，但气质完全不同。JS 里 throw 是家常便饭，Go 里 panic 是**异常事件**——正常的 Go 代码可以整个项目一个 panic 都不写。

---

## 2. panic 不会跳过 defer

程序崩溃前，还讲武德：已经注册的 `defer` 会先执行完：

```go
func main() {
    defer fmt.Println("defer")
    panic("boom")
}
```

```text
defer
panic: boom

goroutine 1 [running]:
main.main()
	demo/main.go:7 +0x5c
exit status 2
```

看顺序：`defer` 先打印，然后才崩。这保证了文件该关的关、锁该放的放——**上一篇的资源清理承诺，在 panic 面前依然有效**。这也是下一节 recover 能存在的前提。

---

## 3. recover：在 defer 里接住 panic

`recover` 能捕获正在向上炸的 panic，让程序活下来。它有个硬性规则：**必须写在 defer 的函数里**才有效——因为 panic 发生后，唯一还会执行的代码就是 defer：

```go
func safeCall() {
    defer func() {
        if r := recover(); r != nil {
        //  ────┬────────  ────┬───
        //  ①接住panic的值  ②没panic时返回nil,所以要判断
            fmt.Println("捕获到 panic:", r)
        }
    }()

    panic("故意的 panic")
}

func main() {
    safeCall()
    fmt.Println("程序还活着")
}
```

```text
捕获到 panic: 故意的 panic
程序还活着
```

流程：`panic` 触发 → `safeCall` 的 defer 执行 → `recover()` 接住 panic 的值（`"故意的 panic"`）→ 警报解除，`safeCall` 正常返回 → `main` 继续跑。

对比 JS：`defer + recover` ≈ `try/catch`，但注意用途的差别——JS 的 catch 是常规操作，Go 的 recover 是**少数边界位置**才用的兜底手段（第 5、6 节）。

---

## 4. recover 的边界：只管自己这条 goroutine

goroutine 是 Go 的并发执行单元（可以先粗糙地理解为"另一条并行的执行线"，**并发模块会细讲**）。这里先埋一个重要事实：

```go
func main() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("捕获到:", r)
        }
    }()

    go func() {              // 开一条新的执行线
        panic("goroutine panic")
    }()

    time.Sleep(100 * time.Millisecond)
}
```

```text
panic: goroutine panic

goroutine 35 [running]:
main.main.func2()
	demo/main.go:16 +0x2c
created by main.main in goroutine 1
	demo/main.go:15 +0x40
exit status 2
```

`main` 里明明有 recover，程序照样崩了——**recover 只能接住自己这条 goroutine 里的 panic**，隔壁炸了它够不着。任何一条 goroutine 的 panic 没人接，整个程序一起死。现在记住结论即可，写并发代码时再回来体会。

---

## 5. 什么时候用 panic

判断标准一句话：**"预料之中的失败"返回 error，"不该发生的事"才 panic。**

适合 panic：

- 程序启动时发现配置根本没法用——起都起不来，早崩早发现。
- 违反内部不变量：代码逻辑保证不可能出现的状态出现了，说明有 bug。
- 库内部用 panic 简化深层控制流，但在包的边界 recover 回来转成 error（第 6 节）。

不适合 panic（这些都是**正常世界的一部分**，返回 error）：

- 用户输入错误
- 文件不存在
- 网络请求失败
- 数据库查询没有结果

🕳️ 坑：JS 转来的直觉是"出错就 throw"。在 Go 里把用户输错密码写成 panic，相当于"顾客点的菜卖完了，厨师直接把餐厅炸了"。菜卖完了是营业的一部分——告诉顾客（返回 error）就好。

---

## 6. 包边界恢复：把 panic 封印成 error

如果一段代码可能 panic（比如执行外部传入的函数），可以在边界处用 recover 兜底，**把 panic 转成普通的 error 交出去**——外面的世界只看到 error，不用面对崩溃：

```go
func runSafely(fn func()) (err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("panic: %v", r)   // defer 修改命名返回值,第 03 篇的技巧
        }
    }()

    fn()
    return nil
}

func main() {
    err := runSafely(func() { panic("内部出事了") })
    fmt.Println(err)

    err = runSafely(func() { fmt.Println("一切正常") })
    fmt.Println(err)
}
```

```text
panic: 内部出事了
一切正常
<nil>
```

注意这里把前面的知识串起来了：**命名返回值**（第 03 章）+ **defer 改返回值**（本章第 03 篇）+ **recover**（本篇）。panic 的传播被拦在 `runSafely` 里，调用方拿到的是熟悉的 `error` 值。真实世界里，HTTP 框架就在用这一招：某个请求处理函数 panic 了，框架 recover 住、返回 500，服务器整体不死。

---

## 本篇重点

- `panic` 中断执行、沿调用栈向上炸、最终程序退出；崩溃输出里的文件名：行号是定位炸点的第一线索。
- panic 时已注册的 defer 依然执行——资源清理不会被跳过，这也是 recover 的生存空间。
- `recover` 只有写在 defer 函数里才有效，接住 panic 后程序继续活。
- recover 只管自己这条 goroutine，别的 goroutine 崩了整个程序陪葬（并发模块细讲）。
- 分工：预料之中的失败（输入错、文件无、网络断）返回 error；不该发生的事才 panic；边界处可 recover 转 error。

---

## 练习

实现函数：

```go
func safeRun(fn func()) (err error)
```

要求：

1. 正常执行时返回 `nil`。
2. 如果 `fn` panic，捕获 panic 并返回 error。
3. 不要在普通业务逻辑里主动 panic。

提示：结构参照第 6 节；分别传一个正常函数和一个会 panic 的函数，验证两条路径的返回值。
