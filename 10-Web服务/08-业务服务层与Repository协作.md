# 08: 业务服务层与 Repository 协作

前面几篇把 HTTP 边界修好了,这篇往里走一层,解决的问题是:**业务规则写在哪、数据访问怎么隔离,两层之间怎么接**。学完你能写出"handler 换个假 service 就能测、service 换个假 repository 就能测、换数据库不动业务代码"的结构。

对比 Node:service/repository 分层在 NestJS 里是一等公民,Express 项目里也常见(controller → service → DAO/Prisma)。概念完全平移,Go 的不同点在于:repository 是**接口**(隐式实现),隔离是编译器保证的,不是口头约定。

---

## 1. Service 负责业务规则

创建任务这个用例背后藏着一堆规则:标题 trim 后非空、长度有限、新任务默认 `open`、时间和 ID 由服务端生成。这些规则的家在 service:

```go
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
    if utf8.RuneCountInString(title) > 120 {
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

### 🕳️ 坑:handler 校验过了,service 就不用再校验?

以为会怎样:第 05 篇的 DTO `Validate` 已经查过空标题,service 再查是重复劳动。
实际怎样:service 还可能被 CLI、定时任务、别的 service、测试直接调用——这些入口都不经过 HTTP 的 DTO 校验。
为什么:DTO 校验保护的是 **HTTP 边界**,service 校验保护的是**业务不变量**。两道门守的不是同一个院子。

实跑 service 层的规则(注入固定 ID 和时钟,不经过 HTTP):

```text
空标题: invalid task title | 是 ErrInvalidTitle? true
创建: task_1 open
完成: task_1 done
重复完成: task already done | 是 ErrAlreadyDone? true
```

状态流转规则("done 不能再 done")牢牢锁在 service 里,谁来调都绕不过去。

---

## 2. Repository 接口表达数据需求

```go
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

方法围绕**业务用例**设计,不是机械翻译数据库操作。判断标准:接口里出现 `WHERE`、`JOIN`、表名这类字眼(哪怕变形出现,比如 `QueryByStatusJoinUser`)就是漏了底;`ListFilter` 这样的参数对象则是干净的"业务想要什么"。

**一句话总结:repository 接口是 service 开给存储层的需求清单,不是数据库 API 的镜像。**

---

## 3. 内存 Repository

内存实现在开发和测试阶段顶大用——map + 锁(模块 06 第 08 篇的 RWMutex 直接上岗):

```go
type MemoryRepository struct {
    mu    sync.RWMutex
    tasks map[string]Task
}

func NewMemoryRepository() *MemoryRepository {
    return &MemoryRepository{tasks: make(map[string]Task)}
}

func (r *MemoryRepository) Create(ctx context.Context, task Task) error {
    if err := ctx.Err(); err != nil {
        return err          // 调用方已取消,尽早退出
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

🕳️ 坑:HTTP 服务器**每个请求一个 goroutine**,不加锁的 map 在两个并发请求下就是数据竞争——`go test -race` 会当场抓获(本模块的测试实测通过 `-race`)。JS 单线程惯出来的"共享对象随便改"的手感,在这里必须扔掉。

读多写少,读用 `RLock`(可并发)、写用 `Lock`(独占);`ctx.Err()` 让被取消的请求别再白干活。

---

## 4. SQL Repository

换成数据库,实现变了,接口一个字不改(模块 09 的 `database/sql` 在这儿归位):

```go
type SQLRepository struct {
    db *sql.DB
}

func (r *SQLRepository) FindByID(ctx context.Context, id string) (Task, error) {
    const query = `
SELECT id, title, status, created_at, updated_at
FROM tasks
WHERE id = ?
`
    var task Task
    err := r.db.QueryRowContext(ctx, query, id).Scan(
        &task.ID, &task.Title, &task.Status, &task.CreatedAt, &task.UpdatedAt,
    )
    if errors.Is(err, sql.ErrNoRows) {
        return Task{}, ErrNotFound      // 关键:翻译成业务错误
    }
    if err != nil {
        return Task{}, fmt.Errorf("query task by id: %w", err)
    }
    return task, nil
}
```

最重要的一行是 `sql.ErrNoRows → ErrNotFound` 的**翻译**:service 和 handler 只认识 `ErrNotFound`,它们不需要知道底下是 SQL 还是 map。第 06 篇的 404 映射之所以对两种 repository 都生效,靠的就是这次翻译。

占位符按驱动调整:SQLite/MySQL 用 `?`,PostgreSQL 用 `$1`、`$2`。

---

## 5. 事务边界

一次业务操作要写多张表时,事务的**边界**在 service(它知道哪几步必须同生共死),事务的**实现**在存储层。用一个接口把两边接起来:

```go
type UnitOfWork interface {
    WithTx(ctx context.Context, fn func(ctx context.Context, repo Repository) error) error
}
```

service 这样用(结构示意,配合模块 09 第 07 篇的事务实现):

```go
func (s *Service) Complete(ctx context.Context, id string) error {
    return s.uow.WithTx(ctx, func(ctx context.Context, repo Repository) error {
        task, err := repo.FindByID(ctx, id)     // 事务里的读
        if err != nil {
            return err
        }
        if task.Status == StatusDone {
            return ErrAlreadyDone               // 返回错误 = 回滚
        }
        task.Status = StatusDone
        return repo.Update(ctx, task)           // 返回 nil = 提交
    })
}
```

回调返回错误就回滚、返回 nil 就提交——形状和 Prisma 的 `$transaction(async (tx) => {...})` 一模一样。

🕳️ 提醒:别一上来就给每个项目配 UnitOfWork。单表操作用不上事务,内存 repository 更没有事务。**只有当一个用例必须保证多步写入的一致性时,再引入它。**

---

## 6. Handler 调用 Service

链条的最后一环,handler 只认识 service:

```go
func (h *HTTPHandler) Complete(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")

    completed, err := h.service.Complete(r.Context(), id)
    if err != nil {
        writeTaskError(w, err)
        return
    }
    if err := httpx.WriteJSON(w, http.StatusOK, taskResponseFromModel(completed)); err != nil {
        h.logger.Error("write response failed", slog.String("error", err.Error()))
    }
}
```

整条链经过 HTTP 实测:

```bash
curl -s  -X POST http://localhost:19080/tasks/task_1/complete
curl -si -X POST http://localhost:19080/tasks/task_1/complete | head -1
```

```text
{"id":"task_1","title":"learn go web deeply","status":"done","created_at":"2026-07-16T17:14:04.298555Z","updated_at":"2026-07-16T17:14:56.489324Z"}
HTTP/1.1 409 Conflict
```

第 1 节 service 直接调用时的 `ErrAlreadyDone`,经过第 06 篇的映射变成了这里的 409——三层各司其职,一条规则只写了一次。

**一句话总结:handler 测试换假 service,service 测试换假 repository,每层都能被单独按住测。**

---

## 本篇重点

- [ ] 业务规则集中在 service;DTO 校验守 HTTP 门,service 校验守业务不变量,两道门都要有。
- [ ] repository 接口按业务用例设计,别镜像数据库 API;`sql.ErrNoRows` 这类底层错误在 repository 里翻译成业务错误。
- [ ] 内存实现必须加锁——每个请求一个 goroutine,不加锁的 map 是数据竞争;读 `RLock` 写 `Lock`。
- [ ] context 一路传到 repository,`ctx.Err()` 让取消尽早生效。
- [ ] 事务边界在 service、实现在存储层,用 `WithTx(func) ` 接口连接;需要多步一致性时才引入。

---

## 练习

给 `book-api` 完成 service + repository 两层:

要求:

1. `Service.Borrow(ctx, id)`:书不存在返回 `ErrNotFound`,已借出返回 `ErrAlreadyBorrowed`,成功则状态变 `borrowed` 并刷新 `UpdatedAt`。
2. 实现并发安全的 `MemoryRepository`(RWMutex + `ctx.Err()`)。
3. 写 service 测试:注入固定时钟,覆盖"成功借出 / 重复借出 / 书不存在"三条路径,用 `errors.Is` 断言。
4. 跑 `go test -race ./...`,然后故意把某个方法的锁去掉,并发跑两个 goroutine 各 Borrow 一次,看看 `-race` 报什么。
5. 思考题:`Borrow` 的"查状态 + 改状态"两步之间,内存实现里会不会被并发穿插?锁应该锁多大范围?

提示:第 5 题的答案决定了锁该加在 repository 单个方法上还是 service 需要更粗的保护——对照第 5 节"事务边界在 service"想一想。
