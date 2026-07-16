# 06: time:时间、时区、定时器与 ticker

这篇解决时间处理的三件事:表示和运算(`Time` / `Duration`)、格式化和解析(那个著名的 `2006-01-02`)、定时任务(Timer / Ticker)。学完你能躲开 Go 时间的第一大坑:**格式模板不是 `YYYY-MM-DD`,而是一个写死的"参考时刻"**。

对 JS 用户:`time.Time` ≈ `Date`,但 Go 把"时间间隔"单独做成了 `time.Duration` 类型,不像 JS 里全靠毫秒数字裸奔。

---

## 1. 两个核心类型:Time 和 Duration

```go
func main() {
    now := time.Now()
    fmt.Println(now)

    d := 2*time.Hour + 30*time.Minute
    fmt.Println(d)
    fmt.Println(d.Minutes())
}
```

```text
2026-07-16 10:19:48.104965 -0700 PDT m=+0.000057751
2h30m0s
150
```

(第一行是运行时刻和本机时区,你的输出会不同。)

- `time.Time`:一个**时间点**(某年某月某日某时刻)。
- `time.Duration`:一段**时间长度**,底层是纳秒计数的整数。写法很自然:`2*time.Hour + 30*time.Minute`,打印出来自带单位。

对比 JS:`setTimeout(fn, 1500)` 里的 `1500` 是裸数字,单位全靠记;Go 里必须写 `1500*time.Millisecond`,**单位错不了,因为单位就在代码里**。

---

## 2. 格式化:模板是"参考时刻",不是占位符

### 🕳️ 坑:YYYY-MM-DD 在这不管用

以为会怎样:像 moment/dayjs 一样写 `format("YYYY-MM-DD")`。
实际怎样:`YYYY-MM-DD` 被原样输出——它不是占位符,只是普通字符。

```go
func main() {
    t := time.Date(2026, 7, 16, 14, 30, 5, 0, time.UTC)

    fmt.Println(t.Format("2006-01-02 15:04:05"))
    fmt.Println(t.Format(time.RFC3339))
    fmt.Println(t.Format("2006年01月02日"))
    fmt.Println(t.Format("YYYY-MM-DD"))
}
```

```text
2026-07-16 14:30:05
2026-07-16T14:30:05Z
2026年07月16日
YYYY-MM-DD
```

为什么:Go 的格式模板是**拿一个固定的"参考时刻"摆出你想要的样子**,格式化时照葫芦画瓢。参考时刻是:

```text
2006-01-02 15:04:05
── ┬─ ─┬ ─┬ ─┬ ─┬ ─┬
  年   月  日  时  分  秒
// 巧记:01月 02日 03(15点=下午3点) 04分 05秒 06年——1,2,3,4,5,6
```

所以"年"必须写 `2006`、"月"必须写 `01`——写别的数字就不是占位符了(第 3 节会看到写错的下场)。常用的直接用现成常量,比如 `time.RFC3339`(对应 JS 的 `toISOString` 风格)。

---

## 3. 解析:time.Parse

格式化的逆操作,模板规则相同:

```go
func main() {
    t, err := time.Parse("2006-01-02", "2026-06-15")
    fmt.Println(t, err)

    _, err = time.Parse("2026-01-02", "2026-06-15") // 布局写错:用了今年
    fmt.Println(err)
}
```

```text
2026-06-15 00:00:00 +0000 UTC <nil>
parsing time "2026-06-15" as "2026-01-02": cannot parse "-06-15" as "6-"
```

第二个调用就是新手高发事故:顺手把布局里的年份写成今年 `2026`,Go 把它当成"20"+"26 号"之类的字面内容去匹配,报出一个莫名其妙的错。**布局里的年份永远是 2006,和今年是哪年无关。**

另外注意输出里的 `+0000 UTC`:`time.Parse` 默认按 UTC 解析。要按具体时区解析,用 `ParseInLocation`:

```go
func main() {
    u, _ := time.Parse("2006-01-02 15:04:05", "2026-06-15 10:30:00")
    fmt.Println(u)

    loc, err := time.LoadLocation("Asia/Shanghai")
    if err != nil {
        fmt.Println(err)
        return
    }
    t, _ := time.ParseInLocation("2006-01-02 15:04:05", "2026-06-15 10:30:00", loc)
    fmt.Println(t)
}
```

```text
2026-06-15 10:30:00 +0000 UTC
2026-06-15 10:30:00 +0800 CST
```

同一个字符串,两种解析——**差了 8 小时**。解析用户输入的本地时间(订单时间、日程)时,忘了 InLocation 就是线上事故。

---

## 4. 时间运算:Add、Sub 与比较

`Time ± Duration` 用方法,不用运算符:

```go
func main() {
    start := time.Date(2026, 7, 16, 9, 0, 0, 0, time.UTC)
    later := start.Add(2*time.Hour + 30*time.Minute)

    fmt.Println(later)

    diff := later.Sub(start)
    fmt.Println(diff, diff.Hours())

    fmt.Println(start.Before(later), later.After(start))
}
```

```text
2026-07-16 11:30:00 +0000 UTC
2h30m0s 2.5
true true
```

方向记法:`Add` 拿 Time 加 Duration 得新 Time;`Sub` 拿两个 Time 相减得 Duration。比较用 `Before` / `After` / `Equal`——注意判断相等用 `Equal` 而不是 `==`(`Equal` 比较的是时刻本身,不管时区表示,第 10 节有例子)。

---

## 5. 日期运算:AddDate 和它的月末陷阱

按"日历"加减(加一个月、退一天)用 `AddDate(年, 月, 日)`:

```go
func main() {
    t := time.Date(2026, 1, 31, 0, 0, 0, 0, time.UTC)

    fmt.Println(t.AddDate(0, 0, 1)) // 加一天
    fmt.Println(t.AddDate(0, 1, 0)) // 加一个月?
}
```

```text
2026-02-01 00:00:00 +0000 UTC
2026-03-03 00:00:00 +0000 UTC
```

🕳️ 坑:1 月 31 日加一个月,出来是 **3 月 3 日**。因为 AddDate 先机械地算出"2 月 31 日",再按日历规范化:2 月只有 28 天,多出的 3 天顺延进 3 月。月末日期做"每月账单日"这类逻辑时必须特殊处理。

`Add(24*time.Hour)` 和 `AddDate(0,0,1)` 的区别:前者是**物理时长**(精确 24 小时),后者是**日历意义的第二天**。在有夏令时的时区,某天可能只有 23 或 25 小时,两者结果不同。规则:**涉及"哪一天"用 AddDate,涉及"多长时间"用 Add。**

---

## 6. 时间戳:Unix

和外部系统(数据库、其他语言、前端)交换时间,最通用的是 Unix 时间戳:

```go
func main() {
    t := time.Date(2026, 7, 16, 0, 0, 0, 0, time.UTC)

    sec := t.Unix()
    fmt.Println(sec)

    back := time.Unix(sec, 0).UTC()
    fmt.Println(back)
}
```

```text
1784160000
2026-07-16 00:00:00 +0000 UTC
```

🕳️ 坑:JS 的 `Date.now()` 是**毫秒**,Go 的 `Unix()` 是**秒**。前后端对接时差三个数量级,时间直接飞到 1970 年附近或几万年后。对毫秒用 `UnixMilli()`,纳秒用 `UnixNano()`——对接前先对齐单位。

---

## 7. Timer:一次性闹钟

```go
func main() {
    start := time.Now()

    timer := time.NewTimer(500 * time.Millisecond)
    defer timer.Stop()

    <-timer.C
    fmt.Println("timeout,等了", time.Since(start).Round(time.Millisecond))
}
```

```text
timeout,等了 501ms
```

(实际时长有毫秒级浮动,你的输出会略有不同。)

`timer.C` 是一个 channel(06 章的老朋友):时间到了,里面会来一个值,`<-timer.C` 就是在等这声铃响。它天然能进 `select`,06-06 的超时模式用的 `time.After(d)` 就是它的便捷包装。高频循环里别反复 `time.After`(每次都新建 Timer),复用 `NewTimer` + `Reset`。

---

## 8. Ticker:周期性闹钟

对应 JS 的 `setInterval`:

```go
func main() {
    ticker := time.NewTicker(200 * time.Millisecond)
    defer ticker.Stop()

    for i := 1; i <= 3; i++ {
        <-ticker.C
        fmt.Println("tick", i)
    }
}
```

```text
tick 1
tick 2
tick 3
```

(每行间隔约 200ms 打出。)

🕳️ 坑:JS 忘了 `clearInterval` 是内存泄漏,Go 忘了 `ticker.Stop()` 同样泄漏资源——而且 Go 编译器不会提醒你。**NewTicker 和 defer Stop 成对出现**,和"Open 配 Close"一个纪律。

---

## 9. Sleep:让当前 goroutine 睡一会

```go
func main() {
    start := time.Now()
    time.Sleep(100 * time.Millisecond)
    fmt.Println("睡了", time.Since(start).Round(time.Millisecond))
}
```

```text
睡了 100ms
```

适合简单等待和测试模拟。🕳️ 坑:别用 Sleep 等 goroutine 干完活("睡 1 秒应该够它跑完了")——这是 06 章反复强调的反模式,时间估短了出 bug,估长了白等。等任务完成用 `WaitGroup`、channel 或 context。

---

## 10. 时区:同一时刻,两种表盘

```go
func main() {
    t := time.Date(2026, 7, 16, 8, 0, 0, 0, time.UTC)

    loc, err := time.LoadLocation("Asia/Shanghai")
    if err != nil {
        fmt.Println(err)
        return
    }

    fmt.Println(t)
    fmt.Println(t.In(loc))
    fmt.Println(t.Equal(t.In(loc)))
}
```

```text
2026-07-16 08:00:00 +0000 UTC
2026-07-16 16:00:00 +0800 CST
true
```

最后一行是关键:UTC 的 8 点和上海的 16 点,`Equal` 判定**是同一个时刻**——`In(loc)` 不改变时间点,只是换一块表盘来读它。

服务端的通行做法:**内部存储和计算一律 UTC,只在展示给用户时 `In(用户时区)` 转换**——和"数据库存 UTC,前端 toLocaleString"是同一个思路。

---

## 本篇重点

- [ ] `Time` 是时间点,`Duration` 是时间长度且自带单位(`2*time.Hour`),不像 JS 用裸毫秒数。
- [ ] 格式模板是参考时刻 `2006-01-02 15:04:05`(巧记 1~6),写 `YYYY-MM-DD` 会被原样输出,布局里写今年年份会解析失败。
- [ ] `Parse` 默认 UTC,带时区解析用 `ParseInLocation`——差 8 小时就是事故;判断相等用 `Equal` 不用 `==`。
- [ ] "多长时间"用 `Add`/`Sub`,"哪一天"用 `AddDate`;注意 1 月 31 日 + 1 月 = 3 月 3 日的规范化陷阱;`Unix()` 是秒,JS 的 `Date.now()` 是毫秒。
- [ ] Timer 是一次性、Ticker 是周期性,都通过 channel 送信,都要 `Stop`;别拿 Sleep 做 goroutine 同步。

---

## 练习

1. 打印当前时间的 RFC3339 格式(提示:现成常量,不用自己写布局)。
2. 解析 `"2026-06-15 10:30:00"` 为上海时区时间,打印出来确认带 `+0800`(提示:第 3 节,注意是哪个函数)。
3. 计算两个时间相差多少分钟(提示:`Sub` 得到 Duration,它身上有现成的方法)。
4. 写一个 ticker,每 200ms 打印一次,打印 5 次后停止(提示:别忘了那个 defer)。
5. 用自己的话解释 `Add(24*time.Hour)` 和 `AddDate(0,0,1)` 的区别,并说出一个两者结果不同的场景(提示:第 5 节最后一段)。
