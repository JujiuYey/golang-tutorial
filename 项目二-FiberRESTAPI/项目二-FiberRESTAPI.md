# 项目二: Fiber REST API + Bun

> 预计时长: 8小时  
> 技术栈: Fiber + Bun + SQLite + JWT

---

## 项目介绍

开发一个完整的用户认证REST API，支持用户注册、登录、JWT认证和用户管理。

---

## 功能需求

1. **用户注册** - 创建新用户
2. **用户登录** - 获取JWT Token
3. **获取用户列表** - 需要认证
4. **获取当前用户** - 需要认证
5. **更新用户信息** - 需要认证

---

## 项目结构

```
user-api/
├── cmd/
│   └── server/
│       └── main.go           # 入口
├── internal/
│   ├── config/
│   │   └── config.go         # 配置
│   ├── handler/
│   │   └── user.go          # HTTP处理
│   ├── middleware/
│   │   └── auth.go          # JWT认证
│   ├── model/
│   │   └── user.go          # 数据模型
│   ├── repository/
│   │   └── user.go          # 数据库操作
│   └── service/
│       └── user.go          # 业务逻辑
├── pkg/
│   └── response/
│       └── response.go       # 统一响应
├── go.mod
├── Makefile
└── config.yaml
```

---

## 代码实现

### 1. 初始化项目

```bash
mkdir user-api && cd user-api
go mod init user-api
go get github.com/gofiber/fiber/v2
go get github.com/uptrace/bun
go get github.com/uptrace/bun/extra/bundebug
go get github.com/go-sql-driver/mysql
go get github.com/golang-jwt/jwt/v5
go get gopkg.in/yaml.v3
```

### 2. 配置文件

```go
// config/config.go
package config

import (
    "os"
    "gopkg.in/yaml.v3"
)

type Config struct {
    Database DatabaseConfig `yaml:"database"`
    App      AppConfig      `yaml:"app"`
    JWT      JWTConfig      `yaml:"jwt"`
}

type DatabaseConfig struct {
    Driver string `yaml:"driver"`
    DSN    string `yaml:"dsn"`
}

type AppConfig struct {
    Host string `yaml:"host"`
    Port int    `yaml:"port"`
}

type JWTConfig struct {
    Secret     string `yaml:"secret"`
    Expiration int    `yaml:"expiration"` // 小时
}

func Load(path string) (*Config, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        return nil, err
    }
    
    var cfg Config
    if err := yaml.Unmarshal(data, &cfg); err != nil {
        return nil, err
    }
    
    return &cfg, nil
}
```

```yaml
# config.yaml
database:
  driver: sqlite
  dsn: /tmp/users.sqlite

app:
  host: 0.0.0.0
  port: 8080

jwt:
  secret: your-secret-key-change-in-production
  expiration: 24
```

### 3. Model层

```go
// model/user.go
package model

import (
    "time"
    "github.com/uptrace/bun"
)

type User struct {
    bun.BaseModel `bun:"table:users"`
    
    ID        int64     `bun:"id,pk,autoincrement"`
    Username  string    `bun:"username,unique,notnull"`
    Email     string    `bun:"email,unique,notnull"`
    Password  string    `bun:"password,notnull"`  // bcrypt哈希
    CreatedAt time.Time `bun:"created_at"`
    UpdatedAt time.Time `bun:"updated_at"`
}

type RegisterRequest struct {
    Username string `json:"username"`
    Email    string `json:"email"`
    Password string `json:"password"`
}

type LoginRequest struct {
    Username string `json:"username"`
    Password string `json:"password"`
}

type LoginResponse struct {
    Token string `json:"token"`
    User  User   `json:"user"`
}
```

### 4. Repository层

```go
// repository/user.go
package repository

import (
    "context"
    "user-api/model"
    
    "github.com/uptrace/bun"
)

type UserRepository struct {
    db *bun.DB
}

func NewUserRepository(db *bun.DB) *UserRepository {
    return &UserRepository{db: db}
}

func (r *UserRepository) Create(ctx context.Context, user *model.User) error {
    _, err := r.db.NewInsert().Model(user).Exec(ctx)
    return err
}

func (r *UserRepository) FindByID(ctx context.Context, id int64) (*model.User, error) {
    var user model.User
    err := r.db.NewSelectModel().Model(&user).
        Where("id = ?", id).
        Scan(ctx)
    if err != nil {
        return nil, err
    }
    return &user, nil
}

func (r *UserRepository) FindByUsername(ctx context.Context, username string) (*model.User, error) {
    var user model.User
    err := r.db.NewSelectModel().Model(&user).
        Where("username = ?", username).
        Scan(ctx)
    if err != nil {
        return nil, err
    }
    return &user, nil
}

func (r *UserRepository) FindAll(ctx context.Context) ([]model.User, error) {
    var users []model.User
    err := r.db.NewSelectModel().Model(&users).
        Order("id ASC").
        Scan(ctx)
    return users, err
}

func (r *UserRepository) Update(ctx context.Context, user *model.User) error {
    _, err := r.db.NewUpdate().Model(user).WherePK().Exec(ctx)
    return err
}
```

### 5. Service层

```go
// service/user.go
package service

import (
    "context"
    "errors"
    "time"
    
    "user-api/model"
    "user-api/repository"
    
    "github.com/golang-jwt/jwt/v5"
    "golang.org/x/crypto/bcrypt"
)

var (
    ErrUserNotFound     = errors.New("用户不存在")
    ErrUserExists       = errors.New("用户已存在")
    ErrInvalidPassword  = errors.New("密码错误")
)

type UserService struct {
    repo   *repository.UserRepository
    secret string
    expiration int // 小时
}

func NewUserService(repo *repository.UserRepository, secret string, expiration int) *UserService {
    return &UserService{
        repo:   repo,
        secret: secret,
        expiration: expiration,
    }
}

func (s *UserService) Register(ctx context.Context, req *model.RegisterRequest) (*model.User, error) {
    // 检查是否已存在
    if _, err := s.repo.FindByUsername(ctx, req.Username); err == nil {
        return nil, ErrUserExists
    }
    
    // 密码哈希
    hashedPassword, err := bcrypt.GenerateFromPassword([]byte(req.Password), bcrypt.DefaultCost)
    if err != nil {
        return nil, err
    }
    
    user := &model.User{
        Username: req.Username,
        Email:    req.Email,
        Password: string(hashedPassword),
    }
    
    if err := s.repo.Create(ctx, user); err != nil {
        return nil, err
    }
    
    return user, nil
}

func (s *UserService) Login(ctx context.Context, req *model.LoginRequest) (*model.LoginResponse, error) {
    user, err := s.repo.FindByUsername(ctx, req.Username)
    if err != nil {
        return nil, ErrUserNotFound
    }
    
    if err := bcrypt.CompareHashAndPassword([]byte(user.Password), []byte(req.Password)); err != nil {
        return nil, ErrInvalidPassword
    }
    
    // 生成JWT
    token, err := s.generateToken(user.ID)
    if err != nil {
        return nil, err
    }
    
    return &model.LoginResponse{
        Token: token,
        User:  *user,
    }, nil
}

func (s *UserService) generateToken(userID int64) (string, error) {
    claims := jwt.MapClaims{
        "user_id": userID,
        "exp":     time.Now().Add(time.Hour * time.Duration(s.expiration)).Unix(),
        "iat":     time.Now().Unix(),
    }
    
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString([]byte(s.secret))
}

func (s *UserService) GetUser(ctx context.Context, id int64) (*model.User, error) {
    return s.repo.FindByID(ctx, id)
}

func (s *UserService) ListUsers(ctx context.Context) ([]model.User, error) {
    return s.repo.FindAll(ctx)
}

func (s *UserService) ParseToken(tokenString string) (int64, error) {
    token, err := jwt.Parse(tokenString, func(token *jwt.Token) (interface{}, error) {
        return []byte(s.secret), nil
    })
    
    if err != nil {
        return 0, err
    }
    
    if claims, ok := token.Claims.(jwt.MapClaims); ok && token.Valid {
        userID := int64(claims["user_id"].(float64))
        return userID, nil
    }
    
    return 0, errors.New("invalid token")
}
```

### 6. 中间件

```go
// middleware/auth.go
package middleware

import (
    "strconv"
    
    "user-api/service"
    
    "github.com/gofiber/fiber/v2"
)

func JWTAuth(svc *service.UserService) fiber.Handler {
    return func(c *fiber.Ctx) error {
        token := c.Get("Authorization")
        if token == "" {
            return c.Status(401).JSON(fiber.Map{
                "error": "缺少认证令牌",
            })
        }
        
        // Bearer token
        if len(token) > 7 && token[:7] == "Bearer " {
            token = token[7:]
        }
        
        userID, err := svc.ParseToken(token)
        if err != nil {
            return c.Status(401).JSON(fiber.Map{
                "error": "无效的令牌",
            })
        }
        
        // 存储用户ID到上下文
        c.Locals("userID", userID)
        return c.Next()
    }
}

func GetUserID(c *fiber.Ctx) int64 {
    if id, ok := c.Locals("userID").(int64); ok {
        return id
    }
    return 0
}
```

### 7. Handler层

```go
// handler/user.go
package handler

import (
    "user-api/model"
    "user-api/service"
    
    "github.com/gofiber/fiber/v2"
)

type UserHandler struct {
    svc *service.UserService
}

func NewUserHandler(svc *service.UserService) *UserHandler {
    return &UserHandler{svc: svc}
}

func (h *UserHandler) Register(c *fiber.Ctx) error {
    var req model.RegisterRequest
    if err := c.BodyParser(&req); err != nil {
        return c.Status(400).JSON(fiber.Map{
            "error": "无效的请求",
        })
    }
    
    if req.Username == "" || req.Password == "" {
        return c.Status(400).JSON(fiber.Map{
            "error": "用户名和密码不能为空",
        })
    }
    
    user, err := h.svc.Register(c.Context(), &req)
    if err != nil {
        if err == service.ErrUserExists {
            return c.Status(409).JSON(fiber.Map{
                "error": "用户已存在",
            })
        }
        return c.Status(500).JSON(fiber.Map{
            "error": "注册失败",
        })
    }
    
    return c.Status(201).JSON(fiber.Map{
        "message": "注册成功",
        "user": fiber.Map{
            "id":       user.ID,
            "username": user.Username,
            "email":    user.Email,
        },
    })
}

func (h *UserHandler) Login(c *fiber.Ctx) error {
    var req model.LoginRequest
    if err := c.BodyParser(&req); err != nil {
        return c.Status(400).JSON(fiber.Map{
            "error": "无效的请求",
        })
    }
    
    resp, err := h.svc.Login(c.Context(), &req)
    if err != nil {
        if err == service.ErrUserNotFound || err == service.ErrInvalidPassword {
            return c.Status(401).JSON(fiber.Map{
                "error": "用户名或密码错误",
            })
        }
        return c.Status(500).JSON(fiber.Map{
            "error": "登录失败",
        })
    }
    
    return c.JSON(resp)
}

func (h *UserHandler) GetMe(c *fiber.Ctx) error {
    userID := c.Locals("userID").(int64)
    
    user, err := h.svc.GetUser(c.Context(), userID)
    if err != nil {
        return c.Status(404).JSON(fiber.Map{
            "error": "用户不存在",
        })
    }
    
    return c.JSON(fiber.Map{
        "id":       user.ID,
        "username": user.Username,
        "email":    user.Email,
    })
}

func (h *UserHandler) ListUsers(c *fiber.Ctx) error {
    users, err := h.svc.ListUsers(c.Context())
    if err != nil {
        return c.Status(500).JSON(fiber.Map{
            "error": "获取列表失败",
        })
    }
    
    result := make([]fiber.Map, len(users))
    for i, u := range users {
        result[i] = fiber.Map{
            "id":       u.ID,
            "username": u.Username,
            "email":    u.Email,
        }
    }
    
    return c.JSON(result)
}
```

### 8. 主入口

```go
// cmd/server/main.go
package main

import (
    "context"
    "fmt"
    "log"
    "os"
    "os/signal"
    "syscall"
    
    "user-api/config"
    "user-api/internal/middleware"
    "user-api/internal/model"
    "user-api/internal/repository"
    "user-api/internal/service"
    "user-api/internal/handler"
    
    "github.com/gofiber/fiber/v2"
    "github.com/gofiber/fiber/v2/middleware/cors"
    "github.com/gofiber/fiber/v2/middleware/logger"
    "github.com/gofiber/fiber/v2/middleware/recover"
    "github.com/uptrace/bun"
    "github.com/uptrace/bun/extra/bundebug"
)

func main() {
    // 加载配置
    cfg, err := config.Load("config.yaml")
    if err != nil {
        log.Fatal("加载配置失败:", err)
    }
    
    // 连接数据库
    db, err := bun.OpenDBConnection(cfg.Database.DSN)
    if err != nil {
        log.Fatal("连接数据库失败:", err)
    }
    db.AddQueryHook(bundebug.NewQueryHook())
    
    // 创建表
    ctx := context.Background()
    db.NewCreateTableModel().IfNotExists().Model(&model.User{}).Exec(ctx)
    
    // 初始化服务
    userRepo := repository.NewUserRepository(db)
    userSvc := service.NewUserService(userRepo, cfg.JWT.Secret, cfg.JWT.Expiration)
    userHandler := handler.NewUserHandler(userSvc)
    
    // 创建Fiber应用
    app := fiber.New(fiber.Config{
        ErrorHandler: func(c *fiber.Ctx, err error) error {
            return c.Status(500).JSON(fiber.Map{
                "error": err.Error(),
            })
        },
    })
    
    // 全局中间件
    app.Use(logger.New())
    app.Use(recover.New())
    app.Use(cors.New())
    
    // 路由
    api := app.Group("/api")
    
    // 公开路由
    api.Post("/register", userHandler.Register)
    api.Post("/login", userHandler.Login)
    
    // 受保护路由
    protected := api.Group("", middleware.JWTAuth(userSvc))
    protected.Get("/me", userHandler.GetMe)
    protected.Get("/users", userHandler.ListUsers)
    
    // 启动服务器
    go func() {
        addr := fmt.Sprintf("%s:%d", cfg.App.Host, cfg.App.Port)
        log.Printf("服务器启动在 %s", addr)
        if err := app.Listen(addr); err != nil {
            log.Fatal("服务器启动失败:", err)
        }
    }()
    
    // 优雅关闭
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit
    
    log.Println("关闭服务器...")
    if err := app.Shutdown(); err != nil {
        log.Fatal("服务器关闭失败:", err)
    }
}
```

### 9. Makefile

```makefile
.PHONY: build run clean test

BINARY=user-api
BUILD_DIR=build

build:
    mkdir -p $(BUILD_DIR)
    go build -o $(BUILD_DIR)/$(BINARY) cmd/server/main.go

run: build
    ./$(BUILD_DIR)/$(BINARY)

clean:
    rm -rf $(BUILD_DIR)

test:
    go test -v ./...
```

---

## API文档

### 注册
```
POST /api/register
{
    "username": "zhangsan",
    "email": "zhangsan@example.com",
    "password": "123456"
}
```

### 登录
```
POST /api/login
{
    "username": "zhangsan",
    "password": "123456"
}

Response:
{
    "token": "eyJhbGciOiJIUzI1NiIs...",
    "user": {
        "id": 1,
        "username": "zhangsan",
        "email": "zhangsan@example.com"
    }
}
```

### 获取当前用户
```
GET /api/me
Authorization: Bearer <token>
```

### 获取用户列表
```
GET /api/users
Authorization: Bearer <token>
```

---

## 测试API

```bash
# 构建
make build

# 运行
./build/user-api

# 注册
curl -X POST http://localhost:8080/api/register \
  -H "Content-Type: application/json" \
  -d '{"username":"test","email":"test@example.com","password":"123456"}'

# 登录
curl -X POST http://localhost:8080/api/login \
  -H "Content-Type: application/json" \
  -d '{"username":"test","password":"123456"}'

# 获取用户列表
curl http://localhost:8080/api/users \
  -H "Authorization: Bearer <token>"
```

---

## 扩展功能

1. **邮箱验证** - 发送验证邮件
2. **密码重置** - 忘记密码功能
3. **刷新Token** - Access Token刷新
4. **权限控制** - 管理员/普通用户
5. **数据库切换** - MySQL/PostgreSQL

---

*项目完成！恭喜你完成Go语言学习课程！*
