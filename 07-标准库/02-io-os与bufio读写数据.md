# 02: io、os 与 bufio:读写数据

这篇解决一个日常刚需:在 Go 里读写文件——小文件一口气读完、大文件一行行流着读、写完怎么保证数据真的落盘。学完你能躲开本篇最大的坑:**用了缓冲写却忘了 Flush,文件是空的**。

给 Node 用户校准坐标:`os.ReadFile` ≈ `fs.readFileSync`,`bufio.Scanner` ≈ `readline`,而贯穿一切的 `io.Reader` / `io.Writer` ≈ Node 的 stream——只是 Go 的版本小得多,各只有一个方法。

---

## 1. Reader 和 Writer:两个"通用插座"

Go 的整个 IO 体系就建立在两个单方法接口上:

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}
```

大白话:`Read` 是"往我递来的桶 `p` 里装数据,告诉我装了几个字节";`Write` 是"把 `p` 里的数据拿去写掉"。文件、网络连接、内存 buffer、压缩流,只要实现了这两个方法之一,就能插进所有认这个"插座"的函数,比如:

```go
func Copy(dst io.Writer, src io.Reader) (written int64, err error)
```

对比 JS:Node 的 Readable/Writable stream 是同样的思想,但那是带十几个方法和事件的大类;Go 靠第 05 章的隐式接口,把门槛降到**一个方法**。

**一句话总结:本篇后面所有 API,要么在生产 Reader/Writer,要么在消费它们。**

---

## 2. 一次读整个文件:os.ReadFile

```go
func main() {
    data, err := os.ReadFile("config.txt")
    if err != nil {
        fmt.Println(err)
        return
    }
    fmt.Print(string(data))
}
```

```text
port=8080
debug=true
```

`os.ReadFile` 一个调用干完打开、读取、关闭三件事,返回 `[]byte`(所以打印前用 `string(data)` 转一下——02-05 的老朋友)。

实际项目里通常包一层错误上下文:

```go
func readConfig(path string) ([]byte, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        return nil, fmt.Errorf("read config %s: %w", path, err)
    }
    return data, nil
}
```

```text
read config missing.json: open missing.json: no such file or directory
```

(上面是读取不存在文件时,包装后的错误打印出来的样子——两层信息都在。)

适合小文件、配置文件。**大文件不要这样一次性读入内存**,用第 6 节的流式读法。

---

## 3. 一次写整个文件:os.WriteFile

```go
func main() {
    err := os.WriteFile("out.txt", []byte("hello file\n"), 0644)
    if err != nil {
        fmt.Println(err)
        return
    }

    data, _ := os.ReadFile("out.txt")
    fmt.Print(string(data))
}
```

```text
hello file
```

第三个参数 `0644` 是 Unix 文件权限,拆开看:

```text
0 6 4 4
│ │ │ └─ 其他人:可读(4)
│ │ └─── 同组:可读(4)
│ └───── 所有者:可读+可写(4+2=6)
└─────── 八进制前缀
```

现阶段记两个常用值就够:文件用 `0644`,目录用 `0755`(下一篇还会遇到)。

---

## 4. 打开文件:os.Open 与 os.Create

需要更细的控制(流式读、边处理边写)时,先拿到文件句柄:

```go
f, err := os.Open("data.txt")   // 只读打开
if err != nil {
    return err
}
defer f.Close()
```

写文件用:

```go
f, err := os.Create("out.txt")  // 创建,已存在则清空
if err != nil {
    return err
}
defer f.Close()
```

两个提醒:

- defer 的位置是 03-07 的老坑:**`Open` 成功之后才 `defer f.Close()`**,失败了没东西可关。
- 🕳️ 坑:`os.Create` 对已存在的文件是**截断**(truncate)——旧内容直接清空,不是追加。想追加要用 `os.OpenFile` 加 `os.O_APPEND` 标志。

---

## 5. io.Copy:两头一接,中间不管

复制文件不需要自己搬字节,把一个 Reader 和一个 Writer 接起来就行:

```go
func copyFile(dst, src string) error {
    in, err := os.Open(src)
    if err != nil {
        return fmt.Errorf("open %s: %w", src, err)
    }
    defer in.Close()

    out, err := os.Create(dst)
    if err != nil {
        return fmt.Errorf("create %s: %w", dst, err)
    }
    defer out.Close()

    n, err := io.Copy(out, in)
    if err != nil {
        return fmt.Errorf("copy %s to %s: %w", src, dst, err)
    }
    fmt.Println("copied bytes:", n)
    return nil
}
```

```text
copied bytes: 21
```

`io.Copy(dst, src)` 只要求"src 能读、dst 能写"——换成网络连接、HTTP 响应体照样能用,这就是第 1 节"通用插座"的威力。注意参数顺序:**目标在前,来源在后**(和赋值语句 `dst = src` 同向)。

---

## 6. bufio.Scanner:按行流式读

大文件、日志、持续输入,用 Scanner 一行一行过:

```go
func main() {
    f, err := os.Open("lines.txt")
    if err != nil {
        fmt.Println(err)
        return
    }
    defer f.Close()

    scanner := bufio.NewScanner(f)
    n := 0
    for scanner.Scan() {
        n++
        fmt.Printf("第 %d 行: %s\n", n, scanner.Text())
    }
    if err := scanner.Err(); err != nil {
        fmt.Println("scan:", err)
    }
}
```

```text
第 1 行: first line
第 2 行: second line
第 3 行: third line
```

套路固定:`Scan()` 返回 false 就结束,`Text()` 取当前行(不含换行符)。

🕳️ 坑:循环结束后**必须查 `scanner.Err()`**。`Scan()` 返回 false 有两种原因——读完了(正常)或者读出错了(异常),循环本身分不出来。不查 Err,读到一半断掉你都不知道。

🕳️ 坑:Scanner 默认单行上限 64KB,超长行会报 `token too long`。处理可能有超长行的数据,要用 `scanner.Buffer` 调大上限,或改用下一节的 `bufio.Reader`。

---

## 7. bufio.Reader:按分隔符读

比 Scanner 更底层、没有单行上限的读法:

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

```text
first line
second line
third line
```

注意循环体的顺序:**先处理数据,再判断 EOF**。因为最后一段数据可能和 `io.EOF` 一起返回(文件末尾没有换行符时)——先 break 后处理,最后一行就丢了。

日常按行读优先用 Scanner(API 更省心),Reader 留给超长行和自定义分隔符的场景。

---

## 8. bufio.Writer:缓冲写,必须 Flush

频繁的小段写入,直接写文件每次都是一次系统调用,慢。`bufio.Writer` 先攒在内存里,攒够了一次写出——但这带来本篇最大的坑。

### 🕳️ 坑:忘了 Flush,文件是空的

以为会怎样:`WriteString` 都成功返回了,数据肯定在文件里。
实际怎样:文件 0 字节。
为什么:数据还躺在内存缓冲区里,从没到过文件。

```go
func main() {
    f, _ := os.Create("buffered.txt")
    writer := bufio.NewWriter(f)
    writer.WriteString("hello\n")
    f.Close() // 没 Flush 就关了

    data, _ := os.ReadFile("buffered.txt")
    fmt.Printf("文件内容: %q(%d 字节)\n", string(data), len(data))
}
```

```text
文件内容: ""(0 字节)
```

加上 `Flush`(把缓冲区倒进文件)才对:

```go
    writer := bufio.NewWriter(f)
    writer.WriteString("hello\n")
    writer.Flush() // 把缓冲区倒进文件
    f.Close()
```

```text
文件内容: "hello\n"(6 字节)
```

**一句话总结:用了 bufio.Writer,写完必须 `Flush`,而且要检查 `Flush` 的返回错误——它才是数据真正落盘的时刻。**

---

## 9. strings.Reader 和 bytes.Buffer:测试神器

想测一个收 `io.Reader` 的函数,不用真建文件——字符串就能伪装成 Reader:

```go
func main() {
    r := strings.NewReader("hello")
    data, _ := io.ReadAll(r)
    fmt.Println(string(data))

    var buf bytes.Buffer
    buf.WriteString("hello ")
    buf.WriteString("go")
    fmt.Println(buf.String())
}
```

```text
hello
hello go
```

- `strings.NewReader(s)`:把字符串变成 Reader,当"假输入"。
- `bytes.Buffer`:同时实现 Reader **和** Writer,当"假输出",写进去的东西随时用 `buf.String()` 检查。

这是第 10 篇写测试的基础:**函数收 `io.Reader` / `io.Writer` 而不是写死文件名,测试就不需要碰真实文件系统。**

---

## 10. io.ReadAll:把 Reader 喝干

```go
func main() {
    r := strings.NewReader("read me to EOF")
    data, err := io.ReadAll(r)
    if err != nil {
        fmt.Println(err)
        return
    }
    fmt.Println(string(data), len(data))
}
```

```text
read me to EOF 14
```

`io.ReadAll` 从任意 Reader 一直读到 EOF,全部装进内存。和 `os.ReadFile` 的关系:后者 = 打开文件 + `ReadAll` + 关闭。同样的警告也适用:**小数据可以,无限流和超大输入不行**——它会把整条流都装进内存。

---

## 本篇重点

- [ ] `io.Reader` / `io.Writer` 是 Go IO 的通用插座:文件、网络、内存 buffer 都插得进,`io.Copy(dst, src)` 两头一接就能搬数据。
- [ ] 小文件一次性:`os.ReadFile` / `os.WriteFile`(权限记 `0644`);大文件流式:`bufio.Scanner` 按行读,循环后必查 `scanner.Err()`。
- [ ] `os.Create` 会清空已存在的文件;`defer f.Close()` 写在错误检查之后(03-07 的规矩)。
- [ ] 大坑:`bufio.Writer` 写完必须 `Flush`,否则数据留在内存缓冲区,文件是空的。
- [ ] 函数参数收 `io.Reader`/`io.Writer` 而不是文件名,就能用 `strings.NewReader` / `bytes.Buffer` 做无文件测试。

---

## 练习

1. 写函数 `CopyFile(dst, src string) error`,使用 `io.Copy`(提示:注意 dst 在前;两个 defer 的位置各自紧跟错误检查)。
2. 写函数 `ReadLines(path string) ([]string, error)`,使用 `bufio.Scanner`(提示:别忘了循环后的 `scanner.Err()`)。
3. 写函数 `WriteLines(path string, lines []string) error`,使用 `bufio.Writer` 并检查 `Flush` 错误(提示:第 8 节的坑,Flush 的 error 不能丢)。
4. 用 `strings.NewReader` 测试一个接收 `io.Reader` 的函数(提示:参考第 9 节,函数签名里不要出现文件名)。
5. 故意读取不存在的文件,用 `%w` 包装错误并打印,对比第 2 节的输出格式。
