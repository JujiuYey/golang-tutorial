# 05: encoding/json:结构体、标签与流式处理

这篇解决"JS 里两行的事,Go 里怎么写":`JSON.stringify` → `json.Marshal`,`JSON.parse` → `json.Unmarshal`。核心差异一句话——**JS 解析出来是随便什么形状的对象,Go 要求你先用 struct 声明好形状**,而 struct 和 JSON 字段名之间的桥梁,就是本篇的主角:struct tag。

---

## 1. 编码:json.Marshal ≈ JSON.stringify

```go
type User struct {
    Name    string   `json:"name"`
    Age     int      `json:"age"`
    Email   string   `json:"email,omitempty"`
    Hobbies []string `json:"hobbies"`
}

func main() {
    u := User{Name: "alice", Age: 18, Hobbies: []string{"go", "reading"}}

    data, err := json.Marshal(u)
    if err != nil {
        fmt.Println(err)
        return
    }
    fmt.Println(string(data))
}
```

```text
{"name":"alice","age":18,"hobbies":["go","reading"]}
```

三个和 JS 不同的地方:

- `Marshal` 返回 `[]byte` 不是 string(打印前转一下)。
- 它可能失败(比如值里有函数、channel),所以有 error。
- 字段名后面那串反引号字符串是 **struct tag**——第 4 节专门拆它。

### 🕳️ 坑:小写字段会被静默跳过

以为会怎样:字段写了 tag 就会被编码。
实际怎样:小写开头(未导出)的字段**直接消失,不报错**。

```go
type user struct {
    Name string `json:"name"`
    age  int    `json:"age"` // 小写开头:未导出
}

func main() {
    data, err := json.Marshal(user{Name: "alice", age: 18})
    fmt.Println(string(data), err)
}
```

```text
{"name":"alice"} <nil>
```

为什么:`encoding/json` 是外部包,和你写的任何包一样,**只能访问导出(大写开头)的字段**——04 章的可见性规则在这里变成了运行时的"字段失踪"。JSON 字段莫名缺失时,第一件事就是检查大小写。

---

## 2. 格式化输出:MarshalIndent

给人看的 JSON(配置文件、调试输出)用缩进版,对应 `JSON.stringify(v, null, 2)`:

```go
data, err := json.MarshalIndent(u, "", "  ")
if err != nil {
    return err
}
fmt.Println(string(data))
```

```text
{
  "name": "alice",
  "age": 18,
  "hobbies": [
    "go",
    "reading"
  ]
}
```

第二个参数是每行前缀(一般传 `""`),第三个是缩进单位(两个空格)。

---

## 3. 解码:json.Unmarshal ≈ JSON.parse

```go
func main() {
    input := []byte(`{"name":"alice","age":18,"hobbies":["go"]}`)

    var u User
    if err := json.Unmarshal(input, &u); err != nil {
        fmt.Println(err)
        return
    }
    fmt.Printf("%+v\n", u)
}
```

```text
{Name:alice Age:18 Email: Hobbies:[go]}
```

和 `JSON.parse` 的本质区别:**结果不是"任意对象",而是填进你声明好的 struct**——字段类型对不上会报错,多出来的字段被忽略,缺的字段保持零值(`Email` 就是空串)。类型安全从解析那一刻就开始了。

🕳️ 坑:第二个参数必须传指针。忘了 `&` 不是编译错误,而是运行时 error:

```go
err := json.Unmarshal(input, u) // 忘了 &
fmt.Println(err)
```

```text
json: Unmarshal(non-pointer main.User)
```

为什么要指针:`Unmarshal` 要**往你的变量里写数据**,04 章讲过——想让函数修改你的变量,就得把地址给它。

---

## 4. struct tag:字段旁边的"小纸条"

JS 里没有对应物,重点拆解。tag 是写在字段后面的反引号字符串,**给 `encoding/json` 这类包看的元信息**,告诉它"这个字段在 JSON 里叫什么、怎么处理":

```text
Email  string  `json:"email,omitempty"`
//     ──┬───  ─┬────  ──┬──  ───┬─────
//    字段类型   ①给哪个包看  ②JSON里的名字  ③附加选项
```

为什么需要它:Go 字段必须大写开头才能导出(第 1 节的坑),但 JSON 世界的惯例是小写/下划线——tag 就是两边命名法的翻译层。

三种最常用的写法:

```go
type Account struct {
    ID       int    `json:"id"`              // 改名:ID -> id
    Name     string `json:"name"`
    Password string `json:"-"`               // 永不编码(敏感字段)
    Email    string `json:"email,omitempty"` // 零值时整个字段省略
}

func main() {
    a := Account{ID: 1, Name: "alice", Password: "secret123"}

    data, _ := json.Marshal(a)
    fmt.Println(string(data))
}
```

```text
{"id":1,"name":"alice"}
```

对照结果:`password` 没了(`-`),`email` 没了(零值 + omitempty)。`omitempty` 判断的是 Go 零值:`""`、`0`、`false`、nil slice、nil map。

两个书写陷阱,都不报错、只是安静地不生效:

- tag 用的是**反引号**,里面是 `json:"..."` 的固定格式;
- `json:"email, omitempty"`(逗号后多个空格)会让 omitempty 失效——tag 只是个字符串,格式错了没人提醒你(`go vet` 能查出一部分)。

**一句话总结:tag 是给编码器看的翻译说明,格式错了静默失效,写完值得多看一眼。**

---

## 5. nil slice 和空 slice:null 还是 []

04 章埋过的伏笔在这兑现——两种"空"编码结果不同:

```go
type Response struct {
    Items []string `json:"items"`
}

func main() {
    a, _ := json.Marshal(Response{Items: nil})
    b, _ := json.Marshal(Response{Items: []string{}})

    fmt.Println(string(a))
    fmt.Println(string(b))
}
```

```text
{"items":null}
{"items":[]}
```

为什么值得在意:前端拿到 `null` 调 `.map()` 直接崩,拿到 `[]` 就没事。对外 API 想稳定返回 `[]`,就**主动初始化为空 slice**,别依赖零值。

---

## 6. 不知道形状时:map[string]any

有时 JSON 结构不固定,可以解到 `map[string]any`——这是最接近 `JSON.parse` 手感的用法:

```go
func main() {
    data := []byte(`{"name":"alice","age":18,"active":true,"tags":["a"]}`)

    var m map[string]any
    if err := json.Unmarshal(data, &m); err != nil {
        fmt.Println(err)
        return
    }

    fmt.Printf("%v %T\n", m["name"], m["name"])
    fmt.Printf("%v %T\n", m["age"], m["age"])
    fmt.Printf("%v %T\n", m["active"], m["active"])
    fmt.Printf("%v %T\n", m["tags"], m["tags"])
}
```

```text
alice string
18 float64
true bool
[a] []interface {}
```

对照表(`%T` 打印的 `[]interface {}` 就是 `[]any` 的原名):

| JSON | Go 里变成 |
|------|----------|
| object | `map[string]any` |
| array | `[]any` |
| string | `string` |
| number | `float64`(**注意:整数也是**) |
| boolean | `bool` |
| null | `nil` |

🕳️ 坑:`18` 解出来是 `float64` 不是 `int`——要当整数用得先类型断言 `m["age"].(float64)` 再转 `int`(05 章的断言语法)。用这些值步步都要断言,**结构明确时永远优先 struct**。

---

## 7. 流式处理:Decoder 和 Encoder

`Marshal`/`Unmarshal` 操作内存里的 `[]byte`;数据在 `io.Reader`/`io.Writer` 上流动时(文件、HTTP、网络),用配套的流式版本:

```go
func main() {
    r := strings.NewReader(`{"name":"alice","age":18}`)
    dec := json.NewDecoder(r)

    var u User
    if err := dec.Decode(&u); err != nil {
        fmt.Println(err)
        return
    }
    fmt.Printf("%+v\n", u)

    enc := json.NewEncoder(os.Stdout)
    enc.Encode(u)
}
```

```text
{Name:alice Age:18}
{"name":"alice","age":18}
```

记忆锚点:**Decoder 吃 Reader,Encoder 喂 Writer**——第 02 篇的通用插座又接上了。细节:`Encoder.Encode` 会在末尾补一个换行,正好适合写日志和 HTTP 响应。

---

## 8. 严格模式:DisallowUnknownFields

默认行为是"多出来的字段忽略"。处理外部请求时,这会让调用方**拼错字段名却毫无感知**。开严格模式:

```go
func main() {
    r := strings.NewReader(`{"name":"alice","agee":18}`) // age 拼错了
    dec := json.NewDecoder(r)
    dec.DisallowUnknownFields()

    var u User
    err := dec.Decode(&u)
    fmt.Println(err)
}
```

```text
json: unknown field "agee"
```

拼错的 `agee` 当场被抓出来,而不是 `Age` 静默留在零值。注意这是 `Decoder` 独有的开关,`Unmarshal` 没有——想对 `[]byte` 严格解码,包一层 `bytes.NewReader` 再用 Decoder。

---

## 9. 延迟解析:json.RawMessage

消息带 `type` 字段、`data` 形状随 type 变——这种"信封"结构用 `RawMessage` 分两步拆:

```go
type Event struct {
    Type string          `json:"type"`
    Data json.RawMessage `json:"data"` // 先囫囵吞下,原文保留
}

func main() {
    input := []byte(`{"type":"click","data":{"x":10,"y":20}}`)

    var e Event
    if err := json.Unmarshal(input, &e); err != nil {
        fmt.Println(err)
        return
    }
    fmt.Println(e.Type, string(e.Data))

    if e.Type == "click" {
        var c Click // struct{ X, Y int } 带 tag
        json.Unmarshal(e.Data, &c)
        fmt.Println(c.X, c.Y)
    }
}
```

```text
click {"x":10,"y":20}
10 20
```

第一遍只解信封(`Type`),`Data` 以原始字节原样保留;看清 type 之后,第二遍再把 `Data` 解成对应的具体结构。

---

## 10. 自定义编码:MarshalJSON

默认规则不够用时(比如时间要编码成特定格式),给类型实现这两个方法之一,`encoding/json` 会自动改用你的版本(05 章的隐式接口,又一次):

```go
type Celsius float64

func (c Celsius) MarshalJSON() ([]byte, error) {
    return []byte(fmt.Sprintf(`"%.1f°C"`, float64(c))), nil
}

func main() {
    data, err := json.Marshal(Reading{Temp: Celsius(36.6)}) // Reading 有字段 Temp Celsius `json:"temp"`
    fmt.Println(string(data), err)
}
```

```text
{"temp":"36.6°C"} <nil>
```

对应的解码侧是 `UnmarshalJSON([]byte) error`。注意返回的字节必须是**合法 JSON**(所以字符串要自带双引号)。先掌握默认规则,默认不够再自定义——大多数场景用不上。

---

## 本篇重点

- [ ] `Marshal`/`Unmarshal` 对应 `stringify`/`parse`,但操作 `[]byte`、带 error、解码必须传指针(忘 `&` 是运行时错误)。
- [ ] 只有大写开头的导出字段参与 JSON,小写字段静默消失——字段失踪先查大小写。
- [ ] struct tag 是"命名翻译层":`json:"name"` 改名、`json:"-"` 排除、`,omitempty` 零值省略;格式写错静默失效。
- [ ] nil slice 编码成 `null`,空 slice 编码成 `[]`;解到 `map[string]any` 时所有数字都是 `float64`。
- [ ] 流式用 Decoder(吃 Reader)/Encoder(喂 Writer);对外部请求开 `DisallowUnknownFields`,形状不定的字段用 `RawMessage` 两段式解析。

---

## 练习

1. 定义 `Config` struct(含 `app_name`、`port`、`debug` 三个 JSON 字段),编码成缩进 JSON 打印(提示:字段名和 JSON 名不一致,正是 tag 的用武之地)。
2. 从 JSON 字符串解码到 `Config`,打印 `%+v`(提示:第二个参数别忘了什么?)。
3. 写一个包含 `omitempty` 和 `json:"-"` 的结构体,分别在字段有值/零值时编码,观察差异。
4. 分别编码 nil slice 和空 slice 的字段,确认输出是 `null` 和 `[]`(提示:第 5 节原样复现一遍,自己跑一次印象才深)。
5. 用 `json.Decoder` 解码并开启 `DisallowUnknownFields`,故意拼错一个字段名,看错误信息(提示:输入用 `strings.NewReader` 包)。
