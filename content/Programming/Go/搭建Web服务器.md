---
title: "搭建 Web 服务器"
date: 2020-11-04T09:19:42+01:00
lastmod: 2020-11-04T09:19:42+01:00
draft: false
menu:
  programming:
    parent: "Go"
weight: 300
---

## 搭建一个简单的 web 服务

```go
package main

import (
    "fmt"
    "net/http"
    "strings"
    "log"
)

func sayhelloName(w http.ResponseWriter, r *http.Request) {
    r.ParseForm()  // 解析参数，默认是不会解析的
    fmt.Println(r.Form)  // 这些信息是输出到服务器端的打印信息
    fmt.Println("path", r.URL.Path)
    fmt.Println("scheme", r.URL.Scheme)
    fmt.Println(r.Form["num"])
    for k, v := range r.Form {
        fmt.Println("key:", k)
        fmt.Println("val:", strings.Join(v, ""))
    }
    fmt.Fprintf(w, "Hello World") // 这个写入到 w 的是输出到客户端的
}

func main() {
    http.HandleFunc("/", sayhelloName) // 注册了请求/的路由规则，当请求 uri 为"/"，路由就会转到函数 sayhelloName
    err := http.ListenAndServe(":9090", nil) // 设置监听的端口
    if err != nil {
        log.Fatal("ListenAndServe: ", err)
    }
}
```

在上述代码中，主函数调用了 net/http 包下的两个函数 HandleFunc() 和 ListenAndServe()。

## HandleFunc()

``` go
// http.HandleFunc("/", sayhelloName)
func HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
    DefaultServeMux.HandleFunc(pattern, handler)
}
```

在本例中 pattern = "/", handler = sayhelloName()，代表 HandleFunc() 注册了请求"/"的路由规则：当请求 uri 为"/"，路由就会转到函数 sayhelloName()。

函数 HandleFunc() 的底层实现步骤为：

1. 调用 DefaultServeMux.HandleFunc() 方法

   ```go
   // DefaultServeMux.HandleFunc(pattern, handler)
   func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
       if handler == nil {
           panic("http: nil handler")
       }
       mux.Handle(pattern, HandlerFunc(handler))
   }
   ```

   其中，DefaultServeMux 的类型是 \*ServeMux:

   ```go
   // DefaultServeMux is the default ServeMux used by Serve.
   var DefaultServeMux = &defaultServeMux
   var defaultServeMux ServeMux
   ```

   > ServeMux 类型是 HTTP 请求的多路转接器。它会将每一个接收的请求的 URL 与一个注册模式的列表进行匹配，并调用和 URL 最匹配的模式的处理器。

2. 调用 mux.Handle() 方法，向 DefaultServeMux 的 map[string]muxEntry 中增加对应的 handler 和路由规则。

   ```go
   // mux.Handle(pattern, HandlerFunc(handler))
   func (mux *ServeMux) Handle(pattern string, handler Handler) {
       mux.mu.Lock()
       defer mux.mu.Unlock()
       ...
       if mux.m == nil {
           mux.m = make(map[string]muxEntry)
       }
       e := muxEntry{h: handler, pattern: pattern}
       mux.m[pattern] = e  // 增加对应的 handler 和路由规则
       ...
   }
   ```

   其中，ServeMux 类型的属性如下：

   ```go
   type ServeMux struct {
       mu sync.RWMutex   //锁，由于请求涉及到并发处理，因此这里需要一个锁机制
       m  map[string]muxEntry  // 路由规则，一个 string 对应一个 mux 实体，这里的 string 就是注册的路由表达式
       hosts bool // 是否在任意的规则中带有 host 信息
   }
   
   type muxEntry struct {
       explicit bool   // 是否精确匹配
       h        Handler // 这个路由表达式对应哪个 handler
       pattern  string  //匹配字符串
   }
   ```

3. 我们可以看到 muxEntry 类型的 key[h] 对应 Handler 类型：

   ```go
   type Handler interface {
       ServeHTTP(ResponseWriter, *Request)
   }
   ```

   而我们定义的函数 sayhelloName 并未实现该接口，因此理论上是不能作为参数放入 e := muxEntry{h: handler, pattern: pattern} 表达式中的。这时我们注意到在第二步调用 mux.Handle(pattern, HandlerFunc(handler)) 时，其实已经通过 HandlerFunc(handler) 将函数 sayhelloName 强制转换为 HandlerFunc 类型。而类型 HandlerFunc 默认拥有方法：ServeHTTP ，因此 sayhelloName 就实现了 Handler 接口，从而可以为特定的路径处理 HTTP 请求。

   ```go
   // HandlerFunc(handler)
   type HandlerFunc func(ResponseWriter, *Request)
   
   func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request)
   ```

## ListenAndServe()

```go
// err := http.ListenAndServe(":9090", nil)
func ListenAndServe(addr string, handler Handler) error {
    server := &Server{Addr: addr, Handler: handler}
    return server.ListenAndServe()
}
```

其底层实现的步骤为：

1. server := &Server{Addr: addr, Handler: handler}实例化了一个 Server，它定义了运行 HTTP 服务端的参数：

   ```go
   type Server struct {
       Addr           string        // 监听的 TCP 地址，如果为空字符串会使用":http"
       Handler        Handler       // 调用的处理器，如为 nil 会调用 http.DefaultServeMux
   ...
   }
   ```

2. 调用 Server 的 ListenAndServe() 方法，该方法分为两个部分：监听端口 (ln, err := net.Listen("tcp", addr)) 和 提供服务 (srv.Serve(ln))。

   ```go
   // return server.ListenAndServe()
   func (srv *Server) ListenAndServe() error {
   ...
   addr := srv.Addr
   if addr == "" {
       addr = ":http"
   }
   ln, err := net.Listen("tcp", addr)
   if err != nil {
       return err
   }
   return srv.Serve(ln)
   }
   ```

3. 调用 net.Listen() 方法监听端口，返回一个 Listener 接口。

   ```go
   // ln, err := net.Listen("tcp", addr)
   func Listen(network, address string) (Listener, error) {
   var lc ListenConfig
   return lc.Listen(context.Background(), network, address)
   }
   
   type Listener interface {
       // Addr 返回该接口的网络地址
       Addr() Addr
       // Accept 等待并返回下一个连接到该接口的连接
       Accept() (c Conn, err error)
       // Close 关闭该接口，并使任何阻塞的 Accept 操作都会不再阻塞并返回错误。
       Close() error
   }
   ```

   > Listener 是一个用于面向流的网络协议的公用的网络监听器接口。多个线程可能会同时调用一个 Listener 的方法。

4. 将上一步返回的 Listener 接口 ln 作为参数，调用 Sever.Serve() 方法：

   启动一个 for 循环，在循环体中调用 Listener 接口的 Accept() 方法，等待接收客户端发出的请求。

   ```go
   // return srv.Serve(ln)
   func (srv *Server) Serve(l net.Listener) error {
   defer l.Close()
   var tempDelay time.Duration // how long to sleep on accept failure
   for {
           // 等待接收客户端发出的请求，返回一个接口类型 Conn 的实例 rw
       rw, err := l.Accept()
       ...
           c := srv.newConn(rw)
       ...
       go c.serve(connCtx)
   }
   ```

5. c := srv.newConn(rw) 对客户端的每个请求实例化一个 conn（此处的 c 为结构体类型 conn，上文中的 rw 为接口类型 Conn, 注意大小写）, 即创建了一个连接实体。这个连接实体里面保存了该次请求的信息。

   ```go
   func (srv *Server) newConn(rwc net.Conn) *conn {
   c := &conn{
       server: srv,
       rwc:    rwc,
   }
   if debugServerConnections {
       c.rwc = newLoggingConn("server", c.rwc)
   }
   return c
   }
   ```

6. go c.serve(connCtx) 调用 conn.Serve() 方法开启一个 goroutine 为这个连接进行服务。这体现了 go 的支持高并发，用户的每一次请求都是在一个新的 goroutine 去服务，相互不影响。

   ```go
   func (c *conn) serve(ctx context.Context) {
       ...
       for {
           // 读取每个请求的内容
           w, err := c.readRequest(ctx)
           if c.r.remain != c.server.initialReadLimitSize() {
               // If we read any bytes off the wire, we're active.
               c.setState(c.rwc, StateActive)
           }
    ...
    // 处理请求
    serverHandler{c.server}.ServeHTTP(w, w.req)
    ....
    }
   ```

7. Serve() 方法分为两个部分，分别是：w, err := c.readRequest(ctx) 和 serverHandler{c.server}.ServeHTTP(w, w.req)。首先调用 conn.readRequest() 方法对请求进行解析，返回了一个 \*response 类型的响应 w：

   ```go
   // w, err := c.readRequest(ctx)
   func (c *conn) readRequest(ctx context.Context) (w *response, err error) {
       ...
           w = &response{
           conn:          c,
           cancelCtx:     cancelCtx,
           req:           req,
           reqBody:       req.Body,
        ...
        }
    ...
   }
   ```

   结构体类型 response 代表对一个 HTTP 请求的响应。由于该结构体首字母小写，因此无法在包外调用。而 \*response 实现了 ResponseWriter 的全部方法。因此我们可以将 \*response 当作一个 ResponseWriter 传入相应的函数，从而通过引用传递来操作 response。

   ```go
   type ResponseWriter interface {
       Header() Header
       WriteHeader(int)
       Write([]byte) (int, error)
   }
   
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

8. 随后在 Serve() 方法中调用 serverHandler 结构体类型的 ServeHTTP 方法，并将 w 和 w.req 分别作为响应和请求传入：

   ```go
   serverHandler{c.server}.ServeHTTP(w, w.req)
   ```

   serverHandler 结构体只有一个字段 srv，它是一个指向 Server 结构体类型的指针。调用 ServeHTTP 方法时，会首先判断之前实例化得到的 Server 中的 Handler 字段是否为空。若为空，则设置 handler = DefaultServeMux。

   ```go
   type serverHandler struct {
    srv *Server
   }
   
   func (sh serverHandler) ServeHTTP(rw ResponseWriter, req *Request) {
    handler := sh.srv.Handler
    if handler == nil {
        handler = DefaultServeMux
    }
    ...
    handler.ServeHTTP(rw, req)
   }
   ```

   由于本例中调用 http.ListenAndServe(":9090", nil) 时，传入 Handler 的参数值为 nil，因此设置 handler = DefaultServeMux。上文提到，DefaultServeMux 是一个指向 ServeMux 的指针，而 \*ServeMux 的 ServeHTTP 方法为：

   ```go
   \\ handler.ServeHTTP(rw, req)
   func (mux *ServeMux) ServeHTTP(w ResponseWriter, r *Request) {
    ...
    h, _ := mux.Handler(r)
    h.ServeHTTP(w, r)
   }
   ```

   其中，h, _ := mux.Handler(r) 内部还调用了 ServeMux.handler 方法，其作用是调用 mux.match() 根据 path 在 DefaultServeMux 中寻找对应的 Handler 函数。由于我们已经通过在主函数中的第一行代码中调用 HandleFunc() 向 DefaultServeMux 中注册了请求"/"的路由规则（映射关系），因此当请求 uri 为"/"时，返回的变量 h 便是 Handler 函数 sayhelloName。

   ```go
   func (mux *ServeMux) Handler(r *Request) (h Handler, pattern string) {
    ...
    host := stripHostPort(r.Host)
    ...
    return mux.handler(host, r.URL.Path)
   }
   
   // handler is the main implementation of Handler.
   func (mux *ServeMux) handler(host, path string) (h Handler, pattern string) {
    ...
    if mux.hosts {
        h, pattern = mux.match(host + path)
    }
    if h == nil {
        h, pattern = mux.match(path)
    }
    if h == nil {
        h, pattern = NotFoundHandler(), ""
    }
    return
   }
   ```

9. 因此上述代码中的 h.ServeHTTP(w, r) 就是调用了 Handler 函数 sayhelloName 的 ServeHttp 方法：

   ```go
   func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
       f(w, r)
   }
   ```

   > ServeHTTP 方法有 2 个参数，第二个参数是 *Request ，该对象包含了该 HTTP 请求的所有的信息，比如请求地址、Header 和 Body 等信息；第一个参数是 ResponseWriter ，利用 ResponseWriter 可以构造针对该请求的响应。

   这个方法内部其实就是调用 sayhelloName 本身。

## 实现流程图

![3.3.illustrator](https://cdn.jsdelivr.net/gh/koktlzz/NoteImg@main/3.3.illustrator.png)

## 测试结果

### 无参数发起请求

- 客户端

![20201225020323](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20201225020323.png)

- 服务端

![20201225020402](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20201225020402.png)

### 带参数发起请求

- 客户端

![20201225020554](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20201225020554.png)

- 服务端

![20201226200222](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20201226200222.png)
