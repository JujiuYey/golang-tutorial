# 02: 连接数据库、Ping 与连接池配置

这篇解决"服务一启动就该把数据库这条命脉接稳"的问题:怎么正确地打开、验证、关闭一个 `*sql.DB`,以及连接池的四个旋钮各管什么。学完你能写出一个项目里可以直接复用的 `OpenDB` 构造函数,并且亲眼看到"连接池被占满时请求在排队"长什么样。

> 本篇可运行示例用 SQLite(`modernc.org/sqlite`)演示,连接池行为由 `database/sql` 这层统一管理,对 MySQL/PostgreSQL 语义一致。

---

## 1. 打开数据库:驱动名 + DSN

`sql.Open(驱动名, DSN)`,DSN(Data Source Name)就是"数据库在哪、用什么身份连"的一串描述——对应 Node 里传给 `mysql2.createPool` 的那坨配置对象,只是 Go 惯用一个字符串。

SQLite(本模块演示用):

```go
db, err := sql.Open("sqlite", "file:app.db")
if err != nil {
    return nil, fmt.Errorf("open db: %w", err)
}
```

MySQL:

```go
dsn := "user:password@tcp(localhost:3306)/app?parseTime=true"
db, err := sql.Open("mysql", dsn)
```

PostgreSQL:

```go
dsn := "postgres://user:password@localhost:5432/app?sslmode=disable"
db, err := sql.Open("postgres", dsn)
```

三种 DSN 长相各异(格式由**驱动**规定,不是 `database/sql`),但后续用法完全相同。DSN 里有密码,**不要硬编码进源码**,第 7 节从环境变量读。

---

## 2. PingContext:确认真的连得上

上一篇的坑还记得吧——`sql.Open` 不报错不代表连上了。所以启动时紧跟一次 `PingContext`:

```go
ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
defer cancel()

if err := db.PingContext(ctx); err != nil {
    db.Close()
    fmt.Println("ping db:", err)
    return
}
fmt.Println("数据库可达")
```

指向一个不存在的路径跑一遍,错误在 Ping 这步现形:

```text
ping db: unable to open database file: out of memory (14)
```

两个细节:① 带 3 秒超时——数据库挂了时你想要的是"3 秒内报错退出",不是无限等;② Ping 失败要 `db.Close()`,把已经创建的句柄收掉。

**一句话总结:Open 只是填表,Ping 才是真握手。**

---

## 3. 关闭数据库

```go
defer db.Close()
```

`Close` 关的是**整个连接池**,只该在程序退出时执行一次。不要在每个请求结束时关全局 db——那相当于每个请求都把公司的总闸拉了。

---

## 4. 连接池的四个旋钮

```go
db.SetMaxOpenConns(25)                  // 最多同时开多少连接
db.SetMaxIdleConns(25)                  // 最多留多少空闲连接备用
db.SetConnMaxLifetime(5 * time.Minute)  // 一个连接最多复用多久就换新
db.SetConnMaxIdleTime(1 * time.Minute)  // 空闲连接最多闲置多久就回收
```

JS 对比:`mysql2` 的 `createPool({ connectionLimit: 10 })` 只有一个上限旋钮;Go 把"上限、备胎数、寿命、闲置期"四个维度都交给你调。

| 旋钮 | 管什么 | 调错的代价 |
|------|--------|-----------|
| `MaxOpenConns` | 并发上限 | 太大压垮数据库,太小请求排队 |
| `MaxIdleConns` | 空闲备胎 | 太小频繁建连,浪费握手成本 |
| `ConnMaxLifetime` | 连接寿命 | 不设可能撞上服务端/中间件掐掉旧连接 |
| `ConnMaxIdleTime` | 闲置回收 | 让低峰期资源自动缩回去 |

---

## 5. 亲眼看一次"池子满了"

`MaxOpenConns` 设成 2,用两个事务把连接占死,第 3 个操作只能排队——排到超时:

```go
db.SetMaxOpenConns(2)

tx1, _ := db.Begin() // 占住第 1 个连接
tx2, _ := db.Begin() // 占住第 2 个连接
defer tx1.Rollback()
defer tx2.Rollback()

ctx, cancel := context.WithTimeout(context.Background(), 500*time.Millisecond)
defer cancel()

var n int
err := db.QueryRowContext(ctx, "SELECT 1").Scan(&n)
fmt.Println("第 3 个操作:", err)

s := db.Stats()
fmt.Printf("InUse=%d WaitCount=%d WaitDuration=%v\n",
    s.InUse, s.WaitCount, s.WaitDuration.Round(time.Millisecond))
```

```text
第 3 个操作: context deadline exceeded
InUse=2 WaitCount=1 WaitDuration=501ms
```

这个小实验说清了两件事:

- 连接不够时请求**静默排队**,不报"连接池满"——你只会看到超时。这就是第 1 篇说"操作要带 context"的原因之一。
- 事务会一直**独占**一个连接直到 Commit/Rollback,长事务是吃光池子的头号嫌犯。

所以设池子大小时要考虑:数据库允许的最大连接数、服务实例数量(每个实例一个池!)、每个请求持有连接多久、慢查询和长事务多不多。**连接池不是越大越好,是"够用且不压垮数据库"。**

---

## 6. db.Stats:池子的仪表盘

上面已经用过了,`db.Stats()` 返回池子的实时状态:

```go
s := db.Stats()
s.OpenConnections // 当前打开的连接数
s.InUse           // 正在被使用的
s.Idle            // 空闲待命的
s.WaitCount       // 累计有多少次操作等过连接
```

`WaitCount` 持续上涨 = 池子偏小或有连接被长期占用。生产环境通常把这些指标接进监控,别等用户喊慢才看。

---

## 7. DSN 从环境变量来

```go
dsn := os.Getenv("DATABASE_URL")
if dsn == "" {
    return "", errors.New("DATABASE_URL is required")
}
```

没配环境变量直接跑:

```text
启动失败: DATABASE_URL is required
```

配上再跑(`DATABASE_URL=postgres://... go run .`):

```text
dsn = postgres://user:***@localhost:5432/app
```

密码、地址、库名进环境变量或配置文件,不进代码仓库——和 Node 的 `.env` + `process.env.DATABASE_URL` 同一套纪律。缺了就**启动即失败**,别让服务带病跑到第一个请求才炸。

---

## 8. 收拢成一个构造函数

把配置集中成 struct,打开、调池、验活一条龙:

```go
type DBConfig struct {
    Driver          string
    DSN             string
    MaxOpenConns    int
    MaxIdleConns    int
    ConnMaxLifetime time.Duration
}

func OpenDB(ctx context.Context, cfg DBConfig) (*sql.DB, error) {
    db, err := sql.Open(cfg.Driver, cfg.DSN)
    if err != nil {
        return nil, fmt.Errorf("open db: %w", err)
    }

    db.SetMaxOpenConns(cfg.MaxOpenConns)
    db.SetMaxIdleConns(cfg.MaxIdleConns)
    db.SetConnMaxLifetime(cfg.ConnMaxLifetime)

    if err := db.PingContext(ctx); err != nil {
        db.Close()
        return nil, fmt.Errorf("ping db: %w", err)
    }
    return db, nil
}
```

用 SQLite 跑一遍:

```go
db, err := OpenDB(ctx, DBConfig{
    Driver: "sqlite", DSN: ":memory:",
    MaxOpenConns: 25, MaxIdleConns: 25, ConnMaxLifetime: 5 * time.Minute,
})
```

```text
连接就绪, MaxOpen=25
```

🕳️ 坑:Ping 失败的分支里那句 `db.Close()` 别省——`sql.Open` 成功后句柄已经存在,直接 return 错误会泄漏它。**先创建成功的资源,失败路径上也要负责收尾**(06 模块 defer 篇的老规矩)。

---

## 9. 启动检查清单

服务启动时至少确认:

1. DSN 存在(环境变量缺了就快速失败)。
2. 驱动名正确(且驱动已 `import _`)。
3. `PingContext` 成功,带超时。
4. 连接池参数按负载设置过,不是默认值裸奔。
5. 退出路径上有 `db.Close()`。

---

## 本篇重点

- [ ] 标准开场三连:`sql.Open` → 设连接池参数 → `PingContext`(带超时,失败要 `db.Close()`)。
- [ ] `db.Close()` 关的是整个池子,只在程序退出时调一次。
- [ ] 四个旋钮:MaxOpenConns(上限)、MaxIdleConns(备胎)、ConnMaxLifetime(寿命)、ConnMaxIdleTime(闲置回收);不是越大越好。
- [ ] 池子满时请求静默排队直到超时,长事务是独占连接的大户;用 `db.Stats()` 的 WaitCount 观察。
- [ ] DSN 从环境变量读,缺了启动即失败。

---

## 练习

1. 写 `OpenDB(ctx, cfg)` 函数。
2. 加上 `PingContext` 和 3 秒超时。
3. 配置连接池参数。
4. 打印 `db.Stats()`。
5. 从 `DATABASE_URL` 读取 DSN。

提示:五个练习其实是同一个程序的五步,照第 8 节的骨架搭,再把第 7 节的环境变量接进 `DBConfig.DSN`;想验证 Ping 失败分支,把 DSN 指向一个不存在的目录即可。
