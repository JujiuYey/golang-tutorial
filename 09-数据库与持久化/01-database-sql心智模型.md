# 01: database/sql 心智模型

这篇解决一个"上来就懵"的问题:在 Node 里装个 `mysql2` 或 `prisma` 就能连库,为什么 Go 要同时 import 两个东西(`database/sql` + 一个驱动),而且驱动前面还有个奇怪的下划线?搞懂这套分层,后面 11 篇的每个 API 你都知道它属于哪一层、在替你管什么。

> 本模块的可运行示例统一用纯 Go 的 SQLite 驱动 `modernc.org/sqlite` 演示——不用装任何数据库,`go run` 直接跑。`database/sql` 这层的用法对 MySQL/PostgreSQL 完全一致,换库只是换驱动和 DSN。

---

## 1. 两层分工:database/sql 管"怎么用",驱动管"怎么连"

先对齐 JS 坐标:Node 的 `mysql2` 是"协议实现 + 使用 API"揉在一个包里;Go 把这两件事拆开了——

| 层 | 负责 | 不负责 |
|----|------|--------|
| `database/sql`(标准库) | 统一 API、连接池、执行 SQL、Scan 结果、事务入口 | 任何具体数据库的网络协议 |
| 驱动(第三方包) | 和 MySQL/PostgreSQL/SQLite 真正对话的协议细节 | — |

两层都不负责的:自动生成业务 SQL、schema 迁移、ORM 模型映射——那是 Prisma/GORM 这类工具的事(第 9 节再说)。

**一句话总结:`database/sql` 是插座标准,驱动是各家的插头。** 你写的代码只面向插座,换数据库基本只换插头。

---

## 2. 下划线导入:驱动怎么"插上去"

驱动导入长这样:

```go
import _ "modernc.org/sqlite"
```

拆开看这行:

```text
import _ "modernc.org/sqlite"
//     ┬  ──────────┬────────
//     │            └ ②包路径照常写
//     └ ①匿名导入:不用包里任何函数,只要它的 init() 跑一下
```

大白话:你不需要直接调用驱动里的任何东西,只需要它在启动时**把自己登记到 `database/sql` 的驱动名册上**(驱动包的 `init()` 里调 `sql.Register`)。验证一下名册:

```go
package main

import (
	"database/sql"
	"fmt"

	_ "modernc.org/sqlite"
)

func main() {
	fmt.Println(sql.Drivers())
}
```

```text
[sqlite]
```

之后 `sql.Open` 的第一个参数就是按这个名字查名册:

```go
db, err := sql.Open("sqlite", "file:test.db")
//                  ───┬────
//                     └ 驱动名,必须和驱动注册的名字一致
```

### 🕳️ 坑:忘了导入驱动

以为会怎样:`sql.Open("mysql", ...)` 直接能用。
实际怎样:

```go
db, err := sql.Open("mysql", "user:pass@tcp(localhost:3306)/app")
fmt.Println(db, err)
```

```text
<nil> sql: unknown driver "mysql" (forgotten import?)
```

为什么:名册上没有叫 `"mysql"` 的驱动——没人注册过。错误信息甚至贴心地提示了 `forgotten import?`。补一行 `import _ "github.com/go-sql-driver/mysql"` 就好。

---

## 3. sql.DB 不是单个连接,是连接池

```go
var db *sql.DB
```

`*sql.DB` 不是"一根连着数据库的线",而是**一个句柄 + 一池子连接**:它按需建立连接、用完收回复用、坏了丢弃重建。

JS 对比:它对应的不是 `mysql2` 的 `createConnection`,而是 `createPool`——只是 Go 里没有"单连接"这个入门选项,标准库直接给你池子。

由此推出用法:**整个程序启动时 `sql.Open` 一次,全局共享这一个 `*sql.DB`,退出时 `Close`。** 不要每个请求开一个——那等于每个 HTTP 请求都新建一个连接池。

---

## 4. 🕳️ 坑:sql.Open 不一定真的连了数据库

以为会怎样:`sql.Open` 没报错 = 数据库连上了。
实际怎样:`Open` 大多只是"验证参数 + 创建句柄",连接是之后用到时才懒加载建立的。看证据——故意打开一个不可能存在的路径:

```go
db, err := sql.Open("sqlite", "file:/no/such/dir/app.db")
fmt.Println("Open 返回的 err:", err)

err = db.Ping()
fmt.Println("Ping 返回的 err:", err)
```

```text
Open 返回的 err: <nil>
Ping 返回的 err: unable to open database file: out of memory (14)
```

`Open` 一声不吭,错误直到 `Ping`(真正建一次连接试试)才暴露。所以标准动作是**启动时 Open 之后立刻 Ping 一次**,确认数据库可达——下一篇细讲。

(顺带一提,这条 `out of memory (14)` 是 SQLite 的经典迷惑错误码,14 = CANTOPEN,实际是"目录不存在打不开文件"。)

---

## 5. 三类常用操作

日常数据库代码就三个动词,按"要不要返回行、返回几行"选:

| 方法 | 用途 | 返回 |
|------|------|------|
| `ExecContext` | INSERT/UPDATE/DELETE/DDL(不要结果行) | `sql.Result` |
| `QueryRowContext` | 查询,最多要一行 | `*sql.Row`,链式 `.Scan` |
| `QueryContext` | 查询多行 | `*sql.Rows`,**必须 Close** |

三个动词一次看全:

```go
db, _ := sql.Open("sqlite", ":memory:")
defer db.Close()

db.Exec(`CREATE TABLE users(id INTEGER PRIMARY KEY, name TEXT)`)

res, _ := db.Exec(`INSERT INTO users(name) VALUES(?)`, "alice")
id, _ := res.LastInsertId()

var name string
db.QueryRow(`SELECT name FROM users WHERE id = ?`, id).Scan(&name)
fmt.Println("id =", id, "name =", name)
```

```text
id = 1 name = alice
```

(演示图省事用 `_` 吞了错误,真实代码每个 err 都要接住——第 03 篇给完整写法。)

JS 对比:`mysql2` 里全是一个 `pool.query()` 打天下,靠返回值形状区分;Go 按用途拆成三个方法,还多了一步显式 `Scan`——把结果**逐列**填进你准备好的变量,没有自动对象映射。

---

## 6. context 很重要

上面示例用的 `Exec`/`QueryRow` 都有带 `Context` 的孪生版本,实际项目**一律用带 Context 的**:

```go
ExecContext / QueryContext / QueryRowContext / BeginTx
```

为什么:数据库操作的每一步都可能卡住——等池子里的空闲连接、等数据库执行、等网络返回。context(06 模块学过的那个)让上游超时或取消时,这些等待有机会及时放弃,而不是傻等到天荒地老。

**一句话总结:能传 context 的数据库方法就传,这是止损绳。**

---

## 7. SQL 和参数分开走

```go
row := db.QueryRowContext(ctx, "SELECT name FROM users WHERE id = ?", id)
//                             ──────────────┬──────────────────  ─┬
//                                    ①SQL 模板,占位符 ?           ②参数单独传
```

用户输入**永远走参数,不拼进 SQL 字符串**。驱动会正确处理转义和类型,SQL 注入直接没门。这和 `mysql2` 的 `pool.query('... WHERE id = ?', [id])` 一个道理,Go 只是把数组摊平成变长参数。

一个跨库差异要有印象:占位符长相不统一——

| 数据库 | 占位符 |
|--------|--------|
| MySQL、SQLite | `WHERE id = ?` |
| PostgreSQL | `WHERE id = $1` |

写代码前先确认自己用的是哪个库、哪个驱动。注入风险和动态 SQL 的完整讨论在第 06 篇。

---

## 8. ORM 和 database/sql 的关系

Prisma/TypeORM 在 Node 生态里的位置,Go 里对应 GORM、sqlc 这类工具:它们**建在驱动和连接池之上**,帮你生成 SQL、映射 struct、管理关联。

建议先把 `database/sql` 的底层语义练熟——连接池、context、Rows 关闭、Scan、事务、参数化 SQL——因为 ORM 出问题时,你排查用的全是这层的知识。这也是本模块的路线:01-10 打地基,ORM 以后随用随学。

---

## 9. 数据库代码检查清单

看到(或写完)一段数据库代码,过一遍:

1. 是否传入 context。
2. 是否使用参数化 SQL。
3. `Rows` 是否关闭。
4. `sql.ErrNoRows` 是否单独处理。
5. 事务是否覆盖所有相关写操作。
6. 连接池是否配置。

后面每篇会把这 6 条逐个展开。

---

## 本篇重点

- [ ] `database/sql` 是统一 API + 连接池,驱动负责具体数据库协议;`import _ "驱动"` 靠 init 注册,忘导入 = `unknown driver`。
- [ ] `*sql.DB` 是连接池不是单连接:程序启动 Open 一次,全局共享,退出 Close。
- [ ] `sql.Open` 不一定真连数据库,启动后要 `PingContext` 验一次。
- [ ] 三个动词:`ExecContext`(不要行)、`QueryRowContext`(一行)、`QueryContext`(多行,必须关)。
- [ ] SQL 模板和参数分开传,用户输入永不拼字符串。

---

## 练习

1. 解释 `database/sql` 和数据库驱动的关系。
2. 解释为什么 `*sql.DB` 不是单个连接。
3. 写出 `ExecContext`、`QueryRowContext`、`QueryContext` 分别适合的场景。
4. 说明 MySQL 和 PostgreSQL 占位符的差异。
5. 找一段数据库代码,检查它有没有传 context、关 rows、处理 error。

提示:练习 1、2 可以用"插座/插头"和"连接池"两个比喻组织答案;练习 5 直接套第 9 节的检查清单。
