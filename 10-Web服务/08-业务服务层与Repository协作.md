# 业务服务层与 Repository 协作

service 层表达业务用例，repository 层隔离数据访问。它们之间的边界清楚后，Web 服务会更容易测试，也更容易替换存储实现。

---

## 1. Service 负责业务规则

以任务服务为例，创建任务时可能有这些规则：

- 标题不能为空。
- 标题长度有限制。
- 新任务默认状态是 `open`。
- 创建时间和更新时间由服务端生成。
- ID 由服务端生成。

这些规则不适合散落在 handler 和 repository 中。

```go
package task

import (
	"context"
	"fmt"
	"strings"
	"time"
)

type CreateInput struct {
	Title string
}

type Service struct {
	repo  Repository
	newID func() string
	now   func() time.Time
}

func (s *Service) Create(ctx context.Context, input CreateInput) (Task, error) {
	title := strings.TrimSpace(input.Title)
	if title == "" {
		return Task{}, ErrInvalidTitle
	}
	if len([]rune(title)) > 120 {
		return Task{}, ErrInvalidTitle
	}

	now := s.now().UTC()
	task := Task{
		ID:        s.newID(),
		Title:     title,
		Status:    StatusOpen,
		CreatedAt: now,
		UpdatedAt: now,
	}

	if err := s.repo.Create(ctx, task); err != nil {
		return Task{}, fmt.Errorf("create task: %w", err)
	}
	return task, nil
}
```

handler 做过请求校验，service 仍然要保护自己的业务边界。因为 service 也可能被 CLI、后台任务、测试或其他入口调用。

---

## 2. Repository 接口表达数据需求

```go
package task

import "context"

type Repository interface {
	Create(ctx context.Context, task Task) error
	FindByID(ctx context.Context, id string) (Task, error)
	List(ctx context.Context, filter ListFilter) ([]Task, error)
	Update(ctx context.Context, task Task) error
	Delete(ctx context.Context, id string) error
}

type ListFilter struct {
	Status Status
	Limit  int
	Offset int
}
```

接口方法应该围绕业务用例设计，而非机械暴露所有数据库操作。

---

## 3. 内存 Repository

内存实现适合开发和测试。并发访问时要加锁：

```go
package task

import (
	"context"
	"sync"
)

type MemoryRepository struct {
	mu    sync.RWMutex
	tasks map[string]Task
}

func NewMemoryRepository() *MemoryRepository {
	return &MemoryRepository{
		tasks: make(map[string]Task),
	}
}

func (r *MemoryRepository) Create(ctx context.Context, task Task) error {
	if err := ctx.Err(); err != nil {
		return err
	}

	r.mu.Lock()
	defer r.mu.Unlock()

	if _, exists := r.tasks[task.ID]; exists {
		return ErrDuplicateID
	}
	r.tasks[task.ID] = task
	return nil
}

func (r *MemoryRepository) FindByID(ctx context.Context, id string) (Task, error) {
	if err := ctx.Err(); err != nil {
		return Task{}, err
	}

	r.mu.RLock()
	defer r.mu.RUnlock()

	task, exists := r.tasks[id]
	if !exists {
		return Task{}, ErrNotFound
	}
	return task, nil
}
```

锁保护 map，`ctx.Err()` 让调用方取消时能尽早返回。

---

## 4. SQL Repository

数据库实现隐藏 SQL 细节：

```go
package task

import (
	"context"
	"database/sql"
	"errors"
	"fmt"
)

type SQLRepository struct {
	db *sql.DB
}

func NewSQLRepository(db *sql.DB) (*SQLRepository, error) {
	if db == nil {
		return nil, fmt.Errorf("db is required")
	}
	return &SQLRepository{db: db}, nil
}

func (r *SQLRepository) FindByID(ctx context.Context, id string) (Task, error) {
	const query = `
SELECT id, title, status, created_at, updated_at
FROM tasks
WHERE id = ?
`
	var task Task
	err := r.db.QueryRowContext(ctx, query, id).Scan(
		&task.ID,
		&task.Title,
		&task.Status,
		&task.CreatedAt,
		&task.UpdatedAt,
	)
	if errors.Is(err, sql.ErrNoRows) {
		return Task{}, ErrNotFound
	}
	if err != nil {
		return Task{}, fmt.Errorf("query task by id: %w", err)
	}
	return task, nil
}
```

SQL 占位符要根据驱动调整。SQLite 和 MySQL 常见 `?`，PostgreSQL 常见 `$1`、`$2`。

---

## 5. 事务边界

一次业务操作如果需要多次写入，事务边界通常在 service 层表达：

```go
package task

import (
	"context"
	"time"
)

type UnitOfWork interface {
	WithTx(ctx context.Context, fn func(ctx context.Context, repo Repository) error) error
}

type Service struct {
	repo  Repository
	uow   UnitOfWork
	newID func() string
	now   func() time.Time
}
```

使用方式：

```go
func (s *Service) Complete(ctx context.Context, id string) error {
	return s.uow.WithTx(ctx, func(ctx context.Context, repo Repository) error {
		task, err := repo.FindByID(ctx, id)
		if err != nil {
			return err
		}
		if task.Status == StatusDone {
			return ErrAlreadyDone
		}
		task.Status = StatusDone
		return repo.Update(ctx, task)
	})
}
```

事务接口不一定每个项目都需要。只有当一个用例必须保证多步数据变更一致时，再引入它。

---

## 6. Handler 调用 Service

handler 不关心 repository：

```go
func (h *HTTPHandler) Complete(w http.ResponseWriter, r *http.Request) {
	id := r.PathValue("id")
	if id == "" {
		httpx.WriteErrorWithCode(w, http.StatusBadRequest, "missing_id", "missing task id")
		return
	}

	if err := h.service.Complete(r.Context(), id); err != nil {
		writeTaskError(w, err)
		return
	}

	w.WriteHeader(http.StatusNoContent)
}
```

这样 handler 的测试可以使用假的 service，service 的测试可以使用假的 repository。

---

## 7. 学习检查

检查 service 和 repository 时，重点看：

1. 业务规则是否集中在 service。
2. repository 接口是否符合业务用例。
3. context 是否传到 repository。
4. 数据库错误是否包装并映射。
5. 内存实现是否并发安全。
6. 事务是否只在需要一致性时引入。
7. handler 是否绕过 service 直接访问 repository。
