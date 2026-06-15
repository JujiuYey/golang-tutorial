# 06: time：时间、时区、定时器与 ticker

`time` 包处理时间点、时间间隔、格式化、解析、定时器和周期任务。

---

## 1. time.Time 和 time.Duration

```go
now := time.Now()
fmt.Println(now)
```

`time.Time` 表示一个时间点。

```go
d := 2*time.Hour + 30*time.Minute
fmt.Println(d)
```

`time.Duration` 表示时间间隔，底层单位是纳秒。

---

## 2. 格式化时间

Go 使用固定参考时间作为格式模板：

```go
now.Format("2006-01-02 15:04:05")
```

参考时间是：

```text
Mon Jan 2 15:04:05 MST 2006
```

常用格式：

```go
time.RFC3339
"2006-01-02"
"2006-01-02 15:04:05"
```

---

## 3. 解析时间

```go
t, err := time.Parse("2006-01-02", "2026-06-15")
if err != nil {
    return err
}
fmt.Println(t)
```

`time.Parse` 默认按 UTC 解析。

如果要按特定时区解析：

```go
loc, err := time.LoadLocation("Asia/Shanghai")
if err != nil {
    return err
}
t, err := time.ParseInLocation("2006-01-02 15:04:05", "2026-06-15 10:30:00", loc)
```

---

## 4. 时间运算

```go
now := time.Now()

later := now.Add(2 * time.Hour)
diff := later.Sub(now)

fmt.Println(diff.Hours()) // 2
```

比较：

```go
now.Before(later)
later.After(now)
now.Equal(now)
```

---

## 5. 日期运算

```go
nextMonth := now.AddDate(0, 1, 0)
yesterday := now.AddDate(0, 0, -1)
```

日历意义上的年月日变化用 `AddDate`，固定时长变化用 `Add`。

一天不总是等于 24 小时，夏令时地区尤其要注意。涉及日期时优先用日期语义。

---

## 6. 时间戳

```go
sec := now.Unix()
nano := now.UnixNano()
```

从时间戳恢复：

```go
t := time.Unix(sec, 0)
```

和外部系统对接时，要确认时间戳单位是秒、毫秒、微秒还是纳秒。

---

## 7. Timer

```go
timer := time.NewTimer(time.Second)
defer timer.Stop()

<-timer.C
fmt.Println("timeout")
```

简单超时也可以用：

```go
<-time.After(time.Second)
```

高频循环里不要反复创建大量 `time.After`，考虑复用 `Timer`。

---

## 8. Ticker

```go
ticker := time.NewTicker(time.Second)
defer ticker.Stop()

for i := 0; i < 3; i++ {
    <-ticker.C
    fmt.Println("tick")
}
```

`Ticker` 用完要 `Stop`，否则相关资源会继续存在。

---

## 9. Sleep

```go
time.Sleep(100 * time.Millisecond)
```

`Sleep` 适合简单等待和测试模拟。不要用它作为 goroutine 完成同步手段。等待任务完成应该使用 `WaitGroup`、channel 或 context。

---

## 10. 时区

```go
loc, err := time.LoadLocation("Asia/Shanghai")
if err != nil {
    return err
}

now := time.Now().In(loc)
fmt.Println(now.Format("2006-01-02 15:04:05"))
```

服务端系统常见做法是内部用 UTC，展示时转换到用户所在时区。

---

## 练习

1. 打印当前时间的 RFC3339 格式。
2. 解析 `"2026-06-15 10:30:00"` 为上海时区时间。
3. 计算两个时间相差多少分钟。
4. 写一个 ticker，每 200ms 打印一次，打印 5 次后停止。
5. 解释 `Add(24*time.Hour)` 和 `AddDate(0,0,1)` 的区别。
