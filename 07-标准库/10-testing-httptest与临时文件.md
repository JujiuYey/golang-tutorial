# 10: testing、httptest 与临时文件

标准库提供了足够强的测试基础设施。即使不用第三方测试框架，也能写出清晰可靠的测试。

---

## 1. 基本测试函数

```go
func TestAdd(t *testing.T) {
    got := Add(1, 2)
    want := 3
    if got != want {
        t.Fatalf("Add(1, 2) = %d, want %d", got, want)
    }
}
```

测试文件以 `_test.go` 结尾，测试函数以 `Test` 开头。

---

## 2. 表格测试

```go
func TestParsePort(t *testing.T) {
    tests := []struct {
        name    string
        input   string
        want    int
        wantErr bool
    }{
        {name: "valid", input: "8080", want: 8080},
        {name: "invalid", input: "abc", wantErr: true},
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

表格测试适合多输入、多边界条件。

---

## 3. 临时目录

```go
func TestWriteFile(t *testing.T) {
    dir := t.TempDir()
    path := filepath.Join(dir, "out.txt")

    if err := WriteFile(path, []byte("hello")); err != nil {
        t.Fatal(err)
    }
}
```

`t.TempDir()` 会自动清理临时目录。测试文件系统代码时优先使用它。

---

## 4. t.Cleanup

```go
resource := openResource()
t.Cleanup(func() {
    resource.Close()
})
```

`t.Cleanup` 会在测试结束时执行。它比手写 defer 更适合和 helper 函数组合。

---

## 5. Helper

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

`t.Helper()` 会让失败位置指向调用 helper 的地方，而不是 helper 内部。

---

## 6. httptest.ResponseRecorder

```go
req := httptest.NewRequest(http.MethodGet, "/health", nil)
rr := httptest.NewRecorder()

handler.ServeHTTP(rr, req)

if rr.Code != http.StatusOK {
    t.Fatalf("status = %d", rr.Code)
}
```

`httptest` 可以不启动真实端口就测试 HTTP handler。HTTP 会在后面章节详细讲。

---

## 7. httptest.Server

```go
server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintln(w, "ok")
}))
defer server.Close()

resp, err := http.Get(server.URL)
if err != nil {
    t.Fatal(err)
}
defer resp.Body.Close()
```

`httptest.Server` 适合测试真实 HTTP client 行为。

---

## 8. 捕获输出

如果函数接收 `io.Writer`，测试输出很容易：

```go
var buf bytes.Buffer

if err := WriteReport(&buf, data); err != nil {
    t.Fatal(err)
}

if got := buf.String(); got != "expected" {
    t.Fatalf("got %q", got)
}
```

这也是为什么函数参数用 `io.Writer` 比直接写 `os.Stdout` 更容易测试。

---

## 9. Benchmark

```go
func BenchmarkJoin(b *testing.B) {
    for i := 0; i < b.N; i++ {
        strings.Join([]string{"a", "b", "c"}, ",")
    }
}
```

运行：

```bash
go test -bench=. ./...
```

基准测试用于比较性能，不要凭感觉判断优化。

---

## 10. 示例测试

```go
func ExampleAdd() {
    fmt.Println(Add(1, 2))
    // Output:
    // 3
}
```

Example 既是文档，也是测试。输出不匹配时测试会失败。

---

## 练习

1. 为 `ParsePort` 写表格测试。
2. 用 `t.TempDir` 测试文件写入函数。
3. 写一个 helper，并加上 `t.Helper()`。
4. 用 `bytes.Buffer` 测试写入 `io.Writer` 的函数。
5. 用 `httptest.NewRecorder` 测试一个简单 handler。
