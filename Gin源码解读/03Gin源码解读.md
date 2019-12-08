<!-- TOC -->

- [简介](#简介)
- [Context 初始化](#context-初始化)
- [Context 之请求参数获取](#context-之请求参数获取)
- [Context 之模型绑定和验证](#context-之模型绑定和验证)
- [Context 之响应](#context-之响应)
- [Context 之高级响应](#context-之高级响应)
- [Context 之内容协商](#context-之内容协商)
- [总结](#总结)

<!-- /TOC -->

## 简介

Gin 源码解读, 基于 [v1.5.0 ](https://github.com/gin-gonic/gin/tree/v1.5.0) 版本.

## Context 初始化

Context 是 Gin 中很重要的一个部分, 先看一下注释是怎么说的.

```go
// Context is the most important part of gin. It allows us to pass variables between middleware,
// manage the flow, validate the JSON of a request and render a JSON response for example.
type Context struct {
	writermem responseWriter
	Request   *http.Request
	Writer    ResponseWriter

	Params   Params
	handlers HandlersChain
	index    int8
	fullPath string

	engine *Engine

	// Keys is a key/value pair exclusively for the context of each request.
	Keys map[string]interface{}

	// Errors is a list of errors attached to all the handlers/middlewares who used this context.
	Errors errorMsgs

	// Accepted defines a list of manually accepted formats for content negotiation.
	Accepted []string

	// queryCache use url.ParseQuery cached the param query result from c.Request.URL.Query()
	queryCache url.Values

	// formCache use url.ParseQuery cached PostForm contains the parsed form data from POST, PATCH,
	// or PUT body parameters.
	formCache url.Values
}
```

注释中说到, Context 用于中间件中的变量传递, 流程控制, 验证请求的 JSON 格式以及返回 JSON 响应等.

Context 是在每次接受请求的时候初始化的:

```go
// ServeHTTP conforms to the http.Handler interface.
func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	c := engine.pool.Get().(*Context)
	c.writermem.reset(w)
	c.Request = req
	c.reset()

	engine.handleHTTPRequest(c)

	engine.pool.Put(c)
}
```

里面用到了 `sync.Pool`, `sync.Pool` 适用于缓存已分配但未使用的 items, 以便后续重用, 并减轻垃圾回收的压力.

```go
type Pool struct {

    // New optionally specifies a function to generate
    // a value when Get would otherwise return nil.
    // It may not be changed concurrently with calls to Get.
    New func() interface{}
    // contains filtered or unexported fields
}
```

`sync.Pool` 需要实现一个名为 `New` 的方法, 这其实在初始化 `Engine` 的时候就已经完成了.

```go
// New returns a new blank Engine instance without any middleware attached.
// By default the configuration is:
// - RedirectTrailingSlash:  true
// - RedirectFixedPath:      false
// - HandleMethodNotAllowed: false
// - ForwardedByClientIP:    true
// - UseRawPath:             false
// - UnescapePathValues:     true
func New() *Engine {
	debugPrintWARNINGNew()
	engine := &Engine{
		RouterGroup: RouterGroup{
			Handlers: nil,
			basePath: "/",
			root:     true,
		},
		FuncMap:                template.FuncMap{},
		RedirectTrailingSlash:  true,
		RedirectFixedPath:      false,
		HandleMethodNotAllowed: false,
		ForwardedByClientIP:    true,
		AppEngine:              defaultAppEngine,
		UseRawPath:             false,
		UnescapePathValues:     true,
		MaxMultipartMemory:     defaultMultipartMemory,
		trees:                  make(methodTrees, 0, 9),
		delims:                 render.Delims{Left: "{{", Right: "}}"},
		secureJsonPrefix:       "while(1);",
	}
	engine.RouterGroup.engine = engine
	engine.pool.New = func() interface{} {
		return engine.allocateContext()
	}
	return engine
}

func (engine *Engine) allocateContext() *Context {
	return &Context{engine: engine}
}
```

由此, 我们已经知道了 `Context` 是如何初始化的了.

## Context 之请求参数获取

Context 肩负着很重要的使命, 所有的处理函数的唯一参数就是 Context.

```go
// HandlerFunc defines the handler used by gin middleware as return value.
type HandlerFunc func(*Context)
```

在探究中间件的原理时, 我们已经看过了流程控制, 即 `context.Next()` 方法:

```go
// Next should be used only inside middleware.
// It executes the pending handlers in the chain inside the calling handler.
// See example in GitHub.
func (c *Context) Next() {
	c.index++
	for c.index < int8(len(c.handlers)) {
		c.handlers[c.index](c)
		c.index++
	}
}
```

接着看一下如何获取请求参数, 比如 URL 中的参数, GET 中的 query, 或者是 POST 中的 data.

```go
// However, this one will match /user/john/ and also /user/john/send
// If no other routers match /user/john, it will redirect to /user/john/
router.GET("/user/:name/*action", func(c *gin.Context) {
  name := c.Param("name")
  action := c.Param("action")
  message := name + " is " + action
  c.String(http.StatusOK, message)
})

// Query string parameters are parsed using the existing underlying request object.
// The request responds to a url matching:  /welcome?firstname=Jane&lastname=Doe
router.GET("/welcome", func(c *gin.Context) {
  firstname := c.DefaultQuery("firstname", "Guest")
  lastname := c.Query("lastname") // shortcut for c.Request.URL.Query().Get("lastname")

  c.String(http.StatusOK, "Hello %s %s", firstname, lastname)
})

router.POST("/form_post", func(c *gin.Context) {
  message := c.PostForm("message")
  nick := c.DefaultPostForm("nick", "anonymous")

  c.JSON(200, gin.H{
    "status":  "posted",
    "message": message,
    "nick":    nick,
  })
})

router.POST("/post", func(c *gin.Context) {

  ids := c.QueryMap("ids")
  names := c.PostFormMap("names")

  fmt.Printf("ids: %v; names: %v", ids, names)
})
```

上面的示例来自官方文档, 看一下其中涉及到的方法是如何实现的.

```go
func (c *Context) Param(key string) string {
	return c.Params.ByName(key)
}

func (c *Context) Query(key string) string {
	value, _ := c.GetQuery(key)
	return value
}

func (c *Context) DefaultQuery(key, defaultValue string) string {
	if value, ok := c.GetQuery(key); ok {
		return value
	}
	return defaultValue
}

func (c *Context) GetQuery(key string) (string, bool) {
	if values, ok := c.GetQueryArray(key); ok {
		return values[0], ok
	}
	return "", false
}

func (c *Context) getQueryCache() {
	if c.queryCache == nil {
		c.queryCache = c.Request.URL.Query()
	}
}

func (c *Context) GetQueryArray(key string) ([]string, bool) {
	c.getQueryCache()
	if values, ok := c.queryCache[key]; ok && len(values) > 0 {
		return values, true
	}
	return []string{}, false
}

func (c *Context) PostForm(key string) string {
	value, _ := c.GetPostForm(key)
	return value
}

func (c *Context) DefaultPostForm(key, defaultValue string) string {
	if value, ok := c.GetPostForm(key); ok {
		return value
	}
	return defaultValue
}

func (c *Context) GetPostForm(key string) (string, bool) {
	if values, ok := c.GetPostFormArray(key); ok {
		return values[0], ok
	}
	return "", false
}

func (c *Context) PostFormArray(key string) []string {
	values, _ := c.GetPostFormArray(key)
	return values
}

func (c *Context) getFormCache() {
	if c.formCache == nil {
		c.formCache = make(url.Values)
		req := c.Request
		if err := req.ParseMultipartForm(c.engine.MaxMultipartMemory); err != nil {
			if err != http.ErrNotMultipart {
				debugPrint("error on parse multipart form array: %v", err)
			}
		}
		c.formCache = req.PostForm
	}
}

func (c *Context) GetPostFormArray(key string) ([]string, bool) {
	c.getFormCache()
	if values := c.formCache[key]; len(values) > 0 {
		return values, true
	}
	return []string{}, false
}

```

从上面的代码可以看出 `GetQueryArray` 和 `GetPostFormArray` 的实现非常相似, 都使用内部缓存.

```go
func (c *Context) QueryMap(key string) map[string]string {
	dicts, _ := c.GetQueryMap(key)
	return dicts
}

func (c *Context) GetQueryMap(key string) (map[string]string, bool) {
	c.getQueryCache()
	return c.get(c.queryCache, key)
}

func (c *Context) PostFormMap(key string) map[string]string {
	dicts, _ := c.GetPostFormMap(key)
	return dicts
}

func (c *Context) GetPostFormMap(key string) (map[string]string, bool) {
	c.getFormCache()
	return c.get(c.formCache, key)
}

// get is an internal method and returns a map which satisfy conditions.
func (c *Context) get(m map[string][]string, key string) (map[string]string, bool) {
	dicts := make(map[string]string)
	exist := false
	for k, v := range m {
		if i := strings.IndexByte(k, '['); i >= 1 && k[0:i] == key {
			if j := strings.IndexByte(k[i+1:], ']'); j >= 1 {
				exist = true
				dicts[k[i+1:][:j]] = v[0]
			}
		}
	}
	return dicts, exist
}
```

上面的代码实现了参数的 map 化, 可以看下具体的请求参数, 下面的例子中 ids 就是一个 map, 它有两个 key.

```text
POST /post?ids[a]=1234&ids[b]=hello HTTP/1.1
Content-Type: application/x-www-form-urlencoded

names[first]=thinkerou&names[second]=tianou
```

这不是 HTTP 中定义的内容, 使用的时候必须遵从这种规范, 可能在特定的场景下比较有用.
但一般不太会这么使用, 因为如果是公开的 API, 则其他语言都要实现这种类型的解析.
解析的代码倒是没有什么特别的,

再看一下文件类型的如何实现的, 即 `FormFile`.

```go
// FormFile returns the first file for the provided form key.
func (c *Context) FormFile(name string) (*multipart.FileHeader, error) {
	if c.Request.MultipartForm == nil {
		if err := c.Request.ParseMultipartForm(c.engine.MaxMultipartMemory); err != nil {
			return nil, err
		}
	}
	f, fh, err := c.Request.FormFile(name)
	if err != nil {
		return nil, err
	}
	f.Close()
	return fh, err
}

// MultipartForm is the parsed multipart form, including file uploads.
func (c *Context) MultipartForm() (*multipart.Form, error) {
	err := c.Request.ParseMultipartForm(c.engine.MaxMultipartMemory)
	return c.Request.MultipartForm, err
}

// SaveUploadedFile uploads the form file to specific dst.
func (c *Context) SaveUploadedFile(file *multipart.FileHeader, dst string) error {
	src, err := file.Open()
	if err != nil {
		return err
	}
	defer src.Close()

	out, err := os.Create(dst)
	if err != nil {
		return err
	}
	defer out.Close()

	_, err = io.Copy(out, src)
	return err
}
```

稍微包装了一下 `c.Request.MultipartForm`, 对于单个文件而言更方便些, 保存文件的方法也有了.

## Context 之模型绑定和验证

模型绑定是一个非常有用的能力, 尤其是和验证结合在一起. 处理请求参数时, 一大重点就是验证.

Gin 支持两种类型的绑定, `Must bind` 和 `Should bind`. 请求类型则支持 JSON, XML, YAML 和标准表单绑定.

先来看一下 `Must bind`:

```go
// Bind checks the Content-Type to select a binding engine automatically,
// Depending the "Content-Type" header different bindings are used:
//     "application/json" --> JSON binding
//     "application/xml"  --> XML binding
// otherwise --> returns an error.
// It parses the request's body as JSON if Content-Type == "application/json" using JSON or XML as a JSON input.
// It decodes the json payload into the struct specified as a pointer.
// It writes a 400 error and sets Content-Type header "text/plain" in the response if input is not valid.
func (c *Context) Bind(obj interface{}) error {
	b := binding.Default(c.Request.Method, c.ContentType())
	return c.MustBindWith(obj, b)
}

// BindJSON is a shortcut for c.MustBindWith(obj, binding.JSON).
func (c *Context) BindJSON(obj interface{}) error {
	return c.MustBindWith(obj, binding.JSON)
}

// BindXML is a shortcut for c.MustBindWith(obj, binding.BindXML).
func (c *Context) BindXML(obj interface{}) error {
	return c.MustBindWith(obj, binding.XML)
}

// BindQuery is a shortcut for c.MustBindWith(obj, binding.Query).
func (c *Context) BindQuery(obj interface{}) error {
	return c.MustBindWith(obj, binding.Query)
}

// BindYAML is a shortcut for c.MustBindWith(obj, binding.YAML).
func (c *Context) BindYAML(obj interface{}) error {
	return c.MustBindWith(obj, binding.YAML)
}

// BindHeader is a shortcut for c.MustBindWith(obj, binding.Header).
func (c *Context) BindHeader(obj interface{}) error {
	return c.MustBindWith(obj, binding.Header)
}

// BindUri binds the passed struct pointer using binding.Uri.
// It will abort the request with HTTP 400 if any error occurs.
func (c *Context) BindUri(obj interface{}) error {
	if err := c.ShouldBindUri(obj); err != nil {
		c.AbortWithError(http.StatusBadRequest, err).SetType(ErrorTypeBind) // nolint: errcheck
		return err
	}
	return nil
}

// MustBindWith binds the passed struct pointer using the specified binding engine.
// It will abort the request with HTTP 400 if any error occurs.
// See the binding package.
func (c *Context) MustBindWith(obj interface{}, b binding.Binding) error {
	if err := c.ShouldBindWith(obj, b); err != nil {
		c.AbortWithError(http.StatusBadRequest, err).SetType(ErrorTypeBind) // nolint: errcheck
		return err
	}
	return nil
}
```

从上面的代码可以发现, `MustBindWith` 其实是 `ShouldBindWith` 的包装, 具体内容还是要看 `ShouldBindWith`.
另一点是绑定支持多种数据类型, 比如 BindQuery, BindHeader, BindUri.

```go
// ShouldBindWith binds the passed struct pointer using the specified binding engine.
// See the binding package.
func (c *Context) ShouldBindWith(obj interface{}, b binding.Binding) error {
	return b.Bind(c.Request, obj)
}
```

但实际上 `ShouldBindWith` 也只是调用了 `binding.Binding` 上的方法而言.

```go
// Binding describes the interface which needs to be implemented for binding the
// data present in the request such as JSON request body, query parameters or
// the form POST.
type Binding interface {
	Name() string
	Bind(*http.Request, interface{}) error
}

// BindingBody adds BindBody method to Binding. BindBody is similar with Bind,
// but it reads the body from supplied bytes instead of req.Body.
type BindingBody interface {
	Binding
	BindBody([]byte, interface{}) error
}

// BindingUri adds BindUri method to Binding. BindUri is similar with Bind,
// but it read the Params.
type BindingUri interface {
	Name() string
	BindUri(map[string][]string, interface{}) error
}

// These implement the Binding interface and can be used to bind the data
// present in the request to struct instances.
var (
	JSON          = jsonBinding{}
	XML           = xmlBinding{}
	Form          = formBinding{}
	Query         = queryBinding{}
	FormPost      = formPostBinding{}
	FormMultipart = formMultipartBinding{}
	ProtoBuf      = protobufBinding{}
	MsgPack       = msgpackBinding{}
	YAML          = yamlBinding{}
	Uri           = uriBinding{}
	Header        = headerBinding{}
)
```

上面的代码显示了 `Binding` 接口, 以及实现了 `Binding` 接口的类型, 具体以 JSON 为例, 看一下 `jsonBinding` 是如何实现的.

```go
package binding

import (
	"bytes"
	"fmt"
	"io"
	"net/http"

	"github.com/gin-gonic/gin/internal/json"
)

// EnableDecoderUseNumber is used to call the UseNumber method on the JSON
// Decoder instance. UseNumber causes the Decoder to unmarshal a number into an
// interface{} as a Number instead of as a float64.
var EnableDecoderUseNumber = false

// EnableDecoderDisallowUnknownFields is used to call the DisallowUnknownFields method
// on the JSON Decoder instance. DisallowUnknownFields causes the Decoder to
// return an error when the destination is a struct and the input contains object
// keys which do not match any non-ignored, exported fields in the destination.
var EnableDecoderDisallowUnknownFields = false

type jsonBinding struct{}

func (jsonBinding) Name() string {
	return "json"
}

func (jsonBinding) Bind(req *http.Request, obj interface{}) error {
	if req == nil || req.Body == nil {
		return fmt.Errorf("invalid request")
	}
	return decodeJSON(req.Body, obj)
}

func (jsonBinding) BindBody(body []byte, obj interface{}) error {
	return decodeJSON(bytes.NewReader(body), obj)
}

func decodeJSON(r io.Reader, obj interface{}) error {
	decoder := json.NewDecoder(r)
	if EnableDecoderUseNumber {
		decoder.UseNumber()
	}
	if EnableDecoderDisallowUnknownFields {
		decoder.DisallowUnknownFields()
	}
	if err := decoder.Decode(obj); err != nil {
		return err
	}
	return validate(obj)
}
```

代码也不长, 内部用了自定义的 json 接口, 以便实现可替换的 JSON 编解码.

解码的最后一步是验证, 调用了 `validate` 函数:

```go
func validate(obj interface{}) error {
	if Validator == nil {
		return nil
	}
	return Validator.ValidateStruct(obj)
}
```

由此, 可以引申到验证方面, 看一下是如何结合验证的.

```go
// StructValidator is the minimal interface which needs to be implemented in
// order for it to be used as the validator engine for ensuring the correctness
// of the request. Gin provides a default implementation for this using
// https://github.com/go-playground/validator/tree/v8.18.2.
type StructValidator interface {
	// ValidateStruct can receive any kind of type and it should never panic, even if the configuration is not right.
	// If the received type is not a struct, any validation should be skipped and nil must be returned.
	// If the received type is a struct or pointer to a struct, the validation should be performed.
	// If the struct is not valid or the validation itself fails, a descriptive error should be returned.
	// Otherwise nil must be returned.
	ValidateStruct(interface{}) error

	// Engine returns the underlying validator engine which powers the
	// StructValidator implementation.
	Engine() interface{}
}

// Validator is the default validator which implements the StructValidator
// interface. It uses https://github.com/go-playground/validator/tree/v8.18.2
// under the hood.
var Validator StructValidator = &defaultValidator{}
```

验证器需要实现 `StructValidator` 接口, 看一下默认的验证器的实现.

```go
package binding

import (
	"reflect"
	"sync"

	"gopkg.in/go-playground/validator.v9"
)

type defaultValidator struct {
	once     sync.Once
	validate *validator.Validate
}

var _ StructValidator = &defaultValidator{}

// ValidateStruct receives any kind of type, but only performed struct or pointer to struct type.
func (v *defaultValidator) ValidateStruct(obj interface{}) error {
	value := reflect.ValueOf(obj)
	valueType := value.Kind()
	if valueType == reflect.Ptr {
		valueType = value.Elem().Kind()
	}
	if valueType == reflect.Struct {
		v.lazyinit()
		if err := v.validate.Struct(obj); err != nil {
			return err
		}
	}
	return nil
}

// Engine returns the underlying validator engine which powers the default
// Validator instance. This is useful if you want to register custom validations
// or struct level validations. See validator GoDoc for more info -
// https://godoc.org/gopkg.in/go-playground/validator.v8
func (v *defaultValidator) Engine() interface{} {
	v.lazyinit()
	return v.validate
}

func (v *defaultValidator) lazyinit() {
	v.once.Do(func() {
		v.validate = validator.New()
		v.validate.SetTagName("binding")
	})
}
```

默认的验证器是 `validator.v9`, 使用了懒初始化, 以及使用 `reflect` 判断数据类型, 只验证结构体.

## Context 之响应

看完了请求参数的获取和模型绑定之后, 来看看响应是如何发送的.

先来看一下 Context 中用到的 `responseWriter` 类型和 `ResponseWriter` 类型.

```go
type Context struct {
	writermem responseWriter
	Request   *http.Request
	Writer    ResponseWriter
```

```go
// ResponseWriter ...
type ResponseWriter interface {
	http.ResponseWriter
	http.Hijacker
	http.Flusher
	http.CloseNotifier

	// Returns the HTTP response status code of the current request.
	Status() int

	// Returns the number of bytes already written into the response http body.
	// See Written()
	Size() int

	// Writes the string into the response body.
	WriteString(string) (int, error)

	// Returns true if the response body was already written.
	Written() bool

	// Forces to write the http header (status code + headers).
	WriteHeaderNow()

	// get the http.Pusher for server push
	Pusher() http.Pusher
}

type responseWriter struct {
	http.ResponseWriter
	size   int
	status int
}

var _ ResponseWriter = &responseWriter{}
```

`ResponseWriter` 接口组合了 `http` 包中用于响应的数据结构, 所有的方法上都有注释.
而 `responseWriter` 实际上就是实现了 `ResponseWriter` 接口的结构体.

在继续之前, 先来了解下 Context 中 writermem 的作用.
`Writer` 是用于写入响应的, 而从 `writermem` 名字的后缀, 可以推断出这和内存有关.
再寻找一下它的用处.

```go
// ServeHTTP conforms to the http.Handler interface.
func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	c := engine.pool.Get().(*Context)
	c.writermem.reset(w)
	c.Request = req
	c.reset()

	engine.handleHTTPRequest(c)

	engine.pool.Put(c)
}

func (c *Context) reset() {
  c.Writer = &c.writermem
  ...
}

func (w *responseWriter) reset(writer http.ResponseWriter) {
	w.ResponseWriter = writer
	w.size = noWritten
	w.status = defaultStatus
}
```

所以, 可以推断出 `writermem` 是每次请求时 `w http.ResponseWriter` 的拥有者, 而 `c.Writer` 是它的指针.

继续看 Context 是如何处理响应的.

```go
func (c *Context) requestHeader(key string) string {
	return c.Request.Header.Get(key)
}

// Status sets the HTTP response code.
func (c *Context) Status(code int) {
	c.Writer.WriteHeader(code)
}

// Header is a intelligent shortcut for c.Writer.Header().Set(key, value).
// It writes a header in the response.
// If value == "", this method removes the header `c.Writer.Header().Del(key)`
func (c *Context) Header(key, value string) {
	if value == "" {
		c.Writer.Header().Del(key)
		return
	}
	c.Writer.Header().Set(key, value)
}

// GetHeader returns value from request headers.
func (c *Context) GetHeader(key string) string {
	return c.requestHeader(key)
}
```

上面是和 Header 有关的部分, 实际上是内部的 `Writer.Header()` 的代理.

接着看和 Cookie 有关的部分:

```go
// SetCookie adds a Set-Cookie header to the ResponseWriter's headers.
// The provided cookie must have a valid Name. Invalid cookies may be
// silently dropped.
func (c *Context) SetCookie(name, value string, maxAge int, path, domain string, secure, httpOnly bool) {
	if path == "" {
		path = "/"
	}
	http.SetCookie(c.Writer, &http.Cookie{
		Name:     name,
		Value:    url.QueryEscape(value),
		MaxAge:   maxAge,
		Path:     path,
		Domain:   domain,
		Secure:   secure,
		HttpOnly: httpOnly,
	})
}

// Cookie returns the named cookie provided in the request or
// ErrNoCookie if not found. And return the named cookie is unescaped.
// If multiple cookies match the given name, only one cookie will
// be returned.
func (c *Context) Cookie(name string) (string, error) {
	cookie, err := c.Request.Cookie(name)
	if err != nil {
		return "", err
	}
	val, _ := url.QueryUnescape(cookie.Value)
	return val, nil
}
```

整合了 Cookie 的读取与设置.

看完 Header 和 Cookie 之后, 接下来就是重点了, 看一下如何渲染内容, 即返回的响应.

Gin 支持 `XML, JSON, YAML and ProtoBuf rendering`, 看一下具体的实现方式.

```go
// Render writes the response headers and calls render.Render to render data.
func (c *Context) Render(code int, r render.Render) {
	c.Status(code)

	if !bodyAllowedForStatus(code) {
		r.WriteContentType(c.Writer)
		c.Writer.WriteHeaderNow()
		return
	}

	if err := r.Render(c.Writer); err != nil {
		panic(err)
	}
}
```

主要的方法就是 `Render`, 而内部使用了 `render.Render` 接口中的 `Render` 方法.

```go
// HTML renders the HTTP template specified by its file name.
// It also updates the HTTP code and sets the Content-Type as "text/html".
// See http://golang.org/doc/articles/wiki/
func (c *Context) HTML(code int, name string, obj interface{}) {
	instance := c.engine.HTMLRender.Instance(name, obj)
	c.Render(code, instance)
}

// IndentedJSON serializes the given struct as pretty JSON (indented + endlines) into the response body.
// It also sets the Content-Type as "application/json".
// WARNING: we recommend to use this only for development purposes since printing pretty JSON is
// more CPU and bandwidth consuming. Use Context.JSON() instead.
func (c *Context) IndentedJSON(code int, obj interface{}) {
	c.Render(code, render.IndentedJSON{Data: obj})
}

// SecureJSON serializes the given struct as Secure JSON into the response body.
// Default prepends "while(1)," to response body if the given struct is array values.
// It also sets the Content-Type as "application/json".
func (c *Context) SecureJSON(code int, obj interface{}) {
	c.Render(code, render.SecureJSON{Prefix: c.engine.secureJsonPrefix, Data: obj})
}

// JSONP serializes the given struct as JSON into the response body.
// It add padding to response body to request data from a server residing in a different domain than the client.
// It also sets the Content-Type as "application/javascript".
func (c *Context) JSONP(code int, obj interface{}) {
	callback := c.DefaultQuery("callback", "")
	if callback == "" {
		c.Render(code, render.JSON{Data: obj})
		return
	}
	c.Render(code, render.JsonpJSON{Callback: callback, Data: obj})
}

// JSON serializes the given struct as JSON into the response body.
// It also sets the Content-Type as "application/json".
func (c *Context) JSON(code int, obj interface{}) {
	c.Render(code, render.JSON{Data: obj})
}

// AsciiJSON serializes the given struct as JSON into the response body with unicode to ASCII string.
// It also sets the Content-Type as "application/json".
func (c *Context) AsciiJSON(code int, obj interface{}) {
	c.Render(code, render.AsciiJSON{Data: obj})
}

// PureJSON serializes the given struct as JSON into the response body.
// PureJSON, unlike JSON, does not replace special html characters with their unicode entities.
func (c *Context) PureJSON(code int, obj interface{}) {
	c.Render(code, render.PureJSON{Data: obj})
}

// XML serializes the given struct as XML into the response body.
// It also sets the Content-Type as "application/xml".
func (c *Context) XML(code int, obj interface{}) {
	c.Render(code, render.XML{Data: obj})
}

// YAML serializes the given struct as YAML into the response body.
func (c *Context) YAML(code int, obj interface{}) {
	c.Render(code, render.YAML{Data: obj})
}

// ProtoBuf serializes the given struct as ProtoBuf into the response body.
func (c *Context) ProtoBuf(code int, obj interface{}) {
	c.Render(code, render.ProtoBuf{Data: obj})
}

// String writes the given string into the response body.
func (c *Context) String(code int, format string, values ...interface{}) {
	c.Render(code, render.String{Format: format, Data: values})
}

// Redirect returns a HTTP redirect to the specific location.
func (c *Context) Redirect(code int, location string) {
	c.Render(-1, render.Redirect{
		Code:     code,
		Location: location,
		Request:  c.Request,
	})
}

// Data writes some data into the body stream and updates the HTTP code.
func (c *Context) Data(code int, contentType string, data []byte) {
	c.Render(code, render.Data{
		ContentType: contentType,
		Data:        data,
	})
}

// DataFromReader writes the specified reader into the body stream and updates the HTTP code.
func (c *Context) DataFromReader(code int, contentLength int64, contentType string, reader io.Reader, extraHeaders map[string]string) {
	c.Render(code, render.Reader{
		Headers:       extraHeaders,
		ContentType:   contentType,
		ContentLength: contentLength,
		Reader:        reader,
	})
}
```

看一下这些迥异的 `render.Render` 接口的实现者.

```go
package render

import "net/http"

// Render interface is to be implemented by JSON, XML, HTML, YAML and so on.
type Render interface {
	// Render writes data with custom ContentType.
	Render(http.ResponseWriter) error
	// WriteContentType writes custom ContentType.
	WriteContentType(w http.ResponseWriter)
}

var (
	_ Render     = JSON{}
	_ Render     = IndentedJSON{}
	_ Render     = SecureJSON{}
	_ Render     = JsonpJSON{}
	_ Render     = XML{}
	_ Render     = String{}
	_ Render     = Redirect{}
	_ Render     = Data{}
	_ Render     = HTML{}
	_ HTMLRender = HTMLDebug{}
	_ HTMLRender = HTMLProduction{}
	_ Render     = YAML{}
	_ Render     = MsgPack{}
	_ Render     = Reader{}
	_ Render     = AsciiJSON{}
	_ Render     = ProtoBuf{}
)

func writeContentType(w http.ResponseWriter, value []string) {
	header := w.Header()
	if val := header["Content-Type"]; len(val) == 0 {
		header["Content-Type"] = value
	}
}
```

上面是 `Render` 接口的定义, 主要需要实现 `Render` 方法.
`WriteContentType` 方法实际上已经被 `writeContentType` 函数实现得差不多了,
只是每种渲染方式对应的 `Content-Type` 值不同.

以 JSON 为例, 看一下具体是如何实现的.

```go
// JSON contains the given interface object.
type JSON struct {
	Data interface{}
}

var jsonContentType = []string{"application/json; charset=utf-8"}

// Render (JSON) writes data with custom ContentType.
func (r JSON) Render(w http.ResponseWriter) (err error) {
	if err = WriteJSON(w, r.Data); err != nil {
		panic(err)
	}
	return
}

// WriteContentType (JSON) writes JSON ContentType.
func (r JSON) WriteContentType(w http.ResponseWriter) {
	writeContentType(w, jsonContentType)
}

// WriteJSON marshals the given interface object and writes it with custom ContentType.
func WriteJSON(w http.ResponseWriter, obj interface{}) error {
	writeContentType(w, jsonContentType)
	encoder := json.NewEncoder(w)
	err := encoder.Encode(&obj)
	return err
}
```

看上去非常简洁, 实现也不复杂.

`Render` 和 `Binding` 非常相似, 都是通过定义接口, 然后用不同的结构体实现具体的功能.

## Context 之高级响应

```go
// File writes the specified file into the body stream in a efficient way.
func (c *Context) File(filepath string) {
	http.ServeFile(c.Writer, c.Request, filepath)
}

// FileAttachment writes the specified file into the body stream in an efficient way
// On the client side, the file will typically be downloaded with the given filename
func (c *Context) FileAttachment(filepath, filename string) {
	c.Writer.Header().Set("content-disposition", fmt.Sprintf("attachment; filename=\"%s\"", filename))
	http.ServeFile(c.Writer, c.Request, filepath)
}
```

托管静态文件, 使用的是 `http.ServeFile`, 也实现了附件下载的功能, 还是挺方便的,
虽然只是 `content-disposition` 这个 Header 的功能.

```go
// SSEvent writes a Server-Sent Event into the body stream.
func (c *Context) SSEvent(name string, message interface{}) {
	c.Render(-1, sse.Event{
		Event: name,
		Data:  message,
	})
}
```

`SSEvent` 实现了服务端推送事件的功能, 具体看一下它的实现.

```go
package sse

import (
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"reflect"
	"strconv"
	"strings"
)

// Server-Sent Events
// W3C Working Draft 29 October 2009
// http://www.w3.org/TR/2009/WD-eventsource-20091029/

const ContentType = "text/event-stream"

var contentType = []string{ContentType}
var noCache = []string{"no-cache"}

var fieldReplacer = strings.NewReplacer(
	"\n", "\\n",
	"\r", "\\r")

var dataReplacer = strings.NewReplacer(
	"\n", "\ndata:",
	"\r", "\\r")

type Event struct {
	Event string
	Id    string
	Retry uint
	Data  interface{}
}

func Encode(writer io.Writer, event Event) error {
	w := checkWriter(writer)
	writeId(w, event.Id)
	writeEvent(w, event.Event)
	writeRetry(w, event.Retry)
	return writeData(w, event.Data)
}

func writeId(w stringWriter, id string) {
	if len(id) > 0 {
		w.WriteString("id:")
		fieldReplacer.WriteString(w, id)
		w.WriteString("\n")
	}
}

func writeEvent(w stringWriter, event string) {
	if len(event) > 0 {
		w.WriteString("event:")
		fieldReplacer.WriteString(w, event)
		w.WriteString("\n")
	}
}

func writeRetry(w stringWriter, retry uint) {
	if retry > 0 {
		w.WriteString("retry:")
		w.WriteString(strconv.FormatUint(uint64(retry), 10))
		w.WriteString("\n")
	}
}

func writeData(w stringWriter, data interface{}) error {
	w.WriteString("data:")
	switch kindOfData(data) {
	case reflect.Struct, reflect.Slice, reflect.Map:
		err := json.NewEncoder(w).Encode(data)
		if err != nil {
			return err
		}
		w.WriteString("\n")
	default:
		dataReplacer.WriteString(w, fmt.Sprint(data))
		w.WriteString("\n\n")
	}
	return nil
}

func (r Event) Render(w http.ResponseWriter) error {
	r.WriteContentType(w)
	return Encode(w, r)
}

func (r Event) WriteContentType(w http.ResponseWriter) {
	header := w.Header()
	header["Content-Type"] = contentType

	if _, exist := header["Cache-Control"]; !exist {
		header["Cache-Control"] = noCache
	}
}

func kindOfData(data interface{}) reflect.Kind {
	value := reflect.ValueOf(data)
	valueType := value.Kind()
	if valueType == reflect.Ptr {
		valueType = value.Elem().Kind()
	}
	return valueType
}
```

`SSEvent` 是作为扩展实现的, 代码并不在 Gin 的源码中. 先看一下 `Event` 结构体.

```go
type Event struct {
	Event string
	Id    string
	Retry uint
	Data  interface{}
}

func (r Event) Render(w http.ResponseWriter) error {
	r.WriteContentType(w)
	return Encode(w, r)
}
```

`Event` 实现了 `Render` 接口, 看一下内部的 `Encode` 函数.

```go
func Encode(writer io.Writer, event Event) error {
	w := checkWriter(writer)
	writeId(w, event.Id)
	writeEvent(w, event.Event)
	writeRetry(w, event.Retry)
	return writeData(w, event.Data)
}
```

过程并不复杂, 分为四步写入, 分别是事件 ID, 事件名 Event, 重连时间 Retry, 消息体 Data.
如果对服务端推送事件不太了解, 可以参考
[MDN-使用服务器发送事件.](https://developer.mozilla.org/zh-CN/docs/Server-sent_events/Using_server-sent_events).

> 事件流仅仅是一个简单的文本数据流,文本应该使用 UTF- 8 格式的编码.每条消息后面都由一个空行作为分隔符.以冒号开头的行为注释行,会被忽略.
> 注:注释行可以用来防止连接超时,服务器可以定期发送一条消息注释行,以保持连接不断.
> 每条消息是由多个字段组成的,每个字段由字段名,一个冒号,以及字段值组成.

实际上并没有对消息体的格式做任何要求, 这属于前后端协定的范围.

```go
func writeData(w stringWriter, data interface{}) error {
	w.WriteString("data:")
	switch kindOfData(data) {
	case reflect.Struct, reflect.Slice, reflect.Map:
		err := json.NewEncoder(w).Encode(data)
		if err != nil {
			return err
		}
		w.WriteString("\n")
	default:
		dataReplacer.WriteString(w, fmt.Sprint(data))
		w.WriteString("\n\n")
	}
	return nil
}
```

该实现中, 主要使用了 JSON 格式, 但对其他类型的数据直接写入纯文本.

接着看一下流式响应是如何实现的.

```go
// Stream sends a streaming response and returns a boolean
// indicates "Is client disconnected in middle of stream"
func (c *Context) Stream(step func(w io.Writer) bool) bool {
	w := c.Writer
	clientGone := w.CloseNotify()
	for {
		select {
		case <-clientGone:
			return true
		default:
			keepOpen := step(w)
			w.Flush()
			if !keepOpen {
				return false
			}
		}
	}
}
```

这是一个非常常见的模式, 使用 for 和 select 以及 channel 实现无限循环.

## Context 之内容协商

内容协商通过 `Accept` Header 实现, 用于为不同类型的客户端提供不同类型的资源,
比如协商网页语言或响应格式等.

具体可以参考 [MDN-内容协商](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Content_negotiation).

```go
// Negotiate contains all negotiations data.
type Negotiate struct {
	Offered  []string
	HTMLName string
	HTMLData interface{}
	JSONData interface{}
	XMLData  interface{}
	Data     interface{}
}

// Negotiate calls different Render according acceptable Accept format.
func (c *Context) Negotiate(code int, config Negotiate) {
	switch c.NegotiateFormat(config.Offered...) {
	case binding.MIMEJSON:
		data := chooseData(config.JSONData, config.Data)
		c.JSON(code, data)

	case binding.MIMEHTML:
		data := chooseData(config.HTMLData, config.Data)
		c.HTML(code, config.HTMLName, data)

	case binding.MIMEXML:
		data := chooseData(config.XMLData, config.Data)
		c.XML(code, data)

	default:
		c.AbortWithError(http.StatusNotAcceptable, errors.New("the accepted formats are not offered by the server")) // nolint: errcheck
	}
}

// NegotiateFormat returns an acceptable Accept format.
func (c *Context) NegotiateFormat(offered ...string) string {
	assert1(len(offered) > 0, "you must provide at least one offer")

	if c.Accepted == nil {
		c.Accepted = parseAccept(c.requestHeader("Accept"))
	}
	if len(c.Accepted) == 0 {
		return offered[0]
	}
	for _, accepted := range c.Accepted {
		for _, offert := range offered {
			// According to RFC 2616 and RFC 2396, non-ASCII characters are not allowed in headers,
			// therefore we can just iterate over the string without casting it into []rune
			i := 0
			for ; i < len(accepted); i++ {
				if accepted[i] == '*' || offert[i] == '*' {
					return offert
				}
				if accepted[i] != offert[i] {
					break
				}
			}
			if i == len(accepted) {
				return offert
			}
		}
	}
	return ""
}

// SetAccepted sets Accept header data.
func (c *Context) SetAccepted(formats ...string) {
	c.Accepted = formats
}
```

## 总结

Context 的内容就到这里了, 虽然源文件有点长, 但配合注释还是挺清晰的.
