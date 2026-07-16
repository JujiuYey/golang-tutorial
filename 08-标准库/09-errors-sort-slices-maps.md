# 09: errors、sort、slices、maps 与常用辅助包

这篇是"工具腰带"合集:错误链判断(`errors.Is/As`,03 章埋的伏笔正式兑现)、排序(`sort`/`slices`)、map 辅助(`maps`)、随机数、base64、正则。单个都不大,但都是写代码时"咦这个 Go 怎么弄"的高频答案。

---

## 1. errors.Is:错误被包装后还认得出吗

03 章说过用 `%w` 包装错误。包装之后,`==` 就认不出原始错误了,得用 `errors.Is` 沿着错误链找:

```go
func main() {
    _, err := os.ReadFile("ghost.json")
    wrapped := fmt.Errorf("load config: %w", err)

    fmt.Println(wrapped)
    fmt.Println(errors.Is(wrapped, os.ErrNotExist))
    fmt.Println(wrapped == os.ErrNotExist)
}
```

```text
load config: open ghost.json: no such file or directory
true
false
```

最后两行是重点:`errors.Is` 说"链子里有 ErrNotExist"(true),`==` 说"这不是同一个值"(false)。

🕳️ 坑:判断错误写 `err == os.ErrNotExist`,只要中间有人 `%w` 包装过一层就判不中了。**判断特定错误,永远用 `errors.Is`。** 上一章第 03 篇判断文件存在时已经用过一次。

---

## 2. errors.As:从错误链里提取具体类型

`Is` 回答"是不是这个错",`As` 回答"能不能把它**掏出来**当具体类型用"——比如拿到出错的路径:

```go
func main() {
    _, err := os.ReadFile("ghost.json")

    var pathErr *fs.PathError
    if errors.As(err, &pathErr) {
        fmt.Println("op:", pathErr.Op)
        fmt.Println("path:", pathErr.Path)
    }
}
```

```text
op: open
path: ghost.json
```

拆一下用法:

```text
var pathErr *fs.PathError      // ①准备一个目标类型的空变量
if errors.As(err, &pathErr) {  // ②As 在链子里找,找到就填进去
    pathErr.Path               // ③然后就能访问具体字段了
```

和 05 章的类型断言 `err.(*fs.PathError)` 的区别:断言只看最外层,`As` 会**沿着包装链一路找**。

---

## 3. sort:经典排序

```go
func main() {
    nums := []int{3, 1, 2}
    sort.Ints(nums)
    fmt.Println(nums)

    names := []string{"bob", "alice"}
    sort.Strings(names)
    fmt.Println(names)

    users := []User{{"bob", 30}, {"alice", 25}}
    sort.Slice(users, func(i, j int) bool {
        return users[i].Age < users[j].Age
    })
    fmt.Println(users)
}
```

```text
[1 2 3]
[alice bob]
[{alice 25} {bob 30}]
```

两个和 JS `arr.sort()` 的不同:

- **原地排序,没有返回值**——写 `sorted := sort.Ints(nums)` 直接编译不过。
- 自定义排序的回调收的是**下标 i、j**,返回"i 应不应该排在 j 前面"(bool),不是 JS 那种返回负零正的比较数。

顺带解毒一个 JS 老坑:JS 的 `[3,1,20].sort()` 会按字符串比出 `[1, 20, 3]`;Go 的 `sort.Ints` 就是数字排序,没这种惊喜。

---

## 4. slices:新一代切片工具箱

Go 1.21+ 的 `slices` 包基于泛型(05 章),不用再按类型挑函数:

```go
func main() {
    nums := []int{3, 1, 2}

    slices.Sort(nums)
    fmt.Println(nums)

    fmt.Println(slices.Contains(nums, 2))
    fmt.Println(slices.Index(nums, 3))
    fmt.Println(slices.Max(nums))
    fmt.Println(slices.Equal([]int{1, 2}, []int{1, 2}))
}
```

```text
[1 2 3]
true
2
3
true
```

对 JS 用户很好记:`Contains` ≈ `includes`,`Index` ≈ `indexOf`(没找到返回 -1,一样)。

`slices.Equal` 存在的理由:04 章说过 slice 不能用 `==` 比较(只能和 nil 比),逐元素比较就用它。新代码优先 `slices.Sort`(泛型、更快),`sort.Slice` 留给按字段自定义排序——或者用它的泛型版 `slices.SortFunc`。

---

## 5. maps:map 工具箱

```go
func main() {
    a := map[string]int{"go": 1}
    b := maps.Clone(a)

    b["go"] = 2
    fmt.Println(a["go"], b["go"])

    fmt.Println(maps.Equal(a, map[string]int{"go": 1}))
}
```

```text
1 2
true
```

`Clone` 呼应 04 章的引用语义:map 赋值 `b := a` 是两个变量指向**同一张表**,改 b 就是改 a;`maps.Clone(a)` 才是真复制——输出证明改克隆不影响原件(浅拷贝:value 是指针/slice 时仍共享)。`maps.Equal` 同理:map 也不能 `==`,逐项比较用它。

---

## 6. math 和 math/rand

```go
func main() {
    fmt.Println(math.Sqrt(9))
    fmt.Println(math.Round(3.6))

    fmt.Println(rand.Intn(10))
    fmt.Println(rand.Intn(10))
}
```

```text
3
4
9
9
```

(后两行是 0~9 的随机数,你的输出会不同。)

`math` 对应 JS 的 `Math`,函数名几乎一样。`math/rand` 对应 `Math.random()`,`Intn(10)` 直接给 `[0,10)` 的整数,不用自己乘再取整。

🕳️ 坑:**`math/rand` 是伪随机,不能用于安全场景。** 生成密码、token、密钥,必须用下一节的 `crypto/rand`——用错这个的后果是安全漏洞,不是 bug。

---

## 7. crypto/rand:安全随机数

```go
func main() {
    buf := make([]byte, 16)
    if _, err := rand.Read(buf); err != nil {
        fmt.Println(err)
        return
    }
    fmt.Printf("%x\n", buf)
}
```

```text
600b90266987d5affb111380f6e313a6
```

(密码学随机,每次必然不同。)

注意这里的 `rand` 来自 `crypto/rand`。如果一个文件同时需要两个 rand 包,就到了第 01 篇说的"唯一该起别名的时候":

```go
import (
    crand "crypto/rand"
    "math/rand"
)
```

---

## 8. encoding/base64

```go
func main() {
    data := []byte("hello, go")

    encoded := base64.StdEncoding.EncodeToString(data)
    fmt.Println(encoded)

    decoded, err := base64.StdEncoding.DecodeString(encoded)
    fmt.Println(string(decoded), err)

    _, err = base64.StdEncoding.DecodeString("not base64!!!")
    fmt.Println(err)
}
```

```text
aGVsbG8sIGdv
hello, go <nil>
illegal base64 data at input byte 3
```

对应 JS 的 `btoa`/`atob`(或 Node 的 `Buffer.from(s, 'base64')`)。注意第三行:**解码可能失败**,error 要接。

🕳️ 坑:base64 是**编码**不是加密——任何人都能一行解回原文。拿它"保护"密码等于明文存储。

---

## 9. regexp:正则

```go
func main() {
    re := regexp.MustCompile(`^[a-z0-9_]+$`)
    fmt.Println(re.MatchString("go_123"))
    fmt.Println(re.MatchString("Go 123"))

    digits := regexp.MustCompile(`\d+`)
    fmt.Println(digits.FindAllString("a1 b22 c333", -1))
}
```

```text
true
false
[1 22 333]
```

和 JS 的差异清单:

- 没有 `/pattern/flags` 字面量,用反引号字符串写(02-05:反引号不转义,`\d` 不用写成 `\\d`)。
- 先 `Compile` 出对象再反复用,不像 JS 直接在字符串方法里塞正则。
- `MustCompile` 在正则非法时 panic——适合**包级变量**这种"写错了应该当场炸"的场景;正则来自用户输入时用 `regexp.Compile` 并处理 error(03 章的 Must 前缀约定)。

---

## 10. cmp:比较小工具

```go
func main() {
    fmt.Println(cmp.Compare(1, 2))
    fmt.Println(cmp.Compare(2, 2))

    fmt.Println(cmp.Or("", "fallback"))
    fmt.Println(cmp.Or("value", "fallback"))
}
```

```text
-1
0
fallback
value
```

`cmp.Compare` 返回 -1/0/1,配合 `slices.SortFunc` 用。`cmp.Or` 返回第一个**非零值**参数——手感接近 JS 的 `a || b` 取默认值,但注意判断标准是 Go 的零值,不是 truthy/falsy(02 章:Go 根本没有 truthy)。

---

## 本篇重点

- [ ] 判断特定错误用 `errors.Is`(穿透 `%w` 包装链),`==` 会被包装挡住;提取具体错误类型用 `errors.As` + 目标指针。
- [ ] Go 排序是原地的、无返回值;新代码优先泛型的 `slices.Sort` / `SortFunc`,`sort.Slice` 的回调收下标返回 bool。
- [ ] slice 和 map 不能 `==`,比较用 `slices.Equal` / `maps.Equal`;复制 map 用 `maps.Clone`(浅拷贝)。
- [ ] 随机数分两种:`math/rand` 玩具级,密码/token/密钥必须 `crypto/rand`;base64 是编码不是加密。
- [ ] `regexp.MustCompile` 只用于写死的包级正则,动态输入用 `Compile` + error;`cmp.Or` 按零值(不是 truthy)取默认值。

---

## 练习

1. 用 `errors.Is` 判断文件不存在(提示:先 `%w` 包装一层再判断,验证它能穿透)。
2. 用 `errors.As` 提取 `*fs.PathError` 并打印 `.Path` 字段(提示:目标变量声明成指针,As 的第二个参数再取一次地址)。
3. 按年龄排序 `[]User`(提示:`sort.Slice` 或 `slices.SortFunc` 任选,试试后者配 `cmp.Compare`)。
4. 用 `slices.Contains` 判断元素是否存在(提示:一行)。
5. 用 `maps.Clone` 克隆 map,验证修改克隆不影响原 map(提示:对照 04 章直接赋值的行为,写两段代码对比)。
