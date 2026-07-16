# 10: testing、httptest 与临时文件

这篇解决"代码写完怎么证明它对":Go 不需要装 jest,测试框架就在标准库里——`go test` 一条命令,测试、HTTP 模拟、基准测试、文档示例全包。学完你能写出 Go 社区最常见的测试形态:**表格测试**。

对 JS 用户:没有 `describe/it/expect`,断言就是普通的 `if` + 报错;像不像 jest 不重要,**能抓住回归**才重要。

---

## 1. 基本测试:一个函数一个 if

约定先行:测试文件叫 `xxx_test.go`,测试函数叫 `TestXxx`,收一个 `*testing.T`:

```go
// calc.go
func Add(a, b int) int {
    return a + b
}

// calc_test.go
func TestAdd(t *testing.T) {
    got := Add(1, 2)
    want := 3
    if got != want {
        t.Fatalf("Add(1, 2) = %d, want %d", got, want)
    }
}
```

```bash
$ go test -v ./calc
=== RUN   TestAdd
--- PASS: TestAdd (0.00s)
PASS
ok  	mod07/calc	0.447s
```

没有断言库:自己比较,不对就 `t.Fatalf`(报错并停止本测试)或 `t.Errorf`(报错但继续)。失败时长这样(把 `Add` 故意改错后的真实输出):

```bash
$ go test ./faildemo
--- FAIL: TestAdd (0.00s)
    calc_test.go:9: Add(1, 2) = -1, want 3
FAIL
FAIL	mod07/faildemo	0.421s
```

所以报错信息的固定句式值得学:`得到什么 = 实际值, want 期望值`——失败时一眼定位。

---

## 2. 表格测试:Go 测试的招牌写法

多组输入不用复制粘贴测试函数,把用例摆成一张表,循环跑:

```go
func TestParsePort(t *testing.T) {
    tests := []struct {
        name    string
        input   string
        want    int
        wantErr bool
    }{
        {name: "valid", input: "8080", want: 8080},
        {name: "not a number", input: "abc", wantErr: true},
        {name: "too big", input: "70000", wantErr: true},
        {name: "zero", input: "0", wantErr: true},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := ParsePort(tt.input)
            if (err != nil) != tt.wantErr {
                t.Fatalf("err = %v, wantErr %v", err, tt.wantErr)
            }
            if got != tt.want {
                t.Fatalf("got %d, want %d", got, tt.want)
            }
        })
    }
}
```

```bash
$ go test -v ./port
=== RUN   TestParsePort
=== RUN   TestParsePort/valid
=== RUN   TestParsePort/not_a_number
=== RUN   TestParsePort/too_big
=== RUN   TestParsePort/zero
--- PASS: TestParsePort (0.00s)
    --- PASS: TestParsePort/valid (0.00s)
    --- PASS: TestParsePort/not_a_number (0.00s)
    --- PASS: TestParsePort/too_big (0.00s)
    --- PASS: TestParsePort/zero (0.00s)
PASS
ok  	mod07/port	0.418s
```

结构拆解:匿名 struct 切片(04 章)当用例表 + `t.Run(名字, 子测试)` 让每个用例独立显示、独立失败。加一个边界用例 = 加一行,这就是它成为惯用法的原因。JS 对应物:`it.each`。

`(err != nil) != tt.wantErr` 这行拆开:左边是"实际出错没",右边是"该不该出错",两个 bool 不相等 = 结果不符合预期。

---

## 3. 临时目录:t.TempDir

测文件操作,不要写进真实目录,用 `t.TempDir()`:

```go
func TestWriteFile(t *testing.T) {
    dir := t.TempDir()
    path := filepath.Join(dir, "out.txt")

    if err := WriteFile(path, []byte("hello")); err != nil {
        t.Fatal(err)
    }
}
```

比第 03 篇的 `os.MkdirTemp` 更省:**测试结束自动删除**,连 defer 都不用。每个测试拿到的目录互不相同,天然隔离、可并行。

---

## 4. t.Cleanup:测试版的 defer

```go
resource := openResource()
t.Cleanup(func() {
    resource.Close()
})
```

和 defer 的差别:defer 在**当前函数**退出时执行,`t.Cleanup` 在**整个测试**(包括所有子测试)结束时执行。所以在 helper 函数里创建资源时必须用 Cleanup——helper 返回时资源还得活着,defer 会关早了。`t.TempDir` 内部正是用它实现自动清理的。

---

## 5. 测试助手:t.Helper

重复的准备代码抽成 helper,记得开头加一行 `t.Helper()`:

```go
func mustReadFile(t *testing.T, path string) []byte {
    t.Helper()
    data, err := os.ReadFile(path)
    if err != nil {
        t.Fatal(err)
    }
    return data
}
```

`t.Helper()` 的作用:helper 里测试失败时,**报错行号指向调用它的那行**,而不是 helper 内部——不然每次失败都显示 `mustReadFile` 第 5 行,等于没说。命名惯例:`must` 前缀表示"失败就直接终止测试"。

---

## 6. 测 HTTP handler:httptest.NewRecorder

不启动服务器、不占端口,就能测一个 handler:

```go
func TestHealth(t *testing.T) {
    req := httptest.NewRequest(http.MethodGet, "/health", nil)
    rr := httptest.NewRecorder()

    http.HandlerFunc(Health).ServeHTTP(rr, req)

    if rr.Code != http.StatusOK {
        t.Fatalf("status = %d", rr.Code)
    }
    if got := strings.TrimSpace(rr.Body.String()); got != `{"status":"ok"}` {
        t.Fatalf("body = %q", got)
    }
}
```

思路:`NewRequest` 造一个假请求,`NewRecorder` 造一个"录音机"当 ResponseWriter,handler 往里写,写完从 `rr.Code`、`rr.Body` 里验货。HTTP 本身下一章才展开,这里先记住**测 handler 不需要真端口**。

---

## 7. 测 HTTP client:httptest.NewServer

反过来,想测"发请求的代码",起一个真实但随机端口的本地服务器:

```go
func TestHealthServer(t *testing.T) {
    server := httptest.NewServer(http.HandlerFunc(Health))
    defer server.Close()

    resp, err := http.Get(server.URL)
    if err != nil {
        t.Fatal(err)
    }
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusOK {
        t.Fatalf("status = %d", resp.StatusCode)
    }
}
```

`server.URL` 是随机分配的本地地址(形如 `http://127.0.0.1:随机端口`)。选择规则:**测 handler 用 Recorder(快),测 client/中间件全链路用 Server(真)。** 别忘了 `defer server.Close()`。

---

## 8. 捕获输出:又是 io.Writer 的回报

第 02 篇的伏笔在这兑现:函数收 `io.Writer` 而不是直接打印,测试就只是换个"假输出":

```go
func TestWriteReport(t *testing.T) {
    var buf bytes.Buffer

    if err := WriteReport(&buf, 42); err != nil {
        t.Fatal(err)
    }

    if got := buf.String(); got != "words: 42\n" {
        t.Fatalf("got %q", got)
    }
}
```

如果 `WriteReport` 内部写死 `os.Stdout`,这个测试根本没法写。**"参数收接口"不是风格洁癖,是可测试性的入场券。**

上面 3、5、8 节的测试跑起来:

```bash
$ go test -v ./files
=== RUN   TestWriteFile
--- PASS: TestWriteFile (0.00s)
=== RUN   TestWriteReport
--- PASS: TestWriteReport (0.00s)
PASS
ok  	mod07/files	0.450s
```

---

## 9. 基准测试:Benchmark

性能争论别靠感觉,靠数据:

```go
func BenchmarkJoin(b *testing.B) {
    for i := 0; i < b.N; i++ {
        strings.Join([]string{"a", "b", "c"}, ",")
    }
}
```

```bash
$ go test -bench=Join -benchmem ./calc
goos: darwin
goarch: arm64
pkg: mod07/calc
cpu: Apple M4
BenchmarkJoin-10    	54171157	        22.12 ns/op	       8 B/op	       1 allocs/op
```

(具体数字取决于机器,你的输出会不同。)

读法:跑了 5400 万次,平均每次 22 纳秒、分配 8 字节内存、1 次分配。`b.N` 由框架自动调到测量足够稳定为止——你只管把被测代码放进循环。

---

## 10. 示例测试:Example

```go
func ExampleAdd() {
    fmt.Println(Add(1, 2))
    // Output:
    // 3
}
```

`// Output:` 注释下面写期望输出,`go test` 会真的运行函数并比对——输出不匹配算测试失败。它同时也是文档:pkg.go.dev 会把 Example 展示在函数文档旁边(第 01 篇你运行过的那些示例就是这么来的)。**永不过期的示例代码**,一石二鸟。

---

## 本篇重点

- [ ] 约定:`_test.go` 文件 + `TestXxx(t *testing.T)`;没有断言库,`if` + `t.Fatalf("got %v, want %v", ...)` 就是全部。
- [ ] 表格测试是 Go 惯用法:匿名 struct 切片摆用例,`t.Run` 跑子测试,加用例只加一行。
- [ ] 文件测试用 `t.TempDir()`(自动清理);helper 里用 `t.Cleanup` 不用 defer,并加 `t.Helper()` 让报错指向调用处。
- [ ] 测 handler 用 `httptest.NewRecorder`(无端口),测 client 用 `httptest.NewServer`(真请求,记得 Close)。
- [ ] 函数收 `io.Writer` 才能用 `bytes.Buffer` 验输出;性能判断跑 `-bench`,别猜。

---

## 练习

1. 为 `ParsePort`(第 04 篇练习 1 写过)写表格测试,覆盖合法、非数字、越界三类用例(提示:照第 2 节的骨架,先把表填出来)。
2. 用 `t.TempDir` 测试一个文件写入函数(提示:路径用 `filepath.Join(dir, ...)` 拼)。
3. 写一个 `mustWriteFile` helper 并加上 `t.Helper()`(提示:对比加与不加时,失败报错的行号指向哪里)。
4. 用 `bytes.Buffer` 测试一个写入 `io.Writer` 的函数(提示:`&buf` 才实现了接口,想想为什么要取地址)。
5. 用 `httptest.NewRecorder` 测试一个返回 JSON 的简单 handler,断言状态码和响应体(提示:响应体末尾可能有换行,`strings.TrimSpace` 处理)。
