# 05: 请求 DTO、校验与响应 DTO

Web 服务的输入来自外部世界,**什么鬼东西都可能进来**:超大 body、多余字段、全空格的标题、拼在一起的两个 JSON。这篇解决的问题是:在 HTTP 边界立好两道墙——请求 DTO 负责"进来的必须干净",响应 DTO 负责"出去的不能带秘密"。

对比 Node:`express.json()` 帮你把 body 变成对象,但它默认相当宽松;校验你大概用 zod 或 Joi。Go 里这两件事都是显式的:`json.Decoder` 配几个开关,校验就是普通方法——没有 schema 库也够用。

---

## 1. 区分 DTO 和业务模型

业务模型(内部规则的载体):

```go
type Task struct {
    ID     string
    Title  string
    Status Status
}
```

创建请求 DTO(只描述"客户端能发什么"):

```go
type createTaskRequest struct {
    Title string `json:"title"`
}
```

响应 DTO(只描述"服务端对外说什么")+ 转换函数:

```go
type taskResponse struct {
    ID     string `json:"id"`
    Title  string `json:"title"`
    Status string `json:"status"`
}

func taskResponseFromModel(task Task) taskResponse {
    return taskResponse{
        ID:     task.ID,
        Title:  task.Title,
        Status: string(task.Status),
    }
}
```

为什么不直接把 `Task` 序列化出去?因为 DTO 是 **HTTP 契约**,业务模型是**内部实现**——分开后,内部加字段、改类型,不会意外改变对外 API。注意 DTO 类型名是小写开头(未导出):它们只在 task 包内部使用,不需要出口。

**一句话总结:模型给自己用,DTO 给外面看,中间靠转换函数摆渡。**

---

## 2. 安全解码 JSON

### 🕳️ 坑:`json.Decode` 默认来者不拒

以为会怎样:解码时 Go 会像校验类型一样严格,多余字段应该报错。
实际怎样:默认**静默忽略**未知字段;body 有 10GB 它也照读;`{"a":1}{"b":2}` 两个 JSON 拼一起,它读完第一个就当没事。
为什么:`encoding/json` 的默认行为面向"宽容读取",但 HTTP 边界需要"严格把关"。三个问题对应三个开关:

```go
func DecodeJSON(w http.ResponseWriter, r *http.Request, dst any, maxBytes int64) error {
    r.Body = http.MaxBytesReader(w, r.Body, maxBytes)   // ① 限制 body 大小
    defer r.Body.Close()

    decoder := json.NewDecoder(r.Body)
    decoder.DisallowUnknownFields()                     // ② 拒绝未知字段

    if err := decoder.Decode(dst); err != nil {
        return fmt.Errorf("decode json: %w", err)
    }

    return ensureSingleJSONValue(decoder)               // ③ 拒绝第二个 JSON 值
}

func ensureSingleJSONValue(decoder *json.Decoder) error {
    var extra struct{}
    if err := decoder.Decode(&extra); err != nil {
        if errors.Is(err, io.EOF) {
            return nil          // 后面没东西了,正常
        }
        return err
    }
    return fmt.Errorf("body must contain a single json value")
}
```

拿各种脏输入实跑一遍:

```text
body={"title":"learn go web"}                   ok title="learn go web"
body={"title":"a","hack":"x"}                   err=decode json: json: unknown field "hack"
body={"title":"a"}{"title":"b"}                 err=json: unknown field "title"
body={"title":"a"} {}                           err=body must contain a single json value
body={"title":                                  err=decode json: unexpected EOF
```

一个有意思的细节:第三行的两个拼接 JSON 也被拒了,但错误信息来自第二次 `Decode` 的未知字段检查(它先撞上 `"title"`),而不是我们自己那句 "single json value"——只有第二个值能成功解进 `struct{}`(比如空对象)时才会走到自定义消息。**两条路都通向拒绝**,行为是对的,只是错误文案来源不同,别在调试时被绕晕。

---

## 3. 请求校验

请求 DTO 自带校验方法,不需要 schema 库:

```go
func (req createTaskRequest) Validate() error {
    title := strings.TrimSpace(req.Title)
    if title == "" {
        return fmt.Errorf("title is required")
    }
    if utf8.RuneCountInString(title) > 120 {
        return fmt.Errorf("title must be <= 120 characters")
    }
    return nil
}

func (req createTaskRequest) ToInput() CreateInput {
    return CreateInput{
        Title: strings.TrimSpace(req.Title),
    }
}
```

两个模块 02 的知识点在这儿上岗了:

- 长度限制用 `utf8.RuneCountInString`(数字符)而不是 `len`(数字节)——不然 40 个汉字的标题就"超长"了。
- 先 `TrimSpace` 再判断:`"   "` 这种全空格标题必须挡住。

校验清单:必填字段、字符串长度、数字范围、枚举值、字段之间的关系、trim 后的真实值。对比 JS:这些就是你写在 zod schema 里的规则,Go 的社区习惯是简单场景手写,复杂了再上 `go-playground/validator` 这类 tag 校验库。

---

## 4. Handler 中使用请求 DTO

解码和校验各自失败时,给不同的状态码:

```go
func (h *HTTPHandler) Create(w http.ResponseWriter, r *http.Request) {
    var req createTaskRequest
    if err := httpx.DecodeJSON(w, r, &req, 1<<20); err != nil {    // 1MB 上限
        httpx.WriteErrorWithCode(w, http.StatusBadRequest, "invalid_json", "invalid json body")
        return
    }

    if err := req.Validate(); err != nil {
        httpx.WriteErrorWithCode(w, http.StatusUnprocessableEntity, "invalid_request", err.Error())
        return
    }

    created, err := h.service.Create(r.Context(), req.ToInput())
    if err != nil {
        writeTaskError(w, err)
        return
    }

    resp := taskResponseFromModel(created)
    if err := httpx.WriteJSON(w, http.StatusCreated, resp); err != nil {
        h.logger.Error("write response failed", slog.String("error", err.Error()))
    }
}
```

实跑三种输入:

```bash
curl -s -H 'Content-Type: application/json' -d '{"title":'       http://localhost:19080/tasks
curl -s -H 'Content-Type: application/json' -d '{"title":"   "}' http://localhost:19080/tasks
curl -si -H 'Content-Type: application/json' -d '{"title":"learn go web"}' http://localhost:19080/tasks
```

```text
{"error":{"code":"invalid_json","message":"invalid json body"}}
{"error":{"code":"invalid_request","message":"title is required"}}
HTTP/1.1 201 Created
Content-Type: application/json; charset=utf-8
X-Request-Id: e2a0d4b3c5fe13d6896785ee8d4b86fc
Date: Thu, 16 Jul 2026 17:14:04 GMT
Content-Length: 141

{"id":"task_1","title":"learn go web","status":"open","created_at":"2026-07-16T17:14:04.298555Z","updated_at":"2026-07-16T17:14:04.298555Z"}
```

状态码约定:`400` 给"JSON 本身坏了",`422` 给"JSON 没坏但字段不合法"。团队统一都用 400 也行——**关键是前后一致**。

---

## 5. 响应编码

成功响应走一个通用函数:

```go
func WriteJSON(w http.ResponseWriter, status int, value any) error {
    w.Header().Set("Content-Type", "application/json; charset=utf-8")
    w.WriteHeader(status)
    return json.NewEncoder(w).Encode(value)
}
```

🕳️ 小坑:顺序必须是 **Header → WriteHeader → Body**。`WriteHeader` 之后再 `Header().Set` 是无效的(头已经发出去了)。

错误响应也要有固定形状——客户端解析错误的代码只想写一次:

```go
type ErrorResponse struct {
    Error ErrorBody `json:"error"`
}

type ErrorBody struct {
    Code    string `json:"code"`
    Message string `json:"message"`
}

func WriteErrorWithCode(w http.ResponseWriter, status int, code string, message string) {
    body := ErrorResponse{Error: ErrorBody{Code: code, Message: message}}
    if err := WriteJSON(w, status, body); err != nil {
        http.Error(w, http.StatusText(http.StatusInternalServerError), http.StatusInternalServerError)
    }
}
```

**一句话总结:错误响应和成功响应一样,都是对外契约的一部分,结构要稳定。**

---

## 6. 字段输出控制

### 🕳️ 坑:内部模型直接 `json.Marshal` 出去

以为会怎样:省一个类型定义,字段反正差不多。
实际怎样:模型里的 `PasswordHash`、内部标记、软删除字段……全部原样进了 HTTP 响应。
为什么:`encoding/json` 会序列化**所有导出字段**,它不知道哪个是秘密。响应 DTO 就是白名单:

```go
type user struct {
    ID           string
    Email        string
    PasswordHash string   // 绝不能出去
    IsAdmin      bool
}

type userResponse struct {
    ID      string `json:"id"`
    Email   string `json:"email"`
    IsAdmin bool   `json:"is_admin"`
}

func userResponseFromModel(user user) userResponse {
    return userResponse{ID: user.ID, Email: user.Email, IsAdmin: user.IsAdmin}
}
```

JS 里同样的事故形态是 `res.json(userDoc)` 把 Mongoose 文档整个吐出去。两边的药方一致:**出口只走白名单 DTO**。

---

## 本篇重点

- [ ] DTO 是 HTTP 契约,业务模型是内部实现,靠转换函数摆渡;两边字段像不代表该合并。
- [ ] 解码三开关:`MaxBytesReader` 限大小、`DisallowUnknownFields` 拒多余字段、二次 `Decode` 查拼接 JSON——默认的 `json.Decode` 三个都不管。
- [ ] 校验写在 DTO 的 `Validate` 方法上:trim 后判空、`RuneCountInString` 数字符、枚举、范围。
- [ ] `400` = JSON 坏了,`422` = 字段不合法;错误响应用固定的 `{"error":{"code","message"}}` 结构。
- [ ] 响应永远走白名单 DTO,别把带敏感字段的内部模型直接 Marshal 出去。

---

## 练习

给 `book-api` 的创建接口立起两道墙:

要求:

1. `createBookRequest` 含 `title`、`author`、`pages` 三个字段;校验:title/author trim 后非空且 ≤ 100 字符,pages 在 1~10000 之间。
2. 用第 2 节的 `DecodeJSON`(1MB 上限)解码;构造四种脏输入(未知字段、两个拼接 JSON、截断 JSON、全空格 title)用 curl 打一遍,记录每种的状态码和 body。
3. `Book` 模型加一个 `InternalNote string` 字段,确认它**不会**出现在任何响应里。
4. JSON 格式错误回 400 + `invalid_json`,校验失败回 422 + `invalid_request`。

提示:中文书名的长度断言,用 `len` 和 `RuneCountInString` 各算一次,看看差多少;拼接 JSON 被拒时留意错误信息来自哪条路(第 2 节末尾的细节)。
