# 03: path/filepath 与文件系统操作

文件路径和目录操作应该使用标准库处理。不要手写字符串拼接路径。

---

## 1. filepath.Join

```go
path := filepath.Join("configs", "app.json")
```

`filepath.Join` 会按当前操作系统的路径分隔符拼接，并清理多余分隔符。

不要这样写：

```go
path := "configs/" + "app.json"
```

手写分隔符容易造成跨平台问题。

---

## 2. Clean、Dir、Base、Ext

```go
p := "/tmp/app/config.json"

fmt.Println(filepath.Clean("/tmp/app/../app/config.json"))
fmt.Println(filepath.Dir(p))  // /tmp/app
fmt.Println(filepath.Base(p)) // config.json
fmt.Println(filepath.Ext(p))  // .json
```

这些函数适合做路径拆解和规范化。

---

## 3. 绝对路径

```go
abs, err := filepath.Abs("config.json")
if err != nil {
    return err
}
fmt.Println(abs)
```

`Abs` 根据当前工作目录计算绝对路径。当前工作目录可能随启动方式变化，不要在业务逻辑里过度依赖它。

---

## 4. 判断文件是否存在

```go
_, err := os.Stat(path)
if err == nil {
    fmt.Println("exists")
} else if errors.Is(err, os.ErrNotExist) {
    fmt.Println("not exists")
} else {
    return err
}
```

判断特定错误时使用 `errors.Is`。

---

## 5. 创建目录

```go
if err := os.MkdirAll("data/cache", 0755); err != nil {
    return err
}
```

`MkdirAll` 会创建多级目录。如果目录已经存在，通常不会报错。

---

## 6. 读取目录

```go
entries, err := os.ReadDir("data")
if err != nil {
    return err
}

for _, entry := range entries {
    fmt.Println(entry.Name(), entry.IsDir())
}
```

`os.ReadDir` 返回目录项，不会递归。

---

## 7. 遍历目录树

```go
err := filepath.WalkDir(root, func(path string, d fs.DirEntry, err error) error {
    if err != nil {
        return err
    }
    if d.IsDir() {
        return nil
    }
    fmt.Println(path)
    return nil
})
if err != nil {
    return err
}
```

`WalkDir` 适合递归扫描文件树。

---

## 8. Glob

```go
matches, err := filepath.Glob("*.json")
if err != nil {
    return err
}
for _, path := range matches {
    fmt.Println(path)
}
```

`Glob` 根据模式匹配路径。没有匹配项时返回空 slice 和 nil error。

---

## 9. 临时文件和目录

```go
dir, err := os.MkdirTemp("", "app-*")
if err != nil {
    return err
}
defer os.RemoveAll(dir)
```

测试里更推荐使用 `t.TempDir()`，它会在测试结束后自动清理。

```go
dir := t.TempDir()
```

---

## 10. 权限和模式

常见文件权限：

- `0644`: 文件所有者可读写，其他人可读。
- `0600`: 只有所有者可读写。
- `0755`: 目录或可执行文件常见权限。

权限语义会受操作系统和 umask 影响。课程阶段先掌握常见写法即可。

---

## 练习

1. 用 `filepath.Join` 拼接配置文件路径。
2. 写函数 `Exists(path string) (bool, error)`。
3. 写函数列出某个目录下所有 `.json` 文件。
4. 用 `filepath.WalkDir` 统计目录里的普通文件数量。
5. 用 `os.MkdirTemp` 创建临时目录，并在函数结束时清理。
