# 11: 文件持久化：JSON、CSV 与原子写入

不是所有持久化都需要数据库。小工具、配置、缓存、导入导出任务，常常用 JSON、CSV 或普通文件。

---

## 1. JSON 文件

```go
func SaveJSON(path string, v any) error {
    data, err := json.MarshalIndent(v, "", "  ")
    if err != nil {
        return fmt.Errorf("marshal json: %w", err)
    }
    if err := os.WriteFile(path, data, 0644); err != nil {
        return fmt.Errorf("write json %s: %w", path, err)
    }
    return nil
}
```

读取：

```go
func LoadJSON(path string, v any) error {
    data, err := os.ReadFile(path)
    if err != nil {
        return fmt.Errorf("read json %s: %w", path, err)
    }
    if err := json.Unmarshal(data, v); err != nil {
        return fmt.Errorf("decode json %s: %w", path, err)
    }
    return nil
}
```

---

## 2. CSV 文件

```go
func WriteCSV(path string, rows [][]string) error {
    f, err := os.Create(path)
    if err != nil {
        return err
    }
    defer f.Close()

    w := csv.NewWriter(f)
    if err := w.WriteAll(rows); err != nil {
        return err
    }
    if err := w.Error(); err != nil {
        return err
    }
    return nil
}
```

`csv.Writer` 内部有缓冲，写完要检查错误。

---

## 3. 读取 CSV

```go
func ReadCSV(path string) ([][]string, error) {
    f, err := os.Open(path)
    if err != nil {
        return nil, err
    }
    defer f.Close()

    r := csv.NewReader(f)
    rows, err := r.ReadAll()
    if err != nil {
        return nil, err
    }
    return rows, nil
}
```

大文件可以循环 `Read`，避免一次性读入全部。

---

## 4. 原子写入

直接写目标文件时，如果程序中途崩溃，文件可能只写了一半。

更稳的方式：

1. 写入同目录临时文件。
2. fsync 或关闭文件。
3. rename 到目标路径。

```go
func AtomicWriteFile(path string, data []byte, perm fs.FileMode) error {
    dir := filepath.Dir(path)
    tmp, err := os.CreateTemp(dir, ".tmp-*")
    if err != nil {
        return err
    }
    tmpName := tmp.Name()
    defer os.Remove(tmpName)

    if _, err := tmp.Write(data); err != nil {
        tmp.Close()
        return err
    }
    if err := tmp.Close(); err != nil {
        return err
    }
    if err := os.Chmod(tmpName, perm); err != nil {
        return err
    }
    if err := os.Rename(tmpName, path); err != nil {
        return err
    }
    return nil
}
```

原子 rename 在同一文件系统内更可靠。

---

## 5. 文件锁

标准库没有跨平台文件锁的高级封装。多个进程同时写同一个文件时，需要额外设计：

- 单进程拥有写权限。
- 使用数据库。
- 使用平台文件锁。
- 使用临时文件和 rename 降低损坏风险。

简单工具通常避免并发写同一个文件。

---

## 6. 配置文件

JSON 配置适合结构简单的场景：

```go
type Config struct {
    Port int    `json:"port"`
    DSN  string `json:"dsn"`
}
```

配置文件里不要保存明文生产密码。敏感信息通常从环境变量或密钥系统读取。

---

## 7. 导入导出

CSV 适合表格数据导入导出。JSON 适合结构化嵌套数据。

选择格式时看：

- 是否给人编辑。
- 是否要表达嵌套结构。
- 是否要和表格软件交换。
- 文件大小和流式处理需求。

---

## 8. 文件路径

持久化文件路径要明确：

- 用户传入路径。
- 配置目录。
- 数据目录。
- 临时目录。

使用 `filepath.Join`，不要手写路径分隔符。

---

## 9. 错误包装

文件持久化错误要带路径：

```go
return fmt.Errorf("save config %s: %w", path, err)
```

路径能极大提高排查效率。

---

## 10. 文件持久化检查

1. 写入是否处理 error。
2. 文件是否关闭。
3. 是否需要原子写入。
4. 路径是否安全。
5. 测试是否使用 `t.TempDir()`。

---

## 练习

1. 实现 `SaveJSON` 和 `LoadJSON`。
2. 实现 `WriteCSV` 和 `ReadCSV`。
3. 实现 `AtomicWriteFile`。
4. 用 `t.TempDir()` 测试 JSON 文件保存。
5. 比较 JSON 和 CSV 适合的场景。
