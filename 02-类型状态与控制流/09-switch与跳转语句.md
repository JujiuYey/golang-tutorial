# 09: switch 与跳转语句

`switch` 用于多分支判断。Go 的 `switch` 默认不会自动继续执行下一个 `case`。

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

匹配到 `case 3` 后，只执行这一支，不会继续执行 `default`。

---

## 2. 多值 case

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

---

## 3. 无表达式 switch

无表达式 `switch` 常用于替代一串 `if-else`。

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
```

---

## 4. fallthrough

Go 默认不会贯穿到下一个 `case`。如果明确需要继续执行下一个 `case`，可以使用 `fallthrough`。

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

输出：

```text
two
three
```

`fallthrough` 不会重新判断下一个 `case` 条件，直接继续执行下一支。业务代码里要谨慎使用。

---

## 5. break 和 continue

`break` 跳出当前循环，`continue` 跳过本次循环。

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

输出：

```text
0
1
3
```

---

## 6. 标签

嵌套循环里可以用标签跳出外层循环。

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

标签不常用，但读别人代码时要认识。

---

## 7. goto

Go 保留了 `goto`，可以跳到当前函数内的标签。

```go
func main() {
    goto done
    fmt.Println("skip")

done:
    fmt.Println("done")
}
```

普通业务代码里很少需要 `goto`。大多数情况用 `if`、`for`、`return` 会更清楚。

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
