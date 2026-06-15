# 07: 标准库

Go 标准库覆盖了文件、IO、编码、时间、命令行、日志、测试、路径、错误处理等常见任务。本模块聚焦日常写 Go 程序最常用、最容易写错、最值得优先掌握的部分。

---

## 文章列表

1. [标准库学习方法与文档阅读](./01-标准库学习方法与文档阅读.md)
2. [io、os 与 bufio：读写数据](./02-io-os与bufio读写数据.md)
3. [path/filepath 与文件系统操作](./03-filepath与文件系统操作.md)
4. [strings、strconv 与 bytes](./04-strings-strconv与bytes.md)
5. [encoding/json：结构体、标签与流式处理](./05-json结构体标签与流式处理.md)
6. [time：时间、时区、定时器与 ticker](./06-time时间时区定时器.md)
7. [flag 与 os.Args：命令行参数](./07-flag与命令行参数.md)
8. [log、slog 与结构化日志](./08-log-slog与结构化日志.md)
9. [errors、sort、slices、maps 与常用辅助包](./09-errors-sort-slices-maps.md)
10. [testing、httptest 与临时文件](./10-testing-httptest与临时文件.md)
11. [标准库综合练习](./11-标准库综合练习.md)

---

## 学习目标

完成本模块后，你应该能够：

- 知道如何阅读标准库文档和示例。
- 正确使用 `io.Reader`、`io.Writer`、`os.File`、`bufio.Scanner`、`io.Copy`。
- 处理文件路径、目录遍历、临时文件和跨平台路径拼接。
- 熟练使用 `strings`、`strconv`、`bytes` 做文本和字节处理。
- 使用 `encoding/json` 编码、解码、设置 tag、处理未知字段和流式数据。
- 使用 `time.Time`、`time.Duration`、时区、timer、ticker。
- 使用 `flag` 写简单 CLI 参数解析。
- 使用 `log/slog` 写结构化日志。
- 使用 `errors.Is`、`errors.As`、`sort`、`slices`、`maps` 等常用辅助包。
- 在测试里使用 `testing`、`httptest`、`t.TempDir`。

---

## 学习建议

标准库的学习方式和语法不同。语法要建立规则，标准库要建立“什么时候该用哪个包”的索引。

建议每学一个包都做三件事：

1. 读包级文档，理解它解决什么问题。
2. 读常用函数签名，看参数和返回值如何表达边界。
3. 写一个小例子，主动处理错误和资源关闭。

---

## 下一步

完成本模块后，继续学习：

- **08**: 网络编程
