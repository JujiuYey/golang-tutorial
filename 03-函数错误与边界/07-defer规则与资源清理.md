# 07: defer 规则与资源清理

打开的文件要关、占用的连接要还——清理代码写在函数末尾，中间随便一个 `return` 就会跳过它。这篇学 `defer`：**把清理动作预约在"函数退出前"，不管从哪条路退出都保证执行**。学完你能正确写出"打开→defer 关闭→放心使用"的标准套路，并躲开 defer 的两个经典坑。

---

## 1. 基本用法：预约清理

`defer` 后面跟一个函数调用，意思是"这行先记下，等**当前函数退出前**再执行"：

```go
func read() {
    file, err := os.Open("test.txt")
    if err != nil {
        fmt.Println("open failed:", err)
        return                    // 打开都失败了,没东西可关
    }
    defer file.Close()            // ← 预约:read 退出前关掉文件

    data, _ := io.ReadAll(file)   // 中间随便写,随便 return
    fmt.Println(string(data))
}
```

```text
hello defer
```

好处：**开和关写在相邻两行**，一眼看到"这个资源有人负责关"。后面无论函数从哪个 `return` 离开、甚至中途出错，`Close` 都会被执行。

🕳️ 坑：`defer file.Close()` 必须写在错误检查**之后**。打开失败时 `file` 是无效的，先 defer 再检查，等于预约了"关一个不存在的文件"。口诀：**先确认资源到手，再预约清理**。

对比 JS：作用类似 `try { ... } finally { file.close() }`，但 defer 不用缩进包裹整段代码，清理声明就贴在资源创建旁边。

---

## 2. defer 执行顺序：后进先出

多个 defer 像**摞盘子**：后放上去的先拿下来（后进先出，LIFO）：

```go
func example() {
    fmt.Println("开始")

    defer fmt.Println("defer 1")
    defer fmt.Println("defer 2")
    defer fmt.Println("defer 3")

    fmt.Println("结束")
}
```

```text
开始
结束
defer 3
defer 2
defer 1
```

两个信息：① 所有 defer 都等到"结束"打印完、函数要退出了才跑；② 顺序和注册顺序**相反**。

为什么要反着来？想想嵌套资源：先开数据库、再开事务——清理时自然要先关事务、再关数据库。倒序刚好是"由内向外"的正确拆法。

---

## 3. 🕳️ 坑：defer 的参数当场求值

```go
func main() {
    i := 1
    defer fmt.Println(i)
    i = 2
}
```

你以为输出 `2`（defer 是最后执行的嘛），实际：

```text
1
```

为什么：`defer fmt.Println(i)` 这行执行时，参数 `i` **立刻**被求值，快照下来的是 `1`。之后被延迟的只是"打印这个快照"这个动作，`i` 再怎么改都与它无关。

拆开看这行发生了什么：

```text
defer fmt.Println(i)
//    ──────┬─────(─┬─)
//   ②这个调用延后执行  ①但参数现在就算好、冻结
```

---

## 4. defer + 闭包：执行时才读变量

想让 defer 看到变量的**最终值**？别把变量当参数传，包一层匿名函数（上一篇的闭包），让它执行时再去读：

```go
func main() {
    i := 1
    defer func() {
        fmt.Println(i)   // 没有参数快照,执行时才来读 i
    }()
    i = 2
}
```

```text
2
```

两种写法对照：

| 写法 | i 何时被读 | 输出 |
|------|-----------|------|
| `defer fmt.Println(i)` | defer 注册那一刻（快照） | 1 |
| `defer func() { fmt.Println(i) }()` | 函数真正退出时（闭包现读） | 2 |

**一句话总结：defer 传参 = 冻结当下，defer 闭包 = 读取最终。**

---

## 5. defer 修改命名返回值

第 02 篇说过"命名返回值可以配合 defer 修改返回的 error，后面细讲"——就是现在。因为命名返回值是函数一进门就存在的变量（第 02 篇陌生点①），而 defer 在 `return` 之后、函数真正退出之前还能跑，所以 defer 里可以**改写即将交出去的返回值**：

```go
func example() (err error) {
    defer func() {
        if err != nil {
            err = fmt.Errorf("example failed: %w", err)   // 出门前统一包一层上下文
        }
    }()

    return errors.New("original")
}

func main() {
    fmt.Println(example())
}
```

```text
example failed: original
```

执行流程：`return` 先把 `errors.New("original")` 赋给命名返回值 `err` → defer 运行，把 `err` 包装了一层 → 调用方拿到包装后的版本。

这个技巧适合"函数有很多条出错路径，想统一加一层上下文"的场景。但要克制——返回值在离开函数的路上被偷偷改掉，读代码的人容易被绕晕。**知道有这回事、看得懂别人这么写**，现阶段就够了。

---

## 6. Close 也会失败：defer 忽略的错误

很多资源的 `Close` 本身也返回 error，而这样写：

```go
defer file.Close()
```

`Close` 的返回值被丢掉了（相当于第 05 篇警告过的 `_` 吞错误）。什么时候能接受？

- **只读**文件：读都读完了，关闭失败通常无损失，忽略可接受——这也是最常见的写法。
- **写**文件、网络连接、压缩流：数据可能还在缓冲区里，`Close` 失败 = 数据可能没落盘。这种场景要处理 Close 的错误（常配合第 5 节的"defer 改命名返回值"技巧）。

现阶段记住区分"读可忽略、写要小心"即可。

---

## 本篇重点

- `defer` 把一个函数调用预约到"当前函数退出前"执行，无论从哪条 return 离开。
- 标准套路：确认资源创建成功（`if err != nil` 之后）再 `defer Close`，开与关相邻可见。
- 多个 defer 后进先出，像摞盘子，天然适合"由内向外"拆嵌套资源。
- 大坑：defer 的参数在注册时当场求值（快照）；想读最终值，包一层闭包。
- defer 能修改命名返回值，常用于出门前统一包装 error；写文件等场景注意 Close 自身的错误。

---

## 练习

实现函数：

```go
func readFile(path string) ([]byte, error)
```

要求：

1. 使用 `os.Open` 打开文件。
2. 打开失败时返回带路径的错误。
3. 打开成功后使用 `defer` 关闭文件。
4. 使用 `io.ReadAll` 读取内容。

提示：注意 defer 的位置（第 1 节的坑）；带路径的错误用上一篇的 `fmt.Errorf("... %s: %w", path, err)`。
