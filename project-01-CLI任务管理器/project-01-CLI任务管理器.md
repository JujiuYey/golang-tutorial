# Project 01: CLI 任务管理器

> 预计时长: 8小时  
> 技术栈: Flag / Cobra + 文件存储 (JSON)

---

## 项目介绍

开发一个命令行任务管理器（Todo CLI），支持任务的增删改查功能。

---

## 功能需求

1. **添加任务** - 添加新任务到列表
2. **列出任务** - 显示所有任务
3. **完成任务** - 将任务标记为已完成
4. **删除任务** - 删除指定任务
5. **数据持久化** - 使用JSON文件存储数据

---

## 项目结构

```
todo-cli/
├── cmd/
│   └── root.go           # 根命令
│   └── add.go            # 添加命令
│   └── list.go           # 列表命令
│   └── done.go            # 完成命令
│   └── delete.go         # 删除命令
├── model/
│   └── task.go           # Task模型
├── repository/
│   └── task_repo.go      # 数据存储
├── main.go
├── go.mod
└── Makefile
```

---

## 代码实现

### 1. 初始化项目

```bash
mkdir todo-cli && cd todo-cli
go mod init todo-cli
```

### 2. Model层

```go
// model/task.go
package model

import "time"

type Task struct {
    ID        int64     `json:"id"`
    Title     string    `json:"title"`
    Completed bool      `json:"completed"`
    CreatedAt time.Time `json:"created_at"`
}

type TaskList struct {
    Tasks    []Task `json:"tasks"`
    NextID    int64 `json:"next_id"`
}
```

### 3. Repository层

```go
// repository/task_repo.go
package repository

import (
    "encoding/json"
    "os"
    "todo-cli/model"
)

type TaskRepository struct {
    filepath string
}

func NewTaskRepository(filepath string) *TaskRepository {
    return &TaskRepository{filepath: filepath}
}

func (r *TaskRepository) Load() (*model.TaskList, error) {
    data, err := os.ReadFile(r.filepath)
    if err != nil {
        if os.IsNotExist(err) {
            return &model.TaskList{
                Tasks: []model.Task{},
                NextID: 1,
            }, nil
        }
        return nil, err
    }
    
    var list model.TaskList
    if err := json.Unmarshal(data, &list); err != nil {
        return nil, err
    }
    
    return &list, nil
}

func (r *TaskRepository) Save(list *model.TaskList) error {
    data, err := json.MarshalIndent(list, "", "  ")
    if err != nil {
        return err
    }
    return os.WriteFile(r.filepath, data, 0644)
}
```

### 4. 命令实现

```go
// cmd/add.go
package cmd

import (
    "fmt"
    "time"
    
    "github.com/spf13/cobra"
    "todo-cli/model"
    "todo-cli/repository"
)

var addCmd = &cobra.Command{
    Use:   "add",
    Short: "添加新任务",
    RunE: func(cmd *cobra.Command, args []string) error {
        if len(args) == 0 {
            return fmt.Errorf("请提供任务标题")
        }
        
        title := args[0]
        repo := repository.NewTaskRepository("tasks.json")
        
        list, err := repo.Load()
        if err != nil {
            return err
        }
        
        task := model.Task{
            ID:        list.NextID,
            Title:     title,
            Completed: false,
            CreatedAt: time.Now(),
        }
        
        list.Tasks = append(list.Tasks, task)
        list.NextID++
        
        if err := repo.Save(list); err != nil {
            return err
        }
        
        fmt.Printf("✓ 任务已添加 (ID: %d)\n", task.ID)
        return nil
    },
}
```

```go
// cmd/list.go
package cmd

import (
    "fmt"
    
    "github.com/spf13/cobra"
    "todo-cli/repository"
)

var listCmd = &cobra.Command{
    Use:   "list",
    Short: "列出所有任务",
    RunE: func(cmd *cobra.Command, args []string) error {
        repo := repository.NewTaskRepository("tasks.json")
        
        list, err := repo.Load()
        if err != nil {
            return err
        }
        
        if len(list.Tasks) == 0 {
            fmt.Println("没有任务")
            return nil
        }
        
        fmt.Println("\n任务列表:")
        fmt.Println("----------------------------------------")
        
        for _, task := range list.Tasks {
            status := "○"
            if task.Completed {
                status = "●"
            }
            fmt.Printf("[%s] %d. %s\n", status, task.ID, task.Title)
        }
        
        fmt.Println("----------------------------------------")
        return nil
    },
}
```

```go
// cmd/done.go
package cmd

import (
    "fmt"
    "strconv"
    
    "github.com/spf13/cobra"
    "todo-cli/repository"
)

var doneCmd = &cobra.Command{
    Use:   "done",
    Short: "完成任务",
    RunE: func(cmd *cobra.Command, args []string) error {
        if len(args) == 0 {
            return fmt.Errorf("请提供任务ID")
        }
        
        id, err := strconv.ParseInt(args[0], 10, 64)
        if err != nil {
            return fmt.Errorf("无效的任务ID")
        }
        
        repo := repository.NewTaskRepository("tasks.json")
        list, err := repo.Load()
        if err != nil {
            return err
        }
        
        found := false
        for i := range list.Tasks {
            if list.Tasks[i].ID == id {
                list.Tasks[i].Completed = true
                found = true
                break
            }
        }
        
        if !found {
            return fmt.Errorf("未找到任务 #%d", id)
        }
        
        if err := repo.Save(list); err != nil {
            return err
        }
        
        fmt.Printf("✓ 任务 #%d 已完成\n", id)
        return nil
    },
}
```

```go
// cmd/delete.go
package cmd

import (
    "fmt"
    "strconv"
    
    "github.com/spf13/cobra"
    "todo-cli/repository"
)

var deleteCmd = &cobra.Command{
    Use:   "delete",
    Short: "删除任务",
    RunE: func(cmd *cobra.Command, args []string) error {
        if len(args) == 0 {
            return fmt.Errorf("请提供任务ID")
        }
        
        id, err := strconv.ParseInt(args[0], 10, 64)
        if err != nil {
            return fmt.Errorf("无效的任务ID")
        }
        
        repo := repository.NewTaskRepository("tasks.json")
        list, err := repo.Load()
        if err != nil {
            return err
        }
        
        found := false
        for i := range list.Tasks {
            if list.Tasks[i].ID == id {
                list.Tasks = append(list.Tasks[:i], list.Tasks[i+1:]...)
                found = true
                break
            }
        }
        
        if !found {
            return fmt.Errorf("未找到任务 #%d", id)
        }
        
        if err := repo.Save(list); err != nil {
            return err
        }
        
        fmt.Printf("✓ 任务 #%d 已删除\n", id)
        return nil
    },
}
```

### 5. 根命令

```go
// cmd/root.go
package cmd

import (
    "github.com/spf13/cobra"
)

var rootCmd = &cobra.Command{
    Use:   "todo",
    Short: "任务管理器 CLI",
}

func init() {
    rootCmd.AddCommand(addCmd)
    rootCmd.AddCommand(listCmd)
    rootCmd.AddCommand(doneCmd)
    rootCmd.AddCommand(deleteCmd)
}

func Execute() error {
    return rootCmd.Execute()
}
```

### 6. 主入口

```go
// main.go
package main

import (
    "os"
    "todo-cli/cmd"
)

func main() {
    if err := cmd.Execute(); err != nil {
        os.Exit(1)
    }
}
```

### 7. Makefile

```makefile
.PHONY: build clean install run test

BINARY=todo
BUILD_DIR=build

build:
    go build -o $(BUILD_DIR)/$(BINARY) .

clean:
    rm -rf $(BUILD_DIR)

run: build
    ./$(BUILD_DIR)/$(BINARY)

install: build
    mv $(BUILD_DIR)/$(BINARY) /usr/local/bin/

test:
    go test -v ./...
```

---

## 使用方法

```bash
# 构建
make build

# 添加任务
./build/todo add "学习Go语言"
./build/todo add "完成项目作业"

# 列出任务
./build/todo list

# 完成任务
./build/todo done 1

# 删除任务
./build/todo delete 2
```

---

## 扩展功能

1. **分类标签** - 为任务添加分类
2. **优先级** - 高/中/低优先级
3. **截止日期** - 设置任务截止时间
4. **搜索筛选** - 按关键字或状态筛选
5. **导出CSV** - 导出任务到CSV文件

---

*项目完成！*
