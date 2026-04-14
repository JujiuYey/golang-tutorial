# Day 18-19: Fiber Web框架

> 预计学习时长: 6小时  
> 目标: 掌握Fiber框架开发高性能Web应用

---

## 目录

1. [Fiber简介](#1-fiber简介)
2. [安装与入门](#2-安装与入门)
3. [路由](#3-路由)
4. [中间件](#4-中间件)
5. [请求与响应](#5-请求与响应)
6. [模板与静态文件](#6-模板与静态文件)
7. [练习作业](#7-练习作业)

---

## 1. Fiber简介

Fiber是一个用Go编写的Express-like Web框架，以高性能著称。

**特点：**
- 性能极高（比Gin快2倍）
- API简洁，类似Express.js
- 内置中间件支持
- 路由分组
- WebSocket支持

---

## 2. 安装与入门

### 2.1 安装

```bash
go get github.com/gofiber/fiber/v2
```

### 2.2 第一个应用

```go
package main

import "github.com/gofiber/fiber/v2"

func main() {
    app := fiber.New()
    
    // 路由
    app.Get("/", func(c *fiber.Ctx) error {
        return c.SendString("Hello, World!")
    })
    
    // 启动服务器
    app.Listen(":3000")
}
```

### 2.3 运行

```bash
go run main.go
# 访问 http://localhost:3000
```

---

## 3. 路由

### 3.1 基本路由

```go
app := fiber.New()

// GET
app.Get("/users", handler)
app.Post("/users", handler)
app.Put("/users/:id", handler)
app.Delete("/users/:id", handler)

// 所有方法
app.All("/about", handler)

// 路由分组
api := app.Group("/api")
api.Get("/users", handler)
api.Post("/users", handler)
```

### 3.2 路径参数

```go
// 参数
app.Get("/user/:id", func(c *fiber.Ctx) error {
    id := c.Params("id")
    return c.SendString("User ID: " + id)
})

// 可选参数
app.Get("/user/:id?", func(c *fiber.Ctx) error {
    id := c.Params("id")
    if id == "" {
        return c.SendString("All users")
    }
    return c.SendString("User: " + id)
})

// 通配符
app.Get("/files/*", func(c *fiber.Ctx) error {
    path := c.Params("*")
    return c.SendString("Path: " + path)
})
```

### 3.3 路由分组

```go
api := app.Group("/api/v1")

users := api.Group("/users")
users.Get("/", listUsers)
users.Get("/:id", getUser)
users.Post("/", createUser)
users.Put("/:id", updateUser)
users.Delete("/:id", deleteUser)
```

---

## 4. 中间件

### 4.1 内置中间件

```go
import "github.com/gofiber/fiber/v2/middleware/logger"
import "github.com/gofiber/fiber/v2/middleware/recover"

func main() {
    app := fiber.New()
    
    // 日志中间件
    app.Use(logger.New())
    
    // 错误恢复中间件
    app.Use(recover.New())
    
    app.Get("/", handler)
    
    app.Listen(":3000")
}
```

### 4.2 自定义中间件

```go
// 简单中间件
app.Use(func(c *fiber.Ctx) error {
    fmt.Println("Before handler")
    return c.Next()  // 继续执行
})

// 带配置的中间件
app.Use("/api", func(c *fiber.Ctx) error {
    // 只对 /api 路径生效
    return c.Next()
})
```

### 4.3 中间件链

```go
app.Use(
    logger.New(),
    recover.New(),
    cors.New(),
)

// 或者链式调用
app.Use(logger.New()).Use(recover.New())
```

### 4.4 常用中间件

```go
import (
    "github.com/gofiber/fiber/v2/middleware/cors"
    "github.com/gofiber/fiber/v2/middleware/compress"
    "github.com/gofiber/fiber/v2/middleware/adaptor"
)
```

---

## 5. 请求与响应

### 5.1 获取请求数据

```go
// Query参数
app.Get("/search", func(c *fiber.Ctx) error {
    q := c.Query("q")
    page := c.QueryInt("page", 1)
    return c.SendString("Search: " + q)
})

// 路径参数
app.Get("/user/:id", func(c *fiber.Ctx) error {
    id := c.Params("id")
    return c.SendString("User: " + id)
})

// 表单数据
app.Post("/form", func(c *fiber.Ctx) error {
    name := c.FormValue("name")
    email := c.FormValue("email")
    return c.SendString("Name: " + name)
})

// JSON Body
app.Post("/json", func(c *fiber.Ctx) error {
    var body map[string]interface{}
    if err := c.BodyParser(&body); err != nil {
        return err
    }
    return c.JSON(body)
})
```

### 5.2 响应数据

```go
// 字符串
c.SendString("Hello")

// JSON
c.JSON(fiber.Map{
    "message": "success",
    "data":    user,
})

// JSONP
c.JSONP(fiber.Map{"msg": "hello"}, "callback")

// 文件
c.SendFile("path/to/file.txt")

// 状态码
c.SendStatus(404)
```

### 5.3 JSON Struct绑定

```go
type User struct {
    Name  string `json:"name"`
    Email string `json:"email"`
    Age   int    `json:"age"`
}

app.Post("/user", func(c *fiber.Ctx) error {
    var user User
    if err := c.BodyParser(&user); err != nil {
        return c.Status(400).JSON(fiber.Map{
            "error": "Invalid JSON",
        })
    }
    
    return c.JSON(user)
})
```

---

## 6. 模板与静态文件

### 6.1 HTML模板

```go
import (
    "github.com/gofiber/fiber/v2"
    "github.com/gofiber/template/html/v2"
)

func main() {
    // 创建模板引擎
    engine := html.New("./views", ".html")
    
    app := fiber.New(fiber.Config{
        Template: engine,
    })
    
    app.Get("/", func(c *fiber.Ctx) error {
        return c.Render("index", fiber.Map{
            "title": "Home Page",
            "users": []string{"张三", "李四"},
        })
    })
    
    app.Listen(":3000")
}
```

### 6.2 模板语法

```html
<!-- views/index.html -->
<!DOCTYPE html>
<html>
<head>
    <title>{{.title}}</title>
</head>
<body>
    <h1>{{.title}}</h1>
    <ul>
        {{range .users}}
        <li>{{.}}</li>
        {{end}}
    </ul>
</body>
</html>
```

### 6.3 静态文件

```go
// 静态文件目录
app.Static("/", "./public")

// 挂载到特定路径
app.Static("/static", "./assets")
```

---

## 7. 练习作业

### 练习1: RESTful API

```go
package main

import (
    "github.com/gofiber/fiber/v2"
)

type User struct {
    ID   int    `json:"id"`
    Name string `json:"name"`
    Age  int    `json:"age"`
}

var users = []User{
    {1, "张三", 25},
    {2, "李四", 30},
}

func main() {
    app := fiber.New()
    
    // 获取所有用户
    app.Get("/api/users", func(c *fiber.Ctx) error {
        return c.JSON(users)
    })
    
    // 获取单个用户
    app.Get("/api/users/:id", func(c *fiber.Ctx) error {
        id := c.ParamsInt("id")
        for _, u := range users {
            if u.ID == id {
                return c.JSON(u)
            }
        }
        return c.Status(404).JSON(fiber.Map{"error": "User not found"})
    })
    
    // 创建用户
    app.Post("/api/users", func(c *fiber.Ctx) error {
        var user User
        if err := c.BodyParser(&user); err != nil {
            return c.Status(400).JSON(fiber.Map{"error": "Invalid request"})
        }
        user.ID = len(users) + 1
        users = append(users, user)
        return c.Status(201).JSON(user)
    })
    
    // 更新用户
    app.Put("/api/users/:id", func(c *fiber.Ctx) error {
        id := c.ParamsInt("id")
        var update User
        if err := c.BodyParser(&update); err != nil {
            return c.Status(400).JSON(fiber.Map{"error": "Invalid request"})
        }
        
        for i, u := range users {
            if u.ID == id {
                users[i].Name = update.Name
                users[i].Age = update.Age
                return c.JSON(users[i])
            }
        }
        return c.Status(404).JSON(fiber.Map{"error": "User not found"})
    })
    
    // 删除用户
    app.Delete("/api/users/:id", func(c *fiber.Ctx) error {
        id := c.ParamsInt("id")
        for i, u := range users {
            if u.ID == id {
                users = append(users[:i], users[i+1:]...)
                return c.SendStatus(204)
            }
        }
        return c.Status(404).JSON(fiber.Map{"error": "User not found"})
    })
    
    app.Listen(":3000")
}
```

### 练习2: 中间件验证

```go
package main

import (
    "github.com/gofiber/fiber/v2"
    "github.com/gofiber/fiber/v2/middleware/logger"
    "github.com/gofiber/fiber/v2/middleware/recover"
)

func authMiddleware(c *fiber.Ctx) error {
    token := c.Get("Authorization")
    if token == "" {
        return c.Status(401).JSON(fiber.Map{
            "error": "Unauthorized",
        })
    }
    return c.Next()
}

func main() {
    app := fiber.New()
    
    app.Use(logger.New())
    app.Use(recover.New())
    
    app.Get("/public", func(c *fiber.Ctx) error {
        return c.JSON(fiber.Map{"message": "Public endpoint"})
    })
    
    api := app.Group("/api", authMiddleware)
    
    api.Get("/protected", func(c *fiber.Ctx) error {
        return c.JSON(fiber.Map{"message": "Protected endpoint"})
    })
    
    app.Listen(":3000")
}
```

### 挑战题: Todo API + Bun

```go
package main

import (
    "context"
    "time"
    
    "github.com/gofiber/fiber/v2"
    "github.com/uptrace/bun"
    _ "modernc.org/sqlite"
)

type Todo struct {
    ID        int64     `bun:"id,pk,autoincrement"`
    Title     string    `bun:"title,notnull"`
    Completed bool      `bun:"completed"`
    CreatedAt time.Time `bun:"created_at"`
}

func main() {
    db, _ := bun.OpenDBConnection("sqlite:/tmp/todos.sqlite")
    db.NewCreateTableModel().IfNotExists().Model(&Todo{}).Exec(context.Background())
    
    app := fiber.New()
    
    // 列表
    app.Get("/api/todos", func(c *fiber.Ctx) error {
        ctx := context.Background()
        var todos []Todo
        db.NewSelectModel().Model(&todos).Order("id DESC").Scan(ctx)
        return c.JSON(todos)
    })
    
    // 创建
    app.Post("/api/todos", func(c *fiber.Ctx) error {
        var todo Todo
        if err := c.BodyParser(&todo); err != nil {
            return c.Status(400).JSON(fiber.Map{"error": "Invalid request"})
        }
        
        ctx := context.Background()
        todo.CreatedAt = time.Now()
        db.NewInsert().Model(&todo).Exec(ctx)
        
        return c.Status(201).JSON(todo)
    })
    
    // 切换完成状态
    app.Put("/api/todos/:id/toggle", func(c *fiber.Ctx) error {
        id := c.ParamsInt("id")
        ctx := context.Background()
        
        var todo Todo
        if err := db.NewSelectModel().Model(&todo).Where("id = ?", id).Scan(ctx); err != nil {
            return c.Status(404).JSON(fiber.Map{"error": "Not found"})
        }
        
        todo.Completed = !todo.Completed
        db.NewUpdate().Model(&todo).WherePK().Exec(ctx)
        
        return c.JSON(todo)
    })
    
    // 删除
    app.Delete("/api/todos/:id", func(c *fiber.Ctx) error {
        id := c.ParamsInt("id")
        ctx := context.Background()
        
        db.NewDelete().Model((*Todo)(nil)).Where("id = ?", id).Exec(ctx)
        
        return c.SendStatus(204)
    })
    
    app.Listen(":3000")
}
```

---

## 下一步

- **Day 20-21**: 工程化（Makefile、测试、CI/CD）

---

*祝练习愉快！*
