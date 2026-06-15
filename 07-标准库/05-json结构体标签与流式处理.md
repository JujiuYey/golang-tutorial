# 05: encoding/json：结构体、标签与流式处理

`encoding/json` 是 Go 中处理 JSON 的标准库包。它通过导出字段、struct tag 和 `Marshal`、`Unmarshal` 完成编码解码。

---

## 1. 编码结构体

```go
type User struct {
    Name    string   `json:"name"`
    Age     int      `json:"age"`
    Email   string   `json:"email,omitempty"`
    Hobbies []string `json:"hobbies"`
}

u := User{
    Name:    "alice",
    Age:     18,
    Hobbies: []string{"go", "reading"},
}

data, err := json.Marshal(u)
if err != nil {
    return err
}
fmt.Println(string(data))
```

只有导出字段会被编码。字段名必须首字母大写，比如 `Hobbies`。未导出字段即使写了 tag，也不会被 `encoding/json` 访问。

---

## 2. 格式化输出

```go
data, err := json.MarshalIndent(u, "", "  ")
if err != nil {
    return err
}
fmt.Println(string(data))
```

`MarshalIndent` 适合配置文件、调试输出、命令行展示。

---

## 3. 解码 JSON

```go
input := []byte(`{"name":"alice","age":18,"hobbies":["go"]}`)

var u User
if err := json.Unmarshal(input, &u); err != nil {
    return err
}
fmt.Printf("%+v\n", u)
```

`Unmarshal` 需要传指针，因为它要修改目标变量。

---

## 4. struct tag

```go
type User struct {
    ID       int    `json:"id"`
    Name     string `json:"name"`
    Password string `json:"-"`
    Email    string `json:"email,omitempty"`
}
```

常见 tag：

- `json:"name"`: 使用指定字段名。
- `json:"-"`: 忽略字段。
- `json:"email,omitempty"`: 零值时省略字段。

`omitempty` 判断的是 Go 零值，比如空字符串、0、false、nil slice、nil map。

---

## 5. nil slice 和空 slice

```go
type Response struct {
    Items []string `json:"items"`
}

a := Response{Items: nil}
b := Response{Items: []string{}}
```

通常编码结果：

- nil slice: `"items":null`
- 空 slice: `"items":[]`

对外 API 经常更希望返回空数组。需要时主动初始化为空 slice。

---

## 6. map[string]any

```go
var m map[string]any
if err := json.Unmarshal(data, &m); err != nil {
    return err
}
```

解到 `map[string]any` 后：

- JSON object -> `map[string]any`
- JSON array -> `[]any`
- JSON string -> `string`
- JSON number -> `float64`
- JSON boolean -> `bool`
- JSON null -> `nil`

这很灵活，但类型信息会变弱。结构明确时优先用 struct。

---

## 7. Decoder 和 Encoder

流式解码：

```go
dec := json.NewDecoder(r)
var u User
if err := dec.Decode(&u); err != nil {
    return err
}
```

流式编码：

```go
enc := json.NewEncoder(w)
if err := enc.Encode(u); err != nil {
    return err
}
```

`Encoder.Encode` 会在输出末尾追加换行。写 HTTP 响应和日志时很常见。

---

## 8. 禁止未知字段

```go
dec := json.NewDecoder(r)
dec.DisallowUnknownFields()

var req Request
if err := dec.Decode(&req); err != nil {
    return err
}
```

这适合处理外部请求，防止调用方传了拼错字段却悄悄被忽略。

---

## 9. 延迟解析 RawMessage

```go
type Event struct {
    Type string          `json:"type"`
    Data json.RawMessage `json:"data"`
}
```

`RawMessage` 可以先保留一段原始 JSON，根据 `Type` 再决定解成什么结构。

---

## 10. 自定义 JSON

类型可以实现：

```go
MarshalJSON() ([]byte, error)
UnmarshalJSON([]byte) error
```

这适合特殊格式，比如把时间编码成自定义字符串。

先掌握默认规则。只有默认规则不符合需求时，再写自定义方法。

---

## 练习

1. 定义 `Config` struct，编码成缩进 JSON。
2. 从 JSON 字符串解码到 `Config`。
3. 写一个包含 `omitempty` 和 `json:"-"` 的结构体。
4. 观察 nil slice 和空 slice 的 JSON 输出差异。
5. 用 `json.Decoder` 解码，并开启 `DisallowUnknownFields`。
