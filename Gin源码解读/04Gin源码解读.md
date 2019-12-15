<!-- TOC -->

- [简介](#简介)
- [内置中间件的实现](#内置中间件的实现)
  - [recovery](#recovery)
  - [auth](#auth)
  - [logger](#logger)
- [errors](#errors)
- [总结](#总结)

<!-- /TOC -->

## 简介

Gin 源码解读, 基于 [v1.5.0 ](https://github.com/gin-gonic/gin/tree/v1.5.0) 版本.

## 内置中间件的实现

前面已经研究过中间件的原理了, 这次来看一下内置的中间件是如何实现的.

### recovery

```go
// Recovery returns a middleware that recovers from any panics and writes a 500 if there was one.
func Recovery() HandlerFunc {
	return RecoveryWithWriter(DefaultErrorWriter)
}
```

recovery 中间件用于从 panic 中恢复, 并返回 500 响应.

在看代码之前, 首先介绍下内置的 `recover` 函数.

```go
func recover() interface{}
```

> The recover built-in function allows a program to manage behavior of a panicking goroutine. Executing a call to recover inside a deferred function (but not any function called by it) stops the panicking sequence by restoring normal execution and retrieves the error value passed to the call of panic. If recover is called outside the deferred function it will not stop a panicking sequence. In this case, or when the goroutine is not panicking, or if the argument supplied to panic was nil, recover returns nil. Thus the return value from recover reports whether the goroutine is panicking.

`recover` 用于控制处于 panic 状态中的 goroutine 的行为, 只能用于 `defer` 语句的函数中.

简单的用法如下:

```go
package main

import (
	"fmt"
)

func main() {
	defer func() {
		err := recover()
		if err != nil {
			fmt.Println("catch panic:", err)
		}
	}()

	panic("hello error")
}
```

具体看一下 `RecoveryWithWriter` 的实现.

```go
// RecoveryWithWriter returns a middleware for a given writer that recovers from any panics and writes a 500 if there was one.
func RecoveryWithWriter(out io.Writer) HandlerFunc {
	var logger *log.Logger
	if out != nil {
		logger = log.New(out, "\n\n\x1b[31m", log.LstdFlags)
	}
	return func(c *Context) {
		defer func() {
			if err := recover(); err != nil {
				// Check for a broken connection, as it is not really a
				// condition that warrants a panic stack trace.
				var brokenPipe bool
				if ne, ok := err.(*net.OpError); ok {
					if se, ok := ne.Err.(*os.SyscallError); ok {
						if strings.Contains(strings.ToLower(se.Error()), "broken pipe") || strings.Contains(strings.ToLower(se.Error()), "connection reset by peer") {
							brokenPipe = true
						}
					}
				}
				if logger != nil {
					stack := stack(3)
					httpRequest, _ := httputil.DumpRequest(c.Request, false)
					headers := strings.Split(string(httpRequest), "\r\n")
					for idx, header := range headers {
						current := strings.Split(header, ":")
						if current[0] == "Authorization" {
							headers[idx] = current[0] + ": *"
						}
					}
					if brokenPipe {
						logger.Printf("%s\n%s%s", err, string(httpRequest), reset)
					} else if IsDebugging() {
						logger.Printf("[Recovery] %s panic recovered:\n%s\n%s\n%s%s",
							timeFormat(time.Now()), strings.Join(headers, "\r\n"), err, stack, reset)
					} else {
						logger.Printf("[Recovery] %s panic recovered:\n%s\n%s%s",
							timeFormat(time.Now()), err, stack, reset)
					}
				}

				// If the connection is dead, we can't write a status to it.
				if brokenPipe {
					c.Error(err.(error)) // nolint: errcheck
					c.Abort()
				} else {
					c.AbortWithStatus(http.StatusInternalServerError)
				}
			}
		}()
		c.Next()
	}
}
```

简单来看, 最后返回的 `func(c *Context)` 中间件函数内部分为两个主要部分, 一个是 `defer` 处理, 另一个是`c.Next()`.

实际上中间件函数什么都不做, 只是调用 `c.Next()` 转移控制权, 顺着调用链去运行其他中间件和 handler 函数.
当调用链全部执行完, `c.Next()` 运行完毕, `recover` 结束之后, 就轮到 `defer` 语句出场了.

首先判断了连接是否已经失效:

```go
// Check for a broken connection, as it is not really a
// condition that warrants a panic stack trace.
var brokenPipe bool
if ne, ok := err.(*net.OpError); ok {
  if se, ok := ne.Err.(*os.SyscallError); ok {
    if strings.Contains(strings.ToLower(se.Error()), "broken pipe") || strings.Contains(strings.ToLower(se.Error()), "connection reset by peer") {
      brokenPipe = true
    }
  }
}
```

然后记录日志:

```go
if logger != nil {
  stack := stack(3)
  httpRequest, _ := httputil.DumpRequest(c.Request, false)
  headers := strings.Split(string(httpRequest), "\r\n")
  for idx, header := range headers {
    current := strings.Split(header, ":")
    if current[0] == "Authorization" {
      headers[idx] = current[0] + ": *"
    }
  }
  if brokenPipe {
    logger.Printf("%s\n%s%s", err, string(httpRequest), reset)
  } else if IsDebugging() {
    logger.Printf("[Recovery] %s panic recovered:\n%s\n%s\n%s%s",
      timeFormat(time.Now()), strings.Join(headers, "\r\n"), err, stack, reset)
  } else {
    logger.Printf("[Recovery] %s panic recovered:\n%s\n%s%s",
      timeFormat(time.Now()), err, stack, reset)
  }
}
```

最后, 根据连接状态, 进行不同的处理:

```go
// If the connection is dead, we can't write a status to it.
if brokenPipe {
  c.Error(err.(error)) // nolint: errcheck
  c.Abort()
} else {
  c.AbortWithStatus(http.StatusInternalServerError)
}
```

总的来看, 没有什么特殊的, 如果你已经熟悉了 Golang 内置的 `recover` 机制.

### auth

auth 中间件用于 `Basic HTTP Authorization`.

```go
// BasicAuth returns a Basic HTTP Authorization middleware. It takes as argument a map[string]string where
// the key is the user name and the value is the password.
func BasicAuth(accounts Accounts) HandlerFunc {
	return BasicAuthForRealm(accounts, "")
}
```

内部实现为:

```go
// BasicAuthForRealm returns a Basic HTTP Authorization middleware. It takes as arguments a map[string]string where
// the key is the user name and the value is the password, as well as the name of the Realm.
// If the realm is empty, "Authorization Required" will be used by default.
// (see http://tools.ietf.org/html/rfc2617#section-1.2)
func BasicAuthForRealm(accounts Accounts, realm string) HandlerFunc {
	if realm == "" {
		realm = "Authorization Required"
	}
	realm = "Basic realm=" + strconv.Quote(realm)
	pairs := processAccounts(accounts)
	return func(c *Context) {
		// Search user in the slice of allowed credentials
		user, found := pairs.searchCredential(c.requestHeader("Authorization"))
		if !found {
			// Credentials doesn't match, we return 401 and abort handlers chain.
			c.Header("WWW-Authenticate", realm)
			c.AbortWithStatus(http.StatusUnauthorized)
			return
		}

		// The user credentials was found, set user's id to key AuthUserKey in this context, the user's id can be read later using
		// c.MustGet(gin.AuthUserKey).
		c.Set(AuthUserKey, user)
	}
}
```

使用 `pairs` 变量保存用户名密码对. 如果用户没有找到, 会返回 401 响应, 并设置对应的 `WWW-Authenticate` Header.

```go
// AuthUserKey is the cookie name for user credential in basic auth.
const AuthUserKey = "user"

// Accounts defines a key/value for user/pass list of authorized logins.
type Accounts map[string]string

type authPair struct {
	value string
	user  string
}

type authPairs []authPair

func (a authPairs) searchCredential(authValue string) (string, bool) {
	if authValue == "" {
		return "", false
	}
	for _, pair := range a {
		if pair.value == authValue {
			return pair.user, true
		}
	}
	return "", false
}

func processAccounts(accounts Accounts) authPairs {
	assert1(len(accounts) > 0, "Empty list of authorized credentials")
	pairs := make(authPairs, 0, len(accounts))
	for user, password := range accounts {
		assert1(user != "", "User can not be empty")
		value := authorizationHeader(user, password)
		pairs = append(pairs, authPair{
			value: value,
			user:  user,
		})
	}
	return pairs
}

func authorizationHeader(user, password string) string {
	base := user + ":" + password
	return "Basic " + base64.StdEncoding.EncodeToString([]byte(base))
}
```

简单认证中间件也没有什么特殊的, 看源码可以对认证过程有更清晰的了解.
可以参考 [MDN-HTTP 身份验证](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Authentication).

### logger

logger 实现了内置的日志记录器.

日志是可配置的, 先来看一下数据结构部分.

```go
// LoggerConfig defines the config for Logger middleware.
type LoggerConfig struct {
	// Optional. Default value is gin.defaultLogFormatter
	Formatter LogFormatter

	// Output is a writer where logs are written.
	// Optional. Default value is gin.DefaultWriter.
	Output io.Writer

	// SkipPaths is a url path array which logs are not written.
	// Optional.
	SkipPaths []string
}

// LogFormatter gives the signature of the formatter function passed to LoggerWithFormatter
type LogFormatter func(params LogFormatterParams) string

// LogFormatterParams is the structure any formatter will be handed when time to log comes
type LogFormatterParams struct {
	Request *http.Request

	// TimeStamp shows the time after the server returns a response.
	TimeStamp time.Time
	// StatusCode is HTTP response code.
	StatusCode int
	// Latency is how much time the server cost to process a certain request.
	Latency time.Duration
	// ClientIP equals Context's ClientIP method.
	ClientIP string
	// Method is the HTTP method given to the request.
	Method string
	// Path is a path the client requests.
	Path string
	// ErrorMessage is set if error has occurred in processing the request.
	ErrorMessage string
	// isTerm shows whether does gin's output descriptor refers to a terminal.
	isTerm bool
	// BodySize is the size of the Response Body
	BodySize int
	// Keys are the keys set on the request's context.
	Keys map[string]interface{}
}
```

日志格式里有个 `isTerm` 是为 shell 优化的标识符, 用于显示颜色.

```go
const (
	green   = "\033[97;42m"
	white   = "\033[90;47m"
	yellow  = "\033[90;43m"
	red     = "\033[97;41m"
	blue    = "\033[97;44m"
	magenta = "\033[97;45m"
	cyan    = "\033[97;46m"
	reset   = "\033[0m"
)

var consoleColorMode = autoColor

// StatusCodeColor is the ANSI color for appropriately logging http status code to a terminal.
func (p *LogFormatterParams) StatusCodeColor() string {
	code := p.StatusCode

	switch {
	case code >= http.StatusOK && code < http.StatusMultipleChoices:
		return green
	case code >= http.StatusMultipleChoices && code < http.StatusBadRequest:
		return white
	case code >= http.StatusBadRequest && code < http.StatusInternalServerError:
		return yellow
	default:
		return red
	}
}

// MethodColor is the ANSI color for appropriately logging http method to a terminal.
func (p *LogFormatterParams) MethodColor() string {
	method := p.Method

	switch method {
	case "GET":
		return blue
	case "POST":
		return cyan
	case "PUT":
		return yellow
	case "DELETE":
		return red
	case "PATCH":
		return green
	case "HEAD":
		return magenta
	case "OPTIONS":
		return white
	default:
		return reset
	}
}

// ResetColor resets all escape attributes.
func (p *LogFormatterParams) ResetColor() string {
	return reset
}

// IsOutputColor indicates whether can colors be outputted to the log.
func (p *LogFormatterParams) IsOutputColor() bool {
	return consoleColorMode == forceColor || (consoleColorMode == autoColor && p.isTerm)
}
```

看一下中间件的具体实现:

```go
// Logger instances a Logger middleware that will write the logs to gin.DefaultWriter.
// By default gin.DefaultWriter = os.Stdout.
func Logger() HandlerFunc {
	return LoggerWithConfig(LoggerConfig{})
}

// LoggerWithFormatter instance a Logger middleware with the specified log format function.
func LoggerWithFormatter(f LogFormatter) HandlerFunc {
	return LoggerWithConfig(LoggerConfig{
		Formatter: f,
	})
}

// LoggerWithWriter instance a Logger middleware with the specified writer buffer.
// Example: os.Stdout, a file opened in write mode, a socket...
func LoggerWithWriter(out io.Writer, notlogged ...string) HandlerFunc {
	return LoggerWithConfig(LoggerConfig{
		Output:    out,
		SkipPaths: notlogged,
	})
}

// LoggerWithConfig instance a Logger middleware with config.
func LoggerWithConfig(conf LoggerConfig) HandlerFunc {
	formatter := conf.Formatter
	if formatter == nil {
		formatter = defaultLogFormatter
	}

	out := conf.Output
	if out == nil {
		out = DefaultWriter
	}

	notlogged := conf.SkipPaths

	isTerm := true

	if w, ok := out.(*os.File); !ok || os.Getenv("TERM") == "dumb" ||
		(!isatty.IsTerminal(w.Fd()) && !isatty.IsCygwinTerminal(w.Fd())) {
		isTerm = false
	}

	var skip map[string]struct{}

	if length := len(notlogged); length > 0 {
		skip = make(map[string]struct{}, length)

		for _, path := range notlogged {
			skip[path] = struct{}{}
		}
	}

	return func(c *Context) {
		// Start timer
		start := time.Now()
		path := c.Request.URL.Path
		raw := c.Request.URL.RawQuery

		// Process request
		c.Next()

		// Log only when path is not being skipped
		if _, ok := skip[path]; !ok {
			param := LogFormatterParams{
				Request: c.Request,
				isTerm:  isTerm,
				Keys:    c.Keys,
			}

			// Stop timer
			param.TimeStamp = time.Now()
			param.Latency = param.TimeStamp.Sub(start)

			param.ClientIP = c.ClientIP()
			param.Method = c.Request.Method
			param.StatusCode = c.Writer.Status()
			param.ErrorMessage = c.Errors.ByType(ErrorTypePrivate).String()

			param.BodySize = c.Writer.Size()

			if raw != "" {
				path = path + "?" + raw
			}

			param.Path = path

			fmt.Fprint(out, formatter(param))
		}
	}
}
```

上面是三个中间件, 内部都使用了 `LoggerWithConfig` 函数.

中间部分有个转换, `notlogged := conf.SkipPaths` 的类型是 `[]string`, 但在初始化的时候改成了 map.

```go
var skip map[string]struct{}

if length := len(notlogged); length > 0 {
  skip = make(map[string]struct{}, length)

  for _, path := range notlogged {
    skip[path] = struct{}{}
  }
}
```

这是因为当判断一个元素是否存在时, hash 的实现 O(1) 比数组 O(n) 要高效, `if _, ok := skip[path]; !ok {`.

最后, 里面最重要的语句是 `fmt.Fprint(out, formatter(param))`, 将 out 输出格式化的日志.

默认的格式化函数是 `defaultLogFormatter`:

```go
formatter := conf.Formatter
if formatter == nil {
  formatter = defaultLogFormatter
}
```

看一下 `defaultLogFormatter` 的实现:

```go
// defaultLogFormatter is the default log format function Logger middleware uses.
var defaultLogFormatter = func(param LogFormatterParams) string {
	var statusColor, methodColor, resetColor string
	if param.IsOutputColor() {
		statusColor = param.StatusCodeColor()
		methodColor = param.MethodColor()
		resetColor = param.ResetColor()
	}

	if param.Latency > time.Minute {
		// Truncate in a golang < 1.8 safe way
		param.Latency = param.Latency - param.Latency%time.Second
	}
	return fmt.Sprintf("[GIN] %v |%s %3d %s| %13v | %15s |%s %-7s %s %s\n%s",
		param.TimeStamp.Format("2006/01/02 - 15:04:05"),
		statusColor, param.StatusCode, resetColor,
		param.Latency,
		param.ClientIP,
		methodColor, param.Method, resetColor,
		param.Path,
		param.ErrorMessage,
	)
}
```

所以, 要实现自定义格式化内容, 就是要实现 `func(param LogFormatterParams) string` 函数.

官方文档中自定义格式化内容的例子如下:

```go
func main() {
	router := gin.New()

	// LoggerWithFormatter middleware will write the logs to gin.DefaultWriter
	// By default gin.DefaultWriter = os.Stdout
	router.Use(gin.LoggerWithFormatter(func(param gin.LogFormatterParams) string {

		// your custom format
		return fmt.Sprintf("%s - [%s] \"%s %s %s %d %s \"%s\" %s\"\n",
				param.ClientIP,
				param.TimeStamp.Format(time.RFC1123),
				param.Method,
				param.Path,
				param.Request.Proto,
				param.StatusCode,
				param.Latency,
				param.Request.UserAgent(),
				param.ErrorMessage,
		)
	}))
	router.Use(gin.Recovery())

	router.GET("/ping", func(c *gin.Context) {
		c.String(200, "pong")
	})

	router.Run(":8080")
}
```

另一点则是计算时延, 在函数的开始计时 `start := time.Now()`, 当 `c.Next()` 处理完请求后,
停止计时 `param.Latency = param.TimeStamp.Sub(start)`.

所以, 如果你需要一个完整的时延, 就需要将 logger 放在中间件的最前面.
当你想要忽略中间件的耗时, 只统计 handler 处理时间, 就需要放在中间件的最后.
但遇到后者的情形, 最好还是自己实现一个计时的中间件.

## errors

看一下错误类型是如何定义的.

```go
// ErrorType is an unsigned 64-bit error code as defined in the gin spec.
type ErrorType uint64

const (
	// ErrorTypeBind is used when Context.Bind() fails.
	ErrorTypeBind ErrorType = 1 << 63
	// ErrorTypeRender is used when Context.Render() fails.
	ErrorTypeRender ErrorType = 1 << 62
	// ErrorTypePrivate indicates a private error.
	ErrorTypePrivate ErrorType = 1 << 0
	// ErrorTypePublic indicates a public error.
	ErrorTypePublic ErrorType = 1 << 1
	// ErrorTypeAny indicates any other error.
	ErrorTypeAny ErrorType = 1<<64 - 1
	// ErrorTypeNu indicates any other error.
	ErrorTypeNu = 2
)

// Error represents a error's specification.
type Error struct {
	Err  error
	Type ErrorType
	Meta interface{}
}

type errorMsgs []*Error
```

`Error` 结构体中有三个字段, 一个是原始的错误 Err, 一个是错误类型 Type, 另一个是 Meta 元信息.

```go
// SetType sets the error's type.
func (msg *Error) SetType(flags ErrorType) *Error {
	msg.Type = flags
	return msg
}

// SetMeta sets the error's meta data.
func (msg *Error) SetMeta(data interface{}) *Error {
	msg.Meta = data
	return msg
}
```

```go
// JSON creates a properly formatted JSON
func (msg *Error) JSON() interface{} {
	json := H{}
	if msg.Meta != nil {
		value := reflect.ValueOf(msg.Meta)
		switch value.Kind() {
		case reflect.Struct:
			return msg.Meta
		case reflect.Map:
			for _, key := range value.MapKeys() {
				json[key.String()] = value.MapIndex(key).Interface()
			}
		default:
			json["meta"] = msg.Meta
		}
	}
	if _, ok := json["error"]; !ok {
		json["error"] = msg.Error()
	}
	return json
}

// MarshalJSON implements the json.Marshaller interface.
func (msg *Error) MarshalJSON() ([]byte, error) {
	return json.Marshal(msg.JSON())
}

// Error implements the error interface.
func (msg Error) Error() string {
	return msg.Err.Error()
}
```

判断错误类型的方式有点特别:

```go
// IsType judges one error.
func (msg *Error) IsType(flags ErrorType) bool {
	return (msg.Type & flags) > 0
}
```

这用到了位运算 `&`, 难道比普通的 `==` 更快吗?

后面都是 `errorMsgs` 的方法:

```go
// ByType returns a readonly copy filtered the byte.
// ie ByType(gin.ErrorTypePublic) returns a slice of errors with type=ErrorTypePublic.
func (a errorMsgs) ByType(typ ErrorType) errorMsgs {
	if len(a) == 0 {
		return nil
	}
	if typ == ErrorTypeAny {
		return a
	}
	var result errorMsgs
	for _, msg := range a {
		if msg.IsType(typ) {
			result = append(result, msg)
		}
	}
	return result
}

// Last returns the last error in the slice. It returns nil if the array is empty.
// Shortcut for errors[len(errors)-1].
func (a errorMsgs) Last() *Error {
	if length := len(a); length > 0 {
		return a[length-1]
	}
	return nil
}

// Errors returns an array will all the error messages.
// Example:
// 		c.Error(errors.New("first"))
// 		c.Error(errors.New("second"))
// 		c.Error(errors.New("third"))
// 		c.Errors.Errors() // == []string{"first", "second", "third"}
func (a errorMsgs) Errors() []string {
	if len(a) == 0 {
		return nil
	}
	errorStrings := make([]string, len(a))
	for i, err := range a {
		errorStrings[i] = err.Error()
	}
	return errorStrings
}

func (a errorMsgs) JSON() interface{} {
	switch len(a) {
	case 0:
		return nil
	case 1:
		return a.Last().JSON()
	default:
		json := make([]interface{}, len(a))
		for i, err := range a {
			json[i] = err.JSON()
		}
		return json
	}
}

// MarshalJSON implements the json.Marshaller interface.
func (a errorMsgs) MarshalJSON() ([]byte, error) {
	return json.Marshal(a.JSON())
}

func (a errorMsgs) String() string {
	if len(a) == 0 {
		return ""
	}
	var buffer strings.Builder
	for i, msg := range a {
		fmt.Fprintf(&buffer, "Error #%02d: %s\n", i+1, msg.Err)
		if msg.Meta != nil {
			fmt.Fprintf(&buffer, "     Meta: %v\n", msg.Meta)
		}
	}
	return buffer.String()
}
```

## 总结

差不多就是这样, 结合前几篇, 已经将 Gin 的源码看的差不多了.
binding 和 render 部分只挑选了 JSON 实现.
