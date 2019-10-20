<!-- TOC -->

- [介绍 jwt](#介绍-jwt)
- [实践](#实践)
  - [定义功能](#定义功能)
  - [签发接口](#签发接口)
  - [验证中间件](#验证中间件)
  - [使用](#使用)
- [总结](#总结)
- [当前部分的代码](#当前部分的代码)

<!-- /TOC -->

在典型的业务场景中, 认证与鉴权是十分基础的.

对于 API 接口, 通常是在第一次验证之后生成一个带有时效的 token.
接下来的一系列请求都携带这个 token, 服务器会对这个 token 进行验证.

## 介绍 jwt

JSON Web Tokens(jwt) 是一种用于在两个主体间传递认证消息的方式.
注意, 消息是通过数字签名的, 因此可以被验证和信任, 但却不是加密的.

![jwt 格式](./img/page/09-01-jwt.png)

一个 jwt 由三部分组成:

- Header
- Payload
- Signature

Header 部分通常只有两个字段, 分别定义了签名算法和 token 类型.

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

Payload 部分是实际负载, 用于声明. 通常存储一些用户 ID 之类的索引数据,
也可以放一些其他有用的信息. 注意, 不要存储机密数据.

jwt 规范也在 Payload 中预定义了推荐字段, 但非强制的, 但很多库都会遵照着实现.
比如 iss 字段定义发布者, exp 定义 token 的过期时间. 更多字段可以在
[rfc7519 规范](https://tools.ietf.org/html/rfc7519#section-4.1)
中查看.

Signature 就是签名了, 大致样式如下:

```text
HMACSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  secret)
```

将 header 和 payload 分别使用 base64UrlEncode 编码, 中间加上点号`.`,
然后使用 header 中指定的签名算法.

签名用于发布者验证消息没有被篡改, 如果使用非对称加密, 还可以验证发布者身份.

jwt 通常用在请求头的 `Authorization` 字段中, 形如:

```text
Authorization: Bearer <token>
```

更多内容可以参考 [Introduction to JSON Web Tokens](https://jwt.io/introduction/).

## 实践

对于如何使用 jwt, 应该是非常清晰了.
首先, 我们要定义一个接口签发 jwt.
获取到 token 之后, 就可以在请求其他资源时带上这个 token.

验证环节可以用到上一节中讲到的中间件技术.

[jwt.io](https://jwt.io/#debugger) 页面上列出了很多 Go 库,
这里选择功能最全的 `github.com/gbrlsnchs/jwt/v3`.

```bash
go get -u github.com/gbrlsnchs/jwt/v3
```

### 定义功能

对于 jwt 有两个必须实现的功能, 签发 token 和验证 token.

首先, 首先定义 Payload 内容, 这里保持用户的 ID 和昵称.

```go
// 记录登录信息的 JWT
type LoginToken struct {
	jwt.Payload
	ID       uint   `json:"id"`
	Username string `json:"username"`
}
```

然后选择签名方法.

```go
// 签名算法, 随机, 不保存密钥, 每次都是随机的
var privateKey, _ = ecdsa.GenerateKey(elliptic.P256(), rand.Reader)
var publicKey = &privateKey.PublicKey
var hs = jwt.NewES256(
	jwt.ECDSAPublicKey(publicKey),
	jwt.ECDSAPrivateKey(privateKey),
)
```

编写签发和验证函数.

```go
// 签名
func Sign(id uint, username string) (string, error) {
	now := time.Now()
	pl := LoginToken{
		Payload: jwt.Payload{
			Issuer:         "coolcat",
			Subject:        "login",
			Audience:       jwt.Audience{},
			ExpirationTime: jwt.NumericDate(now.Add(7 * 24 * time.Hour)),
			NotBefore:      jwt.NumericDate(now.Add(30 * time.Minute)),
			IssuedAt:       jwt.NumericDate(now),
			JWTID:          uuid.NewV4().String(),
		},
		ID:       id,
		Username: username,
	}
	token, err := jwt.Sign(pl, hs)
	return string(token), err
}

// 验证
func Verify(token []byte) (*LoginToken, error) {
	pl := &LoginToken{}
	_, err := jwt.Verify(token, hs, pl)
	return pl, err
}
```

### 签发接口

首先, 构建一个接口签发 jwt.

```go
func Login(ctx *gin.Context) {
	var u model.UserModel
	// 应该使用 ShouldBindJSON, 以便使用自定义的 handler.SendResponse
	if err := ctx.ShouldBindJSON(&u); err != nil {
		handler.SendResponse(ctx, errno.New(errno.ErrBind, err), nil)
		return
	}

	user, err := model.GetUserByName(u.Username)
	if err != nil {
		handler.SendResponse(ctx, errno.New(errno.ErrDatabase, err), nil)
		return
	}

	if err := user.Compare(u.Password); err != nil {
		handler.SendResponse(ctx, errno.New(errno.ErrPasswordIncorrect, err), nil)
		return
	}

	// 签发 token
	t, err := token.Sign(user.ID, user.Username)
	if err != nil {
		handler.SendResponse(ctx, errno.New(errno.ErrTokenSign, err), nil)
		return
	}
	handler.SendResponse(ctx, nil, model.Token{Token: t})
}
```

用户传递用户名和密码, 通过验证后返回 jwt.

### 验证中间件

因为验证可能有很多接口都用得到, 所以写成中间件是最自然的方式.

前面介绍过标准的传递 jwt 的方式是存储在 Authorization 请求头中,

```
Authorization: Bearer <token>
```

所以, 这里也依据这种规范来验证 jwt.

```go
// AuthJWT 验证 JWT 的中间件
func AuthJWT() gin.HandlerFunc {
	return func(ctx *gin.Context) {
		header := ctx.GetHeader("Authorization")
		headerList := strings.Split(header, " ")
		if len(headerList) != 2 {
			err := errors.New("无法解析 Authorization 字段")
			handler.SendResponse(ctx, errno.New(errno.ErrTokenInvalid, err), nil)
			ctx.Abort()
			return
		}
		t := headerList[0]
		content := headerList[1]
		if t != "Bearer" {
			err := errors.New("认证类型错误, 当前只支持 Bearer")
			handler.SendResponse(ctx, errno.New(errno.ErrTokenInvalid, err), nil)
			ctx.Abort()
			return
		}
		if _, err := token.Verify([]byte(content)); err != nil {
			handler.SendResponse(ctx, errno.New(errno.ErrTokenInvalid, err), nil)
			ctx.Abort()
			return
		}

		ctx.Next()
	}
}
```

### 使用

定义好中间件后, 就可以在 router 中使用了.

```go
g.POST("/v1/create", user.Create) // 为了方便创建用户, 无需认证

u := g.Group("/v1/user")
u.Use(middleware.AuthJWT()) // 添加认证
{
  u.GET("", user.List)
  u.POST("", user.Create)
  u.GET("/:id", user.Get)
  u.PUT("/:id", user.Save)
  u.PATCH("/:id", user.Update)
  u.DELETE("/:id", user.Delete)
}
```

这里为了方便, 有个创建用户的接口放在了外边, 逃避了 jwt 验证.
不然一开始没有用户又无法创建挺尴尬的.

## 总结

认证与鉴权是 API 接口比不可少的一部分, 这里介绍了 jwt.
更复杂强大的授权协议是 [OAuth 2.0](https://oauth.net/2/),
OAuth 2.0 更多用在协作共享资源上, 对于简单的 API 服务器, jwt 就足够了.
jwt 也可以作为 OAuth 2.0 的一部分, 用于承载内容.

## 当前部分的代码

作为版本 [v0.9.0](https://github.com/zhenhua32/go_web/tree/v0.9.0)
