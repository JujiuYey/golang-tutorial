# 09: switch 与跳转语句

这篇解决的问题：JS 的 switch 忘写 `break` 就"漏电"贯穿到下一个 case——这个陈年老坑在 Go 里**默认不存在**。学完你能用 Go 的 switch 干掉一串 else if，还能认识 break/continue/标签/goto 这些跳转语句。

---

## 1. 基本 switch

```go
func main() {
    day := 3

    switch day {
    case 1:
        fmt.Println("Mon")
    case 2:
        fmt.Println("Tue")
    case 3:
        fmt.Println("Wed")
    default:
        fmt.Println("Unknown")
    }
}
```

```text
Wed
```

匹配到 `case 3` 后只执行这一支就结束，**不需要写 `break`**。再验证一次——JS 直觉里下面会把 "one" 和 "two" 都打出来，Go 只打一个：

```go
func main() {
    switch 1 {
    case 1:
        fmt.Println("one")
    case 2:
        fmt.Println("two")
    }
}
```

```text
one
```

**一句话总结：Go 的 switch 每个 case 自带隐形 break——JS 里最容易忘的那行，Go 帮你写死了。**

---

## 2. 多值 case

一个 case 想匹配多个值，逗号排开。JS 里你得叠好几个空 case 利用贯穿来实现，Go 直接支持：

```go
func main() {
    day := 6

    switch day {
    case 1, 2, 3, 4, 5:
        fmt.Println("weekday")
    case 6, 7:
        fmt.Println("weekend")
    default:
        fmt.Println("invalid")
    }
}
```

```text
weekend
```

---

## 3. 无表达式 switch

switch 后面可以什么都不写，每个 case 直接放布尔条件——等于一串 `if / else if` 的整齐版：

```go
func grade(score int) string {
    switch {
    case score >= 90:
        return "A"
    case score >= 80:
        return "B"
    case score >= 60:
        return "C"
    default:
        return "D"
    }
}

func main() {
    fmt.Println(grade(95), grade(83), grade(61), grade(30))
}
```

```text
A B C D
```

拆开看它在干什么：

```text
switch {            // 没写表达式，默认当作 switch true
case score >= 90:   // 从上往下找第一个为 true 的 case
```

从上往下逐个判断，命中即止——所以 case 的**顺序**就是逻辑（83 分先被 `>= 80` 截住，轮不到 `>= 60`）。这是 02-07 grade 函数那串 else if 的地道替代写法，JS 没有对应物。

---

## 4. fallthrough

真需要 JS 那种"贯穿到下一支"的行为？Go 让你**显式**写出来——关键字 `fallthrough`：

```go
func main() {
    switch 2 {
    case 1:
        fmt.Println("one")
        fallthrough
    case 2:
        fmt.Println("two")
        fallthrough
    case 3:
        fmt.Println("three")
    }
}
```

```text
two
three
```

注意两点：

- 从匹配处（case 2）才开始执行，case 1 没份——fallthrough 不会"向上"生效。
- **fallthrough 不判断下一个 case 的条件**，直接闯进去执行（所以 case 3 明明不等于 2 也被执行了）。

对比记忆：**JS 默认贯穿、写 `break` 阻断；Go 默认阻断、写 `fallthrough` 贯穿**——正好互为镜像。业务代码里 fallthrough 很少见，用之前想想能不能用多值 case 代替。

---

## 5. break 和 continue

和 JS 完全一致：`break` 整个跳出循环，`continue` 跳过本轮进下一轮：

```go
func main() {
    for i := 0; i < 5; i++ {
        if i == 2 {
            continue
        }
        if i == 4 {
            break
        }
        fmt.Println(i)
    }
}
```

```text
0
1
3
```

`2` 被 continue 跳过，到 `4` 时 break 收工。

---

## 6. 标签

嵌套循环里，裸 `break` 只能跳出最内层。想直接跳出外层，给外层循环贴个标签（JS 也有同款语法，只是你可能没用过）：

```go
func main() {
outer:
    for i := 0; i < 3; i++ {
        for j := 0; j < 3; j++ {
            if i+j > 2 {
                break outer
            }
            fmt.Println(i, j)
        }
    }
}
```

```text
0 0
0 1
0 2
1 0
1 1
```

`i=1, j=2` 时 `i+j > 2` 成立，`break outer` 把内外两层一起结束——如果只写 `break`，外层还会继续跑 `i=2`。标签不常用，但读别人代码时要认识。

---

## 7. goto

Go 保留了 `goto`：无条件跳到当前函数内的某个标签处。

```go
func main() {
    goto done
    fmt.Println("skip")

done:
    fmt.Println("done")
}
```

```text
done
```

`"skip"` 那行被直接跳过了。普通业务代码几乎不需要 goto——`if`、`for`、`return` 能表达得更清楚。认识它就够了，别主动用。

---

## 本篇重点

- [ ] Go 的 switch 默认不贯穿，不用写 break；和 JS 正好相反（JS 默认贯穿，靠 break 阻断）。
- [ ] 一个 case 匹配多个值用逗号：`case 6, 7:`。
- [ ] 无表达式 `switch { case 条件: }` 是一串 else if 的整齐替代，从上往下命中即止。
- [ ] `fallthrough` 显式贯穿，且不判断下一个 case 的条件——慎用。
- [ ] `break 标签` 可以一次跳出多层嵌套循环；goto 认识即可，不要主动用。

---

## 练习

实现一个 `grade(score int) string` 函数。

规则：

| 分数范围 | 返回 |
|----------|------|
| `90-100` | `A` |
| `80-89` | `B` |
| `60-79` | `C` |
| `0-59` | `D` |
| 其他 | `invalid` |

要求：

1. 使用 `switch` 实现。
2. 先处理非法分数。
3. 至少手动测试 `100`、`90`、`89`、`60`、`59`、`-1`、`101`。

提示：用无表达式 switch；"先处理非法分数"意味着 `score < 0 || score > 100` 应该是第一个 case——想想为什么顺序放错结果就不对。
