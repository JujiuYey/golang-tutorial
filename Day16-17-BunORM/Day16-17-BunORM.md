# Day 16-17: 数据库Bun ORM

> 预计学习时长: 6小时  
> 目标: 掌握Bun ORM的使用，进行MySQL/PostgreSQL数据库操作

---

## 目录

1. [Bun ORM简介](#1-bun-orm简介)
2. [安装与配置](#2-安装与配置)
3. [CRUD操作](#3-crud操作)
4. [查询构建器](#4-查询构建器)
5. [事务](#5-事务)
6. [练习作业](#6-练习作业)

---

## 1. Bun ORM简介

Bun是一个高性能的Go ORM库，支持MySQL、PostgreSQL、SQLite。

**特点：**
- 性能高（比GORM快10倍）
- API简洁
- 自动迁移
- 支持事务
- 支持关联查询

---

## 2. 安装与配置

### 2.1 安装

```bash
go get github.com/uptrace/bun
go get github.com/uptrace/bun/extra/bundebug
```

### 2.2 驱动安装

```bash
# MySQL
go get github.com/go-sql-driver/mysql

# PostgreSQL
go get github.com/lib/pq

# SQLite
go get modernc.org/sqlite
```

### 2.3 连接数据库

```go
import (
    "github.com/uptrace/bun"
    _ "github.com/go-sql-driver/mysql"
)

func main() {
    db, err := bun.OpenDBConnection("mysql://root:password@localhost/mydb?parseTime=true")
    if err != nil {
        panic(err)
    }
    defer db.Close()
}
```

### 2.4 SQLite示例

```go
import (
    "github.com/uptrace/bun"
    modernsqlite "modernc.org/sqlite"
)

func main() {
    db, err := bun.OpenDBConnection("sqlite:/tmp/mydb.sqlite")
    if err != nil {
        panic(err)
    }
    defer db.Close()
}
```

---

## 3. CRUD操作

### 3.1 定义Model

```go
import "github.com/uptrace/bun"

type User struct {
    bun.BaseModel `bun:"table:users"`  // 表名
    
    ID        int64     `bun:"id,pk,autoincrement"`
    Name      string    `bun:"name,notnull"`
    Email     string    `bun:"email,unique"`
    Age       int       `bun:"age"`
    CreatedAt time.Time `bun:"created_at"`
    UpdatedAt time.Time `bun:"updated_at"`
}
```

### 3.2 创建表（自动迁移）

```go
func main() {
    db, _ := bun.OpenDBConnection("sqlite:/tmp/mydb.sqlite")
    
    // 自动创建表
    _, err := db.NewCreateTableModel().IfNotExists().Table(reflect.TypeOf(User{})).Exec(ctx)
    if err != nil {
        panic(err)
    }
}
```

### 3.3 插入数据

```go
func main() {
    db, _ := bun.OpenDBConnection("sqlite:/tmp/mydb.sqlite")
    ctx := context.Background()
    
    // 插入单条
    user := User{Name: "张三", Email: "zhangsan@example.com", Age: 25}
    _, err := db.NewInsert().Model(&user).Exec(ctx)
    
    // 插入多条
    users := []User{
        {Name: "李四", Email: "lisi@example.com", Age: 30},
        {Name: "王五", Email: "wangwu@example.com", Age: 28},
    }
    _, err = db.NewInsert().Model(&users).Exec(ctx)
}
```

### 3.4 查询数据

```go
func main() {
    db, _ := bun.OpenDBConnection("sqlite:/tmp/mydb.sqlite")
    ctx := context.Background()
    
    // 查询单条
    var user User
    err := db.NewSelectModel().Model(&user).Where("id = ?", 1).Scan(ctx)
    
    // 查询多条
    var users []User
    err = db.NewSelectModel().Model(&users).Where("age > ?", 20).Scan(ctx)
    
    // 查询所有
    err = db.NewSelectModel().Model(&users).Scan(ctx)
    
    // 限制数量
    err = db.NewSelectModel().Model(&users).Limit(10).Offset(5).Scan(ctx)
}
```

### 3.5 更新数据

```go
func main() {
    db, _ := bun.OpenDBConnection("sqlite:/tmp/mydb.sqlite")
    ctx := context.Background()
    
    // 更新单条
    user := User{ID: 1, Name: "张三", Email: "new@example.com"}
    _, err := db.NewUpdate().Model(&user).WherePK().Exec(ctx)
    
    // 按条件更新
    _, err = db.NewUpdate().Model((*User)(nil)).
        Set("age = age + 1").
        Where("age > ?", 18).
        Exec(ctx)
}
```

### 3.6 删除数据

```go
func main() {
    db, _ := bun.OpenDBConnection("sqlite:/tmp/mydb.sqlite")
    ctx := context.Background()
    
    // 删除单条
    user := User{ID: 1}
    _, err := db.NewDelete().Model(&user).WherePK().Exec(ctx)
    
    // 按条件删除
    _, err = db.NewDelete().Model((*User)(nil)).
        Where("age < ?", 18).
        Exec(ctx)
}
```

---

## 4. 查询构建器

### 4.1 条件查询

```go
// AND条件
err = db.NewSelectModel().Model(&users).
    Where("age > ? AND age < ?", 18, 30).
    WhereOr("name = ?", "张三").
    Scan(ctx)

// IN查询
err = db.NewSelectModel().Model(&users).
    Where("id IN (?)", bun.In([]int{1, 2, 3})).
    Scan(ctx)

// LIKE查询
err = db.NewSelectModel().Model(&users).
    Where("name LIKE ?", "%张%").
    Scan(ctx)
```

### 4.2 排序与分组

```go
// 排序
err = db.NewSelectModel().Model(&users).
    Order("age DESC").
    Order("name ASC").
    Scan(ctx)

// 分页
err = db.NewSelectModel().Model(&users).
    Order("id ASC").
    Limit(10).
    Offset(20).
    Scan(ctx)

// 计数
count, _ := db.NewSelectModel().Model((*User)(nil)).Count(ctx)
```

### 4.3 聚合查询

```go
import "github.com/uptrace/bun"

type Stats struct {
    Count   int `bun:"count"`
    AvgAge  float64 `bun:"avg_age"`
    MaxAge  int `bun:"max_age"`
    MinAge  int `bun:"min_age"`
}

var stats Stats
err := db.NewSelectModel().Model((*User)(nil)).
    Column("count(*) as count").
    Column("avg(age) as avg_age").
    Column("max(age) as max_age").
    Column("min(age) as min_age").
    Scan(ctx)
```

### 4.4 关联查询

```go
type Order struct {
    ID     int64 `bun:"id,pk"`
    UserID int64 `bun:"user_id"`
    Amount float64 `bun:"amount"`
}

type UserWithOrders struct {
    User
    Orders []Order `bun:"rel:has-many,orders"`
}

var results []UserWithOrders
err := db.NewSelectModel().Model(&results).
    Relation("Orders").
    Scan(ctx)
```

---

## 5. 事务

### 5.1 基本事务

```go
func main() {
    db, _ := bun.OpenDBConnection("sqlite:/tmp/mydb.sqlite")
    ctx := context.Background()
    
    tx, err := db.BeginTx(ctx, nil)
    if err != nil {
        panic(err)
    }
    
    // 执行操作
    user := User{Name: "新用户", Email: "new@example.com"}
    _, err = tx.NewInsert().Model(&user).Exec(ctx)
    if err != nil {
        tx.Rollback()
        return
    }
    
    _, err = tx.NewInsert().Model(&User{ID: user.ID, Age: 20}).Exec(ctx)
    if err != nil {
        tx.Rollback()
        return
    }
    
    tx.Commit()
}
```

### 5.2 使用RunInTx

```go
err := db.RunInTx(ctx, nil, func(tx *bun.Tx) error {
    user := User{Name: "事务用户"}
    _, err := tx.NewInsert().Model(&user).Exec(ctx)
    if err != nil {
        return err
    }
    
    order := Order{UserID: user.ID, Amount: 100}
    _, err = tx.NewInsert().Model(&order).Exec(ctx)
    return err
})
```

---

## 6. 练习作业

### 练习1: 用户管理系统

```go
package main

import (
    "context"
    "fmt"
    "time"
    
    "github.com/uptrace/bun"
    _ "modernc.org/sqlite"
)

type User struct {
    ID        int64     `bun:"id,pk,autoincrement"`
    Name      string    `bun:"name,notnull"`
    Email     string    `bun:"email,unique"`
    Age       int       `bun:"age"`
    CreatedAt time.Time `bun:"created_at"`
}

func main() {
    db, err := bun.OpenDBConnection("sqlite:/tmp/users.sqlite")
    if err != nil {
        panic(err)
    }
    defer db.Close()
    
    ctx := context.Background()
    
    // 创建表
    db.NewCreateTableModel().IfNotExists().Model(&User{}).Exec(ctx)
    
    // 插入用户
    user := User{Name: "张三", Email: "zhangsan@example.com", Age: 25}
    db.NewInsert().Model(&user).Exec(ctx)
    fmt.Printf("插入用户: ID=%d, Name=%s\n", user.ID, user.Name)
    
    // 查询用户
    var found User
    db.NewSelectModel().Model(&found).Where("id = ?", user.ID).Scan(ctx)
    fmt.Printf("查询用户: %+v\n", found)
    
    // 更新用户
    db.NewUpdate().Model(&found).Set("age = ?", 26).WherePK().Exec(ctx)
    fmt.Println("更新用户年龄为26")
    
    // 删除用户
    db.NewDelete().Model(&found).WherePK().Exec(ctx)
    fmt.Println("删除用户")
}
```

### 练习2: 分页查询

```go
func getUsersByPage(db *bun.DB, page, pageSize int) ([]User, int64) {
    ctx := context.Background()
    
    offset := (page - 1) * pageSize
    
    var users []User
    db.NewSelectModel().Model(&users).
        Order("id ASC").
        Limit(pageSize).
        Offset(offset).
        Scan(ctx)
    
    total, _ := db.NewSelectModel().Model((*User)(nil)).Count(ctx)
    
    return users, total
}
```

### 挑战题: 博客系统数据库

```go
type Post struct {
    ID        int64     `bun:"id,pk,autoincrement"`
    Title     string    `bun:"title,notnull"`
    Content   string    `bun:"content"`
    AuthorID  int64     `bun:"author_id"`
    CreatedAt time.Time `bun:"created_at"`
    UpdatedAt time.Time `bun:"updated_at"`
}

type Author struct {
    ID    int64  `bun:"id,pk,autoincrement"`
    Name  string `bun:"name,notnull"`
    Email string `bun:"email,unique"`
}

type PostWithAuthor struct {
    Post
    Author Author `bun:"rel:belongs-to,Author"`
}

func main() {
    db, _ := bun.OpenDBConnection("sqlite:/tmp/blog.sqlite")
    ctx := context.Background()
    
    // 创建表
    db.NewCreateTableModel().IfNotExists().Model(&Author{}).Exec(ctx)
    db.NewCreateTableModel().IfNotExists().Model(&Post{}).Exec(ctx)
    
    // 插入作者
    author := Author{Name: "张三", Email: "zhangsan@example.com"}
    db.NewInsert().Model(&author).Exec(ctx)
    
    // 插入文章
    post := Post{
        Title:    "Go语言入门",
        Content:  "这是一篇关于Go语言的教程...",
        AuthorID: author.ID,
    }
    db.NewInsert().Model(&post).Exec(ctx)
    
    // 查询文章及作者
    var results []PostWithAuthor
    db.NewSelectModel().Model(&results).
        Relation("Author").
        Order("created_at DESC").
        Scan(ctx)
    
    for _, r := range results {
        fmt.Printf("文章: %s, 作者: %s\n", r.Title, r.Author.Name)
    }
}
```

---

## 下一步

- **Day 18-19**: Fiber Web框架

---

*祝练习愉快！*
