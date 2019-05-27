---
title: golang开发HTTP，一篇文章就够了
date: 2019-03-14
tags: ["golang","http"]
ncategories: ["golang"]
---

# 创建HTTP服务
```golang
func main(){
    if err := http.ListenAndServe(":12345",nil); err != nil{
        fmt.Println("start http server fail:",err)
    }
}
```
没有添加业务逻辑，所有访问都会 404 page not found

# 添加 http.Handler

就是添加页面业务处理
```golang
func main() {
	http.HandleFunc("/", func(w http.ResponseWriter,r *http.Request) {
		w.Write([]byte("hello world"))
	})
	http.HandleFunc("/abc", func(w http.ResponseWriter,r *http.Request) {
		w.Write([]byte("hello world ,abc"))
	})
	if err := http.ListenAndServe(":12345", nil);err != nil{
		fmt.Println("start http server faild:",err)
	}
}
```
golang自带的路由功能比较弱
上面的代码除了请求路劲`/abc` 会匹配到第二个handler，其他的都匹配到了第一个`/`
所以我们需要自己实现一个路由
```golang
type MyHandler struct{}
func (mh MyHandler) ServeHTTP(w http.ResponseWriter,r *http.Request){
	if r.URL.Path =="/hello"{
		w.Write([]byte("hello page"))
		return //这里的return是要加的，不然下面的代码也会执行了
	}
	if r.URL.Path == "/world"{
		w.Write([]byte("world page"))
		return 
	}
	w.Write([]byte("root page"))
	// 可以继续写自己的路由匹配规则
}
func main() {

	http.Handle("/", MyHandler{})  //note： 这里不是刚才的http.HandleFunc()了
	if err := http.ListenAndServe(":12345", nil);err != nil{
		fmt.Println("start http server faild:",err)
	}
}

```
或者可以更简单一些
```golang
func main() {
	if err := http.ListenAndServe(":12345", MyHandler{});err != nil{
		fmt.Println("start http server faild:",err)
	}
}
```

# http.ServeMux 路由

golang的`net/http`包给我们提供了一个路由`ServeMux`,h上面的方法`http.HandleFunc()`和`http.Handle()` 其实就是把路由规则注册到了默认的`ServeMux`上了，就是DefaultServeMux。我们可以看看源码：
```golang

// DefaultServeMux is the default ServeMux used by Serve.
var DefaultServeMux = &defaultServeMux

var defaultServeMux ServeMux
```
```golang
// Handle registers the handler for the given pattern
// in the DefaultServeMux.
// The documentation for ServeMux explains how patterns are matched.
func Handle(pattern string, handler Handler) { DefaultServeMux.Handle(pattern, handler) }

// HandleFunc registers the handler function for the given pattern
// in the DefaultServeMux.
// The documentation for ServeMux explains how patterns are matched.
func HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	DefaultServeMux.HandleFunc(pattern, handler)
}

```
## 自己写一个ServeMux
```golang
func testhandler(w http.ResponseWriter,r *http.Request){
	w.Write([]byte("this is the ServeMux handler"))
}
func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/", testhandler)
	
	if err := http.ListenAndServe(":12345", mux);err != nil{
		fmt.Println("start http server faild:",err)
	}
}

```
但是由于golang自带的ServerMux路由规则过于简单，实践中一般都不会用。
推荐：https://github.com/gorilla/mux

# http.Handler/http.HandlerFunc 中间件
golang的http处理过程不止一个http.handlerFunc,而是一组http.handleFunc
```golang
func handler1(w http.ResponseWriter,r *http.Request){
	w.Write([]byte("handler1"))
}
func handler2(w http.ResponseWriter,r *http.Request){
	w.Write([]byte("handler2"))
}
func makeHandler(handlers ...http.HandlerFunc)http.HandlerFunc{
	return func(w http.ResponseWriter,r *http.Request) {
		for _,hander := range handlers{
			hander(w,r)
		}
	}
}
func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/", makeHandler(handler1,handler2))
	if err := http.ListenAndServe(":12345", mux);err != nil{
		fmt.Println("start http server faild:",err)
	}
}
```
我们访问 `http://localhost:12345/`,就会看到 handler1和handler2都打印出来了。

框架https://github.com/urfave/negroni 基本上就是这种模式。

# request
一个http请求主要是客户端发过来的 `*http.Request` ，和返回给客户端的`http.reponseWriter`
```golang
func handlerFunc(w http.ResponseWriter, r *http.Request){
	fmt.Println(r.Method) //GET
	fmt.Println(r.URL) ///abc
	fmt.Println(r.URL.Path) //abc
	fmt.Println(r.RemoteAddr) //[::1]:62639
	fmt.Println(r.UserAgent()) //Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/72.0.3626.121 Safari/537.36
	fmt.Println(r.Header.Get("Accept")) //text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8
	fmt.Println(r.Cookies()) //[_ga=GA1.1.668973879.1547800734]
}
func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/", handlerFunc)
	if err := http.ListenAndServe(":12345", mux);err != nil{
		fmt.Println("start http server faild:",err)
	}
}

```
更过信息可查看官方文档

# 表单数据
（测试可以用postman测试）
请求传递的表单数据，存储在`*http.Request.Form` 和 `*http.Request.PostForm`中

## get请求
`Get /?name=xuxiaofeng`,获取请求内容并打印到返回内容
```golang
func handlerFunc(w http.ResponseWriter, r *http.Request){
	name := r.FormValue("name")
	w.Write([]byte(name))
}
func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/", handlerFunc)
	if err := http.ListenAndServe(":12345", mux);err != nil{
		fmt.Println("start http server faild:",err)
	}
}
```
`http://localhost:12345/?name=xuxiaofeng`,就可以看到打印的内容

## post表单请求
```golang
func handlerFunc(w http.ResponseWriter, r *http.Request){
	name1 := r.FormValue("name")  //FormValue(),如果name存在，也是获取第一个值【查源码就知道了】
	name2 := r.PostFormValue("name") // PostFormValue()也是同理，name存在是，也是获取第一个值
	name3 := r.Form.Get("name")
	name4 := r.PostForm.Get("name")
	w.Write([]byte(name1+name2+name3+name4))
}
func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/", handlerFunc)
	if err := http.ListenAndServe(":12345", mux);err != nil{
		fmt.Println("start http server faild:",err)
	}
}

```
上面的四个值是相同的,但仅限于是post请求的情况下
如果我们这样请求`http://localhost:12345/?name=abc`,那`r.PostFromValue()` 和 `r.PostForm.Get("name")`都获取不到数据，带post的都获取不到数据。

## 直接操作r.Form和r.PostForm
如果我们想直接操作r.Form
```golang
func handlerFunc(w http.ResponseWriter, r *http.Request){
	fmt.Println(r.Form["name"])
	w.Write([]byte("over"))
}
func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/", handlerFunc)
	if err := http.ListenAndServe(":12345", mux);err != nil{
		fmt.Println("start http server faild:",err)
	}
}
```
我们会发现，`fmt.Println(r.Form["name"])`打印不出东西。看下面
```golang
func handlerFunc(w http.ResponseWriter, r *http.Request){
	// 这里一定要记得 ParseForm，否则 r.Form 是空的
    // 调用 r.FormValue() 的时候会自动执行 r.ParseForm()
	r.ParseForm()
	fmt.Println(r.Form) // map[name:[abc] age:[13]]
	w.Write([]byte("over"))
}
func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/", handlerFunc)
	if err := http.ListenAndServe(":12345", mux);err != nil{
		fmt.Println("start http server faild:",err)
	}
}
```
表单数据存储在 r.Form，是 map[string][]string 类型，即支持一个表单域多个值的情况。r.FormValue() 只获取第一个值

## 表单数据<>结构体
表单数据是简单的 kv 对应，很容易实现 kv 到 结构体的一一对应
这个库 https://github.com/mholt/binding 就是干这个的
```golang
type User struct {
    Id   int
    Name string
}

func (u *User) FieldMap(req *http.Request) binding.FieldMap {
    return binding.FieldMap{
        &u.Id: "user_id",
        &u.Name: binding.Field{
            Form:     "name",
            Required: true,
        },
    }
}

func handle(w http.ResponseWriter, r *http.Request) {
    user := new(User)
    errs := binding.Bind(r, user)
    if errs.Handle(w) {
        return
    }
}

```
# body消息体

>无论表单数据，还是上传的二进制数据，都是保存在 HTTP 的 Body 中的。操作 *http.Request.Body 可以获取到内容。但是注意 *http.Request.Body 是 io.ReadCloser 类型，只能一次性读取完整，**第二次就是空的**。
```golang
func httpHandler(w http.ResponseWriter,r *http.Request){
	body,err := ioutil.ReadAll(r.Body)
	if err != nil {
		fmt.Println("read body error",err)
	}
	fmt.Println(string(body))
	w.Write([]byte("receive request:"))
	w.Write(body)
}
func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/", httpHandler)
	if err := http.ListenAndServe(":12345", mux);err != nil{
		fmt.Println("start http server faild:",err)
	}
}
```
根据HTTP协议，如果请求的`Content-Type: application/x-www-form-urlencoded`,body 中的数据就是类似 abc=123&abc=abc&abc=xyz 格式的数据，也就是常规的 表单数据。这些使用 r.ParseForm() 然后操作 r.Form 处理数据。如果是纯数据，比如文本abcdefg 、 JSON 数据等，你才需要直接操作 Body 的。比如接收 JSON 数据：
```golang
type User struct {
	Name string `json:"name"`
	Age  int    `json:"age"`
}

func httpHandler(w http.ResponseWriter, r *http.Request) {
	body, err := ioutil.ReadAll(r.Body)
	if err != nil {
		log.Panicf("read body failed:%v", err)
	}
	var u User
	json.Unmarshal(body, &u)
	fmt.Printf("%#v\n", u)
	w.Write([]byte("receive data:"))
	w.Write([]byte(string(body)))
}
func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/", httpHandler)
	if err := http.ListenAndServe(":12345", mux); err != nil {
		fmt.Println("start http server faild:", err)
	}
}
```
# 上传文件
上传的文件经过 Go 的解析保存在 *http.Request.MultipartForm 中,通过 r.FormFile() 去获取收到的文件信息和数据流，并处理：
服务端文件
```golang
func httpHandlerFunc(w http.ResponseWriter, r *http.Request) {
	// 这里一定要记得 r.ParseMultipartForm(), 否则 r.MultipartForm 是空的
	// 调用 r.FormFile() 的时候会自动执行 r.ParseMultipartForm()
	err := r.ParseMultipartForm(32 << 20)
	if err != nil {
		fmt.Println("ParseMultipartForm error:%v", err)
		w.WriteHeader(500)
		return
	}
	// 写明缓冲的大小。如果超过缓冲，文件内容会被放在临时目录中，而不是内存。过大可能较多占用内存，过小可能增加硬盘 I/O
	// FormFile() 时调用 ParseMultipartForm() 使用的大小是 32 << 20，32MB
	srcfile, srcfileheader, err := r.FormFile("file") // file 是上传表单域的名字
	if err != nil {
		fmt.Println("get upload file fail:", err)
		w.WriteHeader(500)
		return
	}
	defer srcfile.Close() // 此时上传内容的 IO 已经打开，需要手动关闭！！
	// fileHeader 有一些文件的基本信息
	fmt.Println(srcfileheader.Header.Get("Content-Type"))
	// / 打开目标地址，把上传的内容存进去
	dstfile, err := os.OpenFile("./test.jpg", os.O_CREATE|os.O_RDWR|os.O_TRUNC, 0666)
	if err != nil {
		fmt.Println("save upload file fail:", err)
		w.WriteHeader(500)
		return
	}
	_, err = io.Copy(dstfile, srcfile)
	if err != nil {
		fmt.Println("copy to dstfile error:", err)
	}
	w.Write([]byte("上传文件完成"))
}
func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/", httpHandlerFunc)
	http.ListenAndServe(":12345", mux)
}

```
用postman发现不能测试上传文件，就有golang写了一个上传文件的客户端
教程看这里：https://matt.aimonetti.net/posts/2013/07/01/golang-multipart-file-upload-example/
```golang
func newUploadFileRequest(uri string, params map[string]string, paramName, path string) (*http.Request, error) {
	srcfile, err := os.Open(path)
	if err != nil {
		return nil, err
	}
	defer srcfile.Close()
	body := &bytes.Buffer{}
	writer := multipart.NewWriter(body) // writing to body
	part, err := writer.CreateFormFile(paramName, filepath.Base(path))
	if err != nil {
		return nil, err
	}
	io.Copy(part, srcfile)
	for k, v := range params {
		writer.WriteField(k, v)
	}
	err = writer.Close()
	if err != nil {
		return nil, err
	}
	req, err := http.NewRequest("POST", uri, body)
	req.Header.Set("Content-Type", writer.FormDataContentType())
	return req, nil

}
func main() {
	//path,_ := os.Getwd()
	//path += "test.png"
	extraParams := map[string]string{
		"title":       "My Document",
		"author":      "Matt Aimonetti",
		"description": "a picture for test file upload",
	}
	request, err := newUploadFileRequest("http://localhost:12345", extraParams, "file", "./111.jpg")
	if err != nil {
		log.Fatal(err)
	}
	client := &http.Client{}
	resp, err := client.Do(request)
	if err != nil {
		log.Fatal(err)
	} else {
		body := &bytes.Buffer{}
		_, err := body.ReadFrom(resp.Body)
		if err != nil {
			log.Fatal(err)
		}
		resp.Body.Close()
		fmt.Println(resp.StatusCode)
		fmt.Println(resp.Header)
		fmt.Println(body)
	}

}
```
上传之后就会发现根目录下多了个`test.jpg`文件，打开一看就是我们刚才上传的那个图片。
## 上传多个文件
上面的例子中，`r.FromFile("file")`只能获取单个文件（也就是第一个文件），如果想要获取多个文件就需要直接操作`r.MultipartForm`了。
```golang
func httpHandler(w http.ResponseWriter, r *http.Request) {
	r.ParseMultipartForm(32 << 20)
	var err error
	var file multipart.File
	for _, fileheader := range r.MultipartForm.File["file"] {
		if file, err := fileheader.Open(); err != nil {
			fmt.Println("open file error:", err)
			continue
		}
		SaveFile(file) // 仿照上面单个文件的操作，保存 file
		file.Close()   // 操作结束一定要 Close，for 循环里不要用 defer file.Close()
		file = nil
		w.Write([]byte("save:" + fileheader.Filename))
	}

}
func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/", httpHandler)
	if err := http.ListenAndServe(":12345", mux); err != nil {
		fmt.Println("start http server faild:", err)
	}
}
```
#  responseWriter
http.ResponseWriter 是一个接口，你可以根据接口，添加一些自己需要的行为：
```
type ResponseWriter interface {
    Header() Header // 添加返回头信息
    Write([]byte) (int, error) // 添加返回的内容
    WriteHeader(int) // 设置返回的状态码
}
```
w.WriteHeader() 是一次性的，不能重复设置状态码，否则会有提示信息：
```
func HttpHandle(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(200) // 设置成功
    w.WriteHeader(404) // 提示：http: multiple response.WriteHeader calls 
    w.WriteHeader(503) // 提示：http: multiple response.WriteHeader calls 
}
```
而且需要在 w.Write() 之前设置 w.WriteHeader()，否则是 200。（要先发送状态码，再发送内容）
```
func HttpHandle(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("Hello World"))
    w.WriteHeader(404) // 提示：http: multiple response.WriteHeader calls，因为 w.Write() 已发布 HTTP 200
}
```
http.ResponseWriter 接口过于简单，实际使用会自己实现 ResponseWriter 来使用，比如获取返回的内容：
```golang
type MyResponseWriter struct {
	http.ResponseWriter
	bodyBytes *bytes.Buffer
}

// 覆写 http.ResponseWriter 的方法
func (mrw MyResponseWriter) Write(body []byte) (int, error) {
	mrw.bodyBytes.Write(body)             // 记录下返回的内容
	return mrw.ResponseWriter.Write(body) // 这里才是http真正的返回
}

// Body 获取记录的返回的内容，这个是自己添加的方法
func (mrw MyResponseWriter) Body() []byte {
	return mrw.bodyBytes.Bytes()
}

func httpHandler(w http.ResponseWriter, r *http.Request) {
	mrw := MyResponseWriter{
		ResponseWriter: w,
		bodyBytes:      bytes.NewBuffer(nil),
	}
	mrw.Header().Set("Content-Type", "text/html") // 要输出HTML记得加头信息
	mrw.Write([]byte("<h1>the game of power!</h1>"))
	mrw.Write([]byte("the server receive some data"))
	fmt.Println(string(mrw.Body()))
}
func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/", httpHandler)
	if err := http.ListenAndServe(":12345", mux); err != nil {
		fmt.Println("start http server faild:", err)
	}
}
```
# 输出其他内容

`net/http` 提供一些便利的方法可以输出其他的内容，比如 cookie:

```
func HttpHandle(w http.ResponseWriter, r *http.Request) {
    c := &http.Cookie{
        Name:     "abc",
        Value:    "xyz",
        Expires:  time.Now().Add(1000 * time.Second),
        MaxAge:   1000,
        HttpOnly: true,
    }
    http.SetCookie(w, c)
}

```
比如服务端返回下载文件：
```
func HttpHandle(w http.ResponseWriter, r *http.Request) {
    http.ServeFile(w, r, "download.txt")
}
```
或者是生成的数据流，比如验证码，当作文件返回：
```
func HttpHandle(w http.ResponseWriter, r *http.Request) {
    captchaImageBytes := createCaptcha() // 假设生成验证码的函数，返回 []byte
    buf := bytes.NewReader(captchaImageBytes)
    http.ServeContent(w, r, "captcha.png", time.Now(), buf)
}
```
还有一些状态码的直接操作：
```
func HttpHandle(w http.ResponseWriter, r *http.Request) {
    http.Redirect(w, r, "/abc", 302)
}
func HttpHandle2(w http.ResponseWriter, r *http.Request) {
    http.NotFound(w, r)
}
```
返回 JSON, XML 和 渲染模板的内容等的代码例子，可以参考 [HTTP Response Snippets for Go](http://www.alexedwards.net/blog/golang-response-snippets)。

# Context
Go 1.7 添加了 context 包，用于传递数据和做超时、取消等处理。*http.Request 添加了 r.Context() 和 r.WithContext() 来操作请求过程需要的 context.Context 对象。
## 传递数据
context 可以在 http.HandleFunc 之间传递数据：
```golang
func httpHandler(w http.ResponseWriter, r *http.Request) {
	ctx := context.WithValue(r.Context(), "name", "权利的游戏") // 写入 string 到 context
	handler2(w, r.WithContext(ctx))                        // 传递给下一个 handleFunc
}
func handler2(w http.ResponseWriter, r *http.Request) {
	name, ok := r.Context().Value("name").(string) // 取出的 interface 需要推断到 string
	if !ok {
		name = ""
	}
	w.Write([]byte("convert name:" + name))
}

func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/", httpHandler)
	if err := http.ListenAndServe(":12345", mux); err != nil {
		fmt.Println("start http server faild:", err)
	}
}

```
## 处理超时的请求
利用 context.WithTimeout 可以创建会超时结束的 context，用来处理业务超时的情况：
```
func HttpHandle(w http.ResponseWriter, r *http.Request) {
    ctx, cancelFn := context.WithTimeout(r.Context(), 1*time.Second)

    // cancelFn 关掉 WithTimeout 里的计时器
    // 如果 ctx 超时，计时器会自动关闭，但是如果没有超时就执行到 <-resCh,就需要手动关掉
    defer cancelFn()

    // 把业务放到 goroutine 执行， resCh 获取结果
    resCh := make(chan string, 1)
    go func() {
        // 故意写业务超时
        time.Sleep(5 * time.Second)
        resCh <- r.FormValue("abc")
    }()

    // 看 ctx 超时还是 resCh 的结果先到达
    select {
    case <-ctx.Done():
        w.WriteHeader(http.StatusGatewayTimeout)
        w.Write([]byte("http handle is timeout:" + ctx.Err().Error()))
    case r := <-resCh:
        w.Write([]byte("get: abc = " + r))
    }
}
```
## 带 context 的中间件
Go 的很多 HTTP 框架使用 `context` 或者自己定义的 `Context` 结果作为 `http.Handler` 中间件之间数据传递的媒介，比如 [xhandler](https://github.com/rs/xhandler):

# Hijack
一些时候需要直接操作 Go 的 HTTP 连接时，使用 Hijack() 将 HTTP 对应的 TCP 取出。连接在 Hijack() 之后，HTTP 的相关操作会受到影响，连接的管理需要用户自己操作，而且例如 w.Write([]byte) 不会返回内容，需要操作 Hijack() 后的 *bufio.ReadWriter。
```golang
func hiJackHandle(w http.ResponseWriter, r *http.Request) {
	hj, ok := w.(http.Hijacker)
	if !ok {
		w.Write([]byte("the server do not support hijacker"))
		return
	}
	conn, buf, err := hj.Hijack() // 需要手动关闭连接
	if err != nil {
		fmt.Println("Hijack error:", err)
	}
	defer conn.Close()
	w.Write([]byte("继续用w返回内容")) //error:http: response.Write on hijacked connection
	// 返回内容需要
	buf.WriteString("用buf返回的hello world")
	buf.Flush()
}

func main() {
	http.HandleFunc("/", hiJackHandle)
	if err := http.ListenAndServe(":12345", nil); err != nil {
		fmt.Println("start http server faild:", err)
	}
}
```
`Hijack` 主要看到的用法是对 HTTP 的 Upgrade 时在用，比如从 HTTP 到 Websocket 时，[golang.org/x/net/websocket](https://github.com/golang/net/blob/master/websocket/server.go#L73):
```golang
func (s Server) serveWebSocket(w http.ResponseWriter, req *http.Request) {
    rwc, buf, err := w.(http.Hijacker).Hijack()
    if err != nil {
        panic("Hijack failed: " + err.Error())
    }
    // The server should abort the WebSocket connection if it finds
    // the client did not send a handshake that matches with protocol
    // specification.
    defer rwc.Close()
    conn, err := newServerConn(rwc, buf, req, &s.Config, s.Handshake)
    if err != nil {
        return
    }
    if conn == nil {
        panic("unexpected nil conn")
    }
    s.Handler(conn)
}
```
# http.Server 的使用细节
上面所有的代码我都是用的 http.ListenAndServe 来启动 HTTP 服务。实际上执行这个过程的 *http.Server 这个结构。有些时候我们不是使用默认的行为，会给 *http.Server 定义更多的内容。

http.ListenAndServe 默认的 *http.Server 是没有超时设置的。一些场景下你必须设置超时，否则会遇到太多连接句柄的问题：
```
func main() {
    server := &http.Server{
        Handler:      MyHandler{}, // 使用实现 http.Handler 的结构处理 HTTP 数据
        ReadTimeout:  10 * time.Second,
        WriteTimeout: 10 * time.Second,
    }
    // 监听 TCP 端口，把监听器交给 *http.Server 使用
    ln, err := net.Listen("tcp", ":12345")
    if err != nil {
        panic("listen :12345 fail:" + err.Error())
    }
    if err = server.Serve(ln); err != nil {
        fmt.Println("start http server fail:", err)
    }
}
```
有朋友用 Beego 的时候希望同时监听两个端口提供一样数据操作的 HTTP 服务。这个需求就可以利用 *http.Server 来实现：
```
import (
    "fmt"
    "net/http"

    "github.com/astaxie/beego"
    "github.com/astaxie/beego/context"
)

func main() {
    beego.Get("/", func(ctx *context.Context) {
        ctx.WriteString("abc")
    })
    go func() { // server 的 ListenAndServe 是阻塞的，应该在另一个 goroutine 开启另一个server
        server2 := &http.Server{
            Handler: beego.BeeApp.Handlers, // 使用实现 http.Handler 的结构处理 HTTP 数据
            Addr:    ":54321",
        }
        if err := server2.ListenAndServe(); err != nil {
            fmt.Println("start http server2 fail:", err)
        }
    }()
    server1 := &http.Server{
        Handler: beego.BeeApp.Handlers, // 使用实现 http.Handler 的结构处理 HTTP 数据
        Addr:    ":12345",
    }
    if err := server1.ListenAndServe(); err != nil {
        fmt.Println("start http server1 fail:", err)
    }
}
```
这样访问 http://localhost:12345 和 http://localhost:54321 都可以看到返回 abc 的内容。
# HTTPS
随着互联网安全的问题日益严重，许多的网站开始使用 HTTPS 提供服务。Go 创建一个 HTTPS 服务是很简便的：

```
import (
    "fmt"
    "net/http"
)

func handler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Hello, HTTPS Server")
}

func main() {
    http.HandleFunc("/", handler)
    http.ListenAndServeTLS(":12345",
        "server.crt",
        "server.key", nil)
}

```

`ListenAndServeTLS` 新增了两个参数 `certFile` 和 `keyFile`。HTTPS的数据传输是加密的。实际使用中，HTTPS利用的是对称与非对称加密算法结合的方式，需要加密用的公私密钥对进行加密，也就是 `server.crt` 和 `server.key` 文件。具体的生成可以阅读 `openssl` 的文档。

关于 Go 和 HTTPS 的内容，可以阅读 **Tony Bai** 的 [Go 和 HTTPS](http://tonybai.com/2015/04/30/go-and-https/)。
### 总结

Go 的 `net/http` 包为开发者提供很多便利的方法的，可以直接开发不复杂的 Web 应用。如果需要复杂的路由功能，及更加集成和简便的 HTTP 操作，推荐使用一些 Web 框架。

各种 Web 框架 : [awesome-go#web-frameworks](https://github.com/avelino/awesome-go#web-frameworks)

参考 http://fuxiaohei.me/2016/9/20/go-and-http-server.html

# 另外参考
## Server.Serve()
上面的文章中没有介绍这个用法
```golang
func (srv *Server) Serve(l net.Listener) error{}
```
用法及其中原理可以参考：https://cizixs.com/2016/08/17/golang-http-server-side/
