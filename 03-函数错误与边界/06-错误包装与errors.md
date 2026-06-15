# 06: 错误包装、errors.Is 与 errors.As

错误不只是字符串。Go 支持错误包装，让调用方既能看到上下文，又能判断底层错误。

---

## 1. 使用 %w 包装错误

```go
func readConfig(path string) ([]byte, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        return nil, fmt.Errorf("read config %s: %w", path, err)
    }
    return data, nil
}
```

`%w` 会保留原始错误，调用方可以继续判断。

如果使用 `%v`，错误只会被格式化进字符串，原始错误链会丢失。

---

## 2. errors.Is

`errors.Is` 用于判断错误链中是否包含某个目标错误。

```go
data, err := readConfig("missing.yaml")
if err != nil {
    if errors.Is(err, os.ErrNotExist) {
        fmt.Println("配置文件不存在")
        return
    }
    fmt.Println(err)
    return
}
_ = data
```

---

## 3. 哨兵错误

包级变量错误常用于可比较的固定错误。

```go
var ErrNotFound = errors.New("not found")

func findUser(id int64) error {
    return ErrNotFound
}
```

调用方：

```go
err := findUser(1)
if errors.Is(err, ErrNotFound) {
    fmt.Println("user not found")
}
```

---

## 4. 自定义错误类型

当错误需要携带结构化信息时，可以定义错误类型。

```go
type ValidationError struct {
    Field string
    Msg   string
}

func (e *ValidationError) Error() string {
    return e.Field + ": " + e.Msg
}
```

---

## 5. errors.As

`errors.As` 用于从错误链中取出某种错误类型。

```go
err := &ValidationError{Field: "email", Msg: "不能为空"}

var validationErr *ValidationError
if errors.As(err, &validationErr) {
    fmt.Println(validationErr.Field)
}
```

---

## 6. 什么时候包装错误

推荐包装：

- 跨越函数边界时补充上下文。
- 底层错误对调用方仍然有判断价值。

不推荐包装：

- 只是为了重复一遍同样信息。
- 想隐藏底层错误，不希望调用方判断。

---

## 练习

实现函数：

```go
func readRequiredFile(path string) ([]byte, error)
```

要求：

1. 使用 `os.ReadFile` 读取文件。
2. 读取失败时用 `%w` 包装错误，带上文件路径。
3. 调用方使用 `errors.Is(err, os.ErrNotExist)` 判断文件不存在。
