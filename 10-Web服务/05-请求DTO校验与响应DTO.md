# 请求 DTO、校验与响应 DTO

Web 服务的输入来自外部世界，不能直接信任。请求 DTO 用来描述 HTTP 输入，响应 DTO 用来描述对外输出。它们和业务模型可以相似，但职责不同。

---

## 1. 区分 DTO 和业务模型

业务模型：

```go
package task

type Task struct {
	ID     string
	Title  string
	Status Status
}
```

创建请求 DTO：

```go
package task

type createTaskRequest struct {
	Title string `json:"title"`
}
```

响应 DTO：

```go
package task

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

DTO 是 HTTP 契约。业务模型是内部规则的载体。两者分开后，内部字段变化不一定影响外部 API。

---

## 2. 安全解码 JSON

解码 JSON 时要限制 body 大小、拒绝未知字段、检查多余内容：

```go
package httpx

import (
	"encoding/json"
	"errors"
	"fmt"
	"io"
	"net/http"
)

func DecodeJSON(w http.ResponseWriter, r *http.Request, dst any, maxBytes int64) error {
	r.Body = http.MaxBytesReader(w, r.Body, maxBytes)
	defer r.Body.Close()

	decoder := json.NewDecoder(r.Body)
	decoder.DisallowUnknownFields()

	if err := decoder.Decode(dst); err != nil {
		return fmt.Errorf("decode json: %w", err)
	}

	if err := ensureSingleJSONValue(decoder); err != nil {
		return err
	}

	return nil
}

func ensureSingleJSONValue(decoder *json.Decoder) error {
	var extra struct{}
	if err := decoder.Decode(&extra); err != nil {
		if errors.Is(err, io.EOF) {
			return nil
		}
		return err
	}
	return fmt.Errorf("body must contain a single json value")
}
```

这里的第二次 `Decode` 只用于确认 body 里没有第二个 JSON 值。`{"title":"a"}{"title":"b"}` 这种输入应该被拒绝。

---

## 3. 请求校验

请求 DTO 可以有自己的校验方法：

```go
package task

import (
	"fmt"
	"strings"
	"unicode/utf8"
)

type createTaskRequest struct {
	Title string `json:"title"`
}

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

func (req createTaskRequest) ToInput() CreateTaskInput {
	return CreateTaskInput{
		Title: strings.TrimSpace(req.Title),
	}
}
```

校验应覆盖：

- 必填字段。
- 字符串长度。
- 数字范围。
- 枚举值。
- 字段之间的关系。
- 去除首尾空白后的真实值。

---

## 4. Handler 中使用请求 DTO

```go
package task

import (
	"log/slog"
	"net/http"

	"example.com/task-api/internal/httpx"
)

func (h *HTTPHandler) Create(w http.ResponseWriter, r *http.Request) {
	var req createTaskRequest
	if err := httpx.DecodeJSON(w, r, &req, 1<<20); err != nil {
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

`400 Bad Request` 更适合 JSON 格式错误。`422 Unprocessable Entity` 更适合字段语义校验失败。团队也可以统一都用 `400`，关键是前后一致。

---

## 5. 响应编码

通用 JSON 响应函数：

```go
package httpx

import (
	"encoding/json"
	"net/http"
)

func WriteJSON(w http.ResponseWriter, status int, value any) error {
	w.Header().Set("Content-Type", "application/json; charset=utf-8")
	w.WriteHeader(status)
	return json.NewEncoder(w).Encode(value)
}
```

错误响应也应该有固定形状：

```go
package httpx

import "net/http"

type ErrorResponse struct {
	Error ErrorBody `json:"error"`
}

type ErrorBody struct {
	Code    string `json:"code"`
	Message string `json:"message"`
}

func WriteErrorWithCode(w http.ResponseWriter, status int, code string, message string) {
	body := ErrorResponse{
		Error: ErrorBody{
			Code:    code,
			Message: message,
		},
	}
	if err := WriteJSON(w, status, body); err != nil {
		http.Error(w, http.StatusText(http.StatusInternalServerError), http.StatusInternalServerError)
	}
}

func WriteError(w http.ResponseWriter, status int, message string) {
	WriteErrorWithCode(w, status, http.StatusText(status), message)
}
```

客户端依赖响应结构，错误也要像成功响应一样稳定。

---

## 6. 字段输出控制

响应 DTO 可以隐藏内部字段：

```go
type user struct {
	ID           string
	Email        string
	PasswordHash string
	IsAdmin      bool
}

type userResponse struct {
	ID      string `json:"id"`
	Email   string `json:"email"`
	IsAdmin bool   `json:"is_admin"`
}

func userResponseFromModel(user user) userResponse {
	return userResponse{
		ID:      user.ID,
		Email:   user.Email,
		IsAdmin: user.IsAdmin,
	}
}
```

不要直接把带敏感字段的内部模型写成 JSON 响应。

---

## 7. 学习检查

检查请求和响应代码时，重点看：

1. body 是否限制大小。
2. JSON 是否拒绝未知字段。
3. 是否检查多余 JSON 值。
4. 请求 DTO 是否有校验。
5. 字符串是否 trim。
6. 响应 DTO 是否避免泄露内部字段。
7. 错误响应结构是否统一。
