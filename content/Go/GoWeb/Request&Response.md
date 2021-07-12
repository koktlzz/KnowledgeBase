---
title: "Request&Response"
date: 2020-11-04T09:19:42+01:00
lastmod: 2020-11-04T09:19:42+01:00
draft: false
menu:
  go:
    parent: "Go Web"
weight: 200
---

## http.Request

### 结构

- 请求行（请求方法/URL/协议）
- 0 个或多个 Header
- 空行
- 可选的消息体 (Body)

例如：

```http
GET www.baidu.com HTTP/1.1
Host: www.w3.org
User-Agent: Mozilla/5.0
（空行）
（无消息体）
```

### 读取查询参数

常见的 URL 的结构为：

```http
scheme://[userinfo@]host/path[?query][#fragment]
```

使用 Request.URL 的 Query 方法即可得到 URL 中的查询参数：

```go
fmt.Println(r.URL.Query())
```

若请求的 URL 为：*http://localhost:8080?name=kokt&age=3* , 则控制台输出：

```go
map[age:[3] name:[kokt]]
```

### 读取请求头

```go
type Header map[string][]string
```

可以使用 Request.Header\[key\] 或 Request.Header.Get(key) 获取请求头中的值。前者返回值类型是 []string，后者返回值类型为 string（不同值以逗号分隔）。

### 读取消息体

```go
func obtainBody(w http.ResponseWriter, r *http.Request) {
    length := r.ContentLength
    body := make([]byte, length)
    // r.Body 是一个 io.ReadCloser 类型
    r.Body.Read(body)
    fmt.Fprint(w, string(body))
}
```

使用 REST Client 向服务端发送带有消息体的 POST 请求：

```http
POST http://localhost:8080/obtainBody  HTTP/1.1
Content-Type: application/json

{
    "name":kokt,
    "age":3
}
```

服务端得到请求的消息体：

```http
HTTP/1.1 200 OK
Date: Wed, 06 Jan 2021 16:25:58 GMT
Content-Length: 32
Content-Type: text/plain; charset=utf-8
Connection: close

{
    "name":kokt,
    "age":3
}
```

### 处理来自表单的请求

如果页面上有一个表单，其功能为：当用户点击提交按钮后，就会向服务端发起一个 POST 请求，请求的页面跳转至/process。上述页面的 html 代码如下：

```html
<!DOCTYPE html>
<body>
<form method="POST" action="/process" enctype="application/x-www-form-urlencoded">
    <input name="name" type="text">
    <input name="age" type="text">
    <input type="submit">
</form>
</body>
```

其中，表单的 enctype 有"application/x-www-form-urlencoded"和"multipart/form-data"两种。前者会将表单数据编码到查询字符串 (r.URL.Query()) 中，适用于简单文本；后者会将每一个键值对转换为一个 MIME 消息部分，每部分具有独立的 Content-Type 和 Content-Disposition，适用于大量数据和文件的上传。

可以通过 Request.Form 获取提交的表单数据：

```go
func form(w http.ResponseWriter, r *http.Request) {
    r.ParseForm() // 解析表单，若不调用则 r.Form 为空
    fmt.Println(r.Form)
}
```

若在页面填入 kokt 和 3 后点击提交按钮，则控制台将会输出：

```go
map[name:[kokt] age:[3]]
```

如果表单提交后跳转的 URL 中带有查询字符串，Request.Form 可以把它们连同表单中的数据一同输出，而若使用 Request.PostForm 则只会输出表单中的数据。另外，上述两种方法仅支持表单的 enctype 属性为"application/x-www-form-urlencoded"，若该属性为"multipart/form-data"，则需要使用 Request.MultipartForm 字段：

```go
func form(w http.ResponseWriter, r *http.Request) {
    r.ParseMultipartForm(1024) // 作用类似于 r.ParseForm()，其参数为需要读取数据的长度（字节数）
    fmt.Println(r.MultipartForm)
}
```

它的返回值不同于 Request.Form 的 map，而是一个指向 multipart.Form 的指针：

```go
type Request struct {
...
    MultipartForm *multipart.Form
...
}
type Form struct {
    Value map[string][]string
    File  map[string][]*FileHeader
}
```

multipart.Form 中有两个 map，第一个包含了表单中提交的数据，第二个包含了上传的文件。因此若在页面填入 kokt 和 3 后点击提交按钮，则控制台将会输出：

```go
&{map[name:[kokt] age:[3]] map[]}
```

### 处理上传的文件

创建一个可以让用户上传文件的页面，该页面的 html 代码如下：

```html
<!DOCTYPE html>
<body>
<form method="POST" action="/process" enctype="multipart/form-data">
    <input name="upload" type="file">
    <input type="submit">
</form>
</body>
```

服务端利用 Request.MultipartForm 字段获取文件，随后写入到页面的响应中：

```go
func upload(w http.ResponseWriter, r *http.Request) {
    r.ParseMultipartForm(1024)
    // 若上传多个文件，只读取第一个
    fileHeader := r.MultipartForm.File["upload"][0]
    // 忽略异常处理
    file, _ := fileHeader.Open()
    defer file.Close()
    data, _ := ioutil.ReadAll(file) //data 的类型为 []byte
    fmt.Fprintf(w, string(data))
}
```

其中，fileHeader 的类型为 multipart.FileHeader：

```go
type FileHeader struct {
    Filename string
    Header   textproto.MIMEHeader
}
```

若上传的文件只有一个，则可使用更为简单的 Request.FormFile 字段。另外，下列代码还添加了一个将上传的文件下载到项目目录下的功能：

```go
    // 无需调用 r.ParseMultipartForm() 方法
    file, fileHeader, err := r.FormFile("upload")
    if err != nil {
        log.Fatal(err)
    }
    defer file.Close()
    // 新建一个与上传文件同名的文件
    newfile, err := os.OpenFile(fileHeader.Filename, os.O_WRONLY|os.O_CREATE, 0666)
    if err != nil {
        log.Fatal(err)
    }
    defer newfile.Close()
    // 将上传的文件内容拷贝至新建的文件中
    io.Copy(newfile, file)
```

上述处理表单数据和文件的方法均是将请求的消息体当作一个对象处理，一次性地获得整个或多个 map。而 Request.MultipartReader 字段可以将请求的消息体当作一个流 (stream) 进行处理，它仅适用于 enctype 值为"multipart/form-data"的 POST 请求。

```go
func (r *Request) MultipartReader() (*multipart.Reader, error)
```

Request.ParseForm 无法解析 application/json。

## http.response

http.response 代表对一个 HTTP 请求的响应。由于该结构体首字母小写，因此无法在包外调用。而 \*http.response 实现了 http.ResponseWriter 的全部方法。因此我们可以将 \*http.response 当作一个 http.ResponseWriter 传入函数，从而通过引用传递的方式操作 http.response。

```go
type ResponseWriter interface {
// Header 返回一个 Header 类型值，该值会被 WriteHeader 方法发送。
// 在调用 WriteHeader 或 Write 方法后再改变该对象是没有意义的。
Header() Header
// WriteHeader 该方法发送 HTTP 回复的头域和状态码。
// 如果没有被显式调用，第一次调用 Write 时会触发隐式调用 WriteHeader(http.StatusOK)
// WriterHeader 的显式调用主要用于发送错误码。
WriteHeader(int) // 参数：HTTP 状态码
// Write 向连接中写入作为 HTTP 的一部分回复的数据。
// 如果被调用时还未调用 WriteHeader，本方法会先调用 WriteHeader(http.StatusOK)
// 如果 Header 中没有"Content-Type"键，本方法会使用包函数 DetectContentType 检查数据的前 512 字节，将返回值作为该键的值。
Write([]byte) (int, error)
}
```

```go
type response struct {
    conn             *conn
    req              *Request // request for this response
    ...
}

func (w *response) Header() Header {
    if w.cw.header == nil && w.wroteHeader && !w.cw.wroteHeader {
        w.cw.header = w.handlerHeader.Clone()
    }
    w.calledHeader = true
    return w.handlerHeader
}

func (w *response) WriteHeader(code int) {
    if w.conn.hijacked() {
        caller := relevantCaller()
        w.conn.server.logf("http: response.WriteHeader on hijacked connection from %s (%s:%d)", caller.Function, path.Base(caller.File), caller.Line)
        return
    }
    ...
}

func (w *response) Write(data []byte) (n int, err error) {
    return w.write(len(data), data, "")
}
```

这也解释了为什么 ServeHTTP 方法中的两个参数：ResponseWriter 和 \*Request，只有 Request 是按引用传递的，因为传入 ResponseWriter 的变量本质是 \*response。

### 内置响应

- NotFound 函数：包装了一个 404 状态码和额外信息；
- ServeFile 函数：从文件系统提供文件，返回给请求者；
- ServeContent 函数：可以把实现了 io.ReadSeeker 接口的任何东西里面的内容返回给请求者（还可以处理范围请求，即只请求了资源中的一部分内容，ServeFile 或 io.Copy 则无法做到）；
- Redirect 函数：告诉客户端重定向到另一个 URL。
