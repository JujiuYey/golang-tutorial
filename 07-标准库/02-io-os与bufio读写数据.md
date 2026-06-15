# 02: io、os 与 bufio：读写数据

Go 的 IO 体系围绕 `io.Reader` 和 `io.Writer` 展开。文件、网络、内存 buffer、压缩流都可以用统一方式读写。

---

## 1. Reader 和 Writer

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}
```

`Read` 把数据读入 `p`。`Write` 把 `p` 中的数据写出去。

很多函数只依赖这两个小接口：

```go
func Copy(dst io.Writer, src io.Reader) (written int64, err error)
```

---

## 2. 读取整个文件

```go
func readConfig(path string) ([]byte, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        return nil, fmt.Errorf("read config %s: %w", path, err)
    }
    return data, nil
}
```

`os.ReadFile` 会打开、读取、关闭文件。适合小文件，比如配置文件。

大文件不要一次性全部读入内存。

---

## 3. 写入整个文件

```go
func writeConfig(path string, data []byte) error {
    if err := os.WriteFile(path, data, 0644); err != nil {
        return fmt.Errorf("write config %s: %w", path, err)
    }
    return nil
}
```

`0644` 是文件权限：所有者可读写，其他人可读。

---

## 4. 打开文件并关闭

```go
f, err := os.Open("data.txt")
if err != nil {
    return err
}
defer f.Close()
```

只有 `Open` 成功后，才应该 `defer f.Close()`。

如果要写文件：

```go
f, err := os.Create("out.txt")
if err != nil {
    return err
}
defer f.Close()
```

`os.Create` 会创建或截断文件。

---

## 5. io.Copy

复制文件：

```go
func copyFile(dst, src string) error {
    in, err := os.Open(src)
    if err != nil {
        return fmt.Errorf("open source %s: %w", src, err)
    }
    defer in.Close()

    out, err := os.Create(dst)
    if err != nil {
        return fmt.Errorf("create destination %s: %w", dst, err)
    }
    defer out.Close()

    if _, err := io.Copy(out, in); err != nil {
        return fmt.Errorf("copy %s to %s: %w", src, dst, err)
    }
    return nil
}
```

`io.Copy` 不关心两端具体是什么，只要求 `src` 能读、`dst` 能写。

---

## 6. bufio.Scanner 分行读取

```go
func readLines(path string) error {
    f, err := os.Open(path)
    if err != nil {
        return err
    }
    defer f.Close()

    scanner := bufio.NewScanner(f)
    for scanner.Scan() {
        fmt.Println(scanner.Text())
    }
    if err := scanner.Err(); err != nil {
        return err
    }
    return nil
}
```

`Scanner` 默认按行扫描。循环结束后必须检查 `scanner.Err()`。

默认 token 大小有限。超长行要调整 buffer，或者改用 `bufio.Reader`。

---

## 7. bufio.Reader

```go
reader := bufio.NewReader(f)

for {
    line, err := reader.ReadString('\n')
    if len(line) > 0 {
        fmt.Print(line)
    }
    if err == io.EOF {
        break
    }
    if err != nil {
        return err
    }
}
```

`ReadString('\n')` 读到分隔符或遇到错误。遇到 EOF 时，可能仍然返回最后一段数据。

---

## 8. bufio.Writer

```go
f, err := os.Create("out.txt")
if err != nil {
    return err
}
defer f.Close()

writer := bufio.NewWriter(f)
if _, err := writer.WriteString("hello\n"); err != nil {
    return err
}
if err := writer.Flush(); err != nil {
    return err
}
```

使用缓冲写入后，必须 `Flush`，否则数据可能还留在内存缓冲区。

---

## 9. strings.Reader 和 bytes.Buffer

用字符串模拟 Reader：

```go
r := strings.NewReader("hello")
data, err := io.ReadAll(r)
```

用 buffer 收集输出：

```go
var buf bytes.Buffer
if _, err := buf.WriteString("hello"); err != nil {
    return err
}
fmt.Println(buf.String())
```

这些类型很适合测试，因为可以不用真实文件。

---

## 10. io.ReadAll

```go
data, err := io.ReadAll(r)
if err != nil {
    return err
}
```

`io.ReadAll` 会从 Reader 读到 EOF。适合小数据，不适合无限流或很大的输入。

---

## 练习

1. 写函数 `CopyFile(dst, src string) error`，使用 `io.Copy`。
2. 写函数 `ReadLines(path string) ([]string, error)`，使用 `bufio.Scanner`。
3. 写函数 `WriteLines(path string, lines []string) error`，使用 `bufio.Writer` 并检查 `Flush` 错误。
4. 用 `strings.NewReader` 测试一个接收 `io.Reader` 的函数。
5. 故意读取不存在的文件，用 `%w` 包装错误。
