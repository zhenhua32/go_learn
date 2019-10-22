<!-- TOC -->

- [简介](#简介)
- [实践](#实践)
  - [生成证书和私钥](#生成证书和私钥)
  - [修改配置文件](#修改配置文件)
  - [修改启动函数](#修改启动函数)
- [总结](#总结)
- [当前部分的代码](#当前部分的代码)

<!-- /TOC -->

## 简介

现在的网站没有 HTTPS 都不好意思见人了.

> 超文本传输安全协议（英语：HyperText Transfer Protocol Secure，缩写：HTTPS；常称为 HTTP over TLS、HTTP over SSL 或 HTTP Secure）是一种通过计算机网络进行安全通信的传输协议。HTTPS 经由 HTTP 进行通信，但利用 SSL/TLS 来加密数据包。HTTPS 开发的主要目的，是提供对网站服务器的身份认证，保护交换数据的隐私与完整性。这个协议由网景公司（Netscape）在 1994 年首次提出，随后扩展到互联网上。

> HTTPS 的信任基于预先安装在操作系统中的证书颁发机构（CA）。因此，到一个网站的 HTTPS 连接仅在这些情况下可被信任：
>
> - 浏览器正确地实现了 HTTPS 且操作系统中安装了正确且受信任的证书颁发机构；
> - 证书颁发机构仅信任合法的网站；
> - 被访问的网站提供了一个有效的证书，也就是说它是一个由操作系统信任的证书颁发机构签发的（大部分浏览器会对无效的证书发出警告）；
> - 该证书正确地验证了被访问的网站（例如，访问https://example.com时收到了签发给example.com而不是其它域名的证书）；
> - 此协议的加密层（SSL/TLS）能够有效地提供认证和高强度的加密。

主要目的在于:

- 安全传输数据
- 防止中间人攻击和窃听
- 验证服务器的可信度

## 实践

在 Go 中使用 HTTPS 也很简单, 接口如下:

```go
func (srv *Server) ListenAndServe() error
func (srv *Server) ListenAndServeTLS(certFile, keyFile string) error
```

对比一下就知道了, 只需要两个参数就可以实现 HTTPS 了.

这两个参数分别是证书文件的路径和私钥文件的路径.
通常要获取这两个文件需要从证书颁发机构获取.
虽然有免费的, 但还是比较麻烦, 通常还需要域名.

为了简单起见, 这里使用自签名证书, 当然, 这样的证书是不会被浏览器信任的.

### 生成证书和私钥

如果是 window 系统, 可以在 git bash 中运行.
`MSYS_NO_PATHCONV=1` 是专为 git bash 设置的环境变量, 没有的话会报错.

```bash
MSYS_NO_PATHCONV=1 openssl req -new -nodes -x509 -out server.crt -keyout server.key -days 3650 -subj "/C=CN/ST=SH/L=SH/O=CoolCat/OU=CoolCat Software/CN=127.0.0.1/emailAddress=coolcat@qq.com"
```

PowerShell 版本, 需要指定配置路径 `-config`, 默认应该是 `"C:\Program Files\Git\usr\ssl\openssl.cnf"`.

```bash
openssl req -config "C:\Program Files\Git\usr\ssl\openssl.cnf" -new -nodes -x509 -out server.crt -keyout server.key -days 3650 -subj "/C=CN/ST=SH/L=SH/O=CoolCat/OU=CoolCat Software/CN=127.0.0.1/emailAddress=coolcat@qq.com"
```

Linux 下就可以直接运行吧.

```bash
openssl req -new -nodes -x509 -out server.crt -keyout server.key -days 3650 -subj "/C=CN/ST=SH/L=SH/O=CoolCat/OU=CoolCat Software/CN=127.0.0.1/emailAddress=coolcat@qq.com"
```

这个命令会在当前目录生成 server.crt 证书文件和 server.key 私钥文件, 都复制到项目的 conf 目录下.

### 修改配置文件

在配置文件 `conf/config.yaml` 中添加 HTTPS 相关的参数.

```yaml
tls:
  addr: :443 # HTTPS 绑定端口
  cert: conf/server.crt # 自签发的数字证书
  key: conf/server.key # 私钥文件
```

HTTPS 的默认端口就是 443, 这里也配置成 443, 就可以在 URL 中省略端口号了.

### 修改启动函数

一开始, 启动函数是在 goroutine 中启动 HTTP 服务器,
这里增加一个 goroutine 来启动 HTTPS 服务器.

```go
// 启动 https 服务
cert := viper.GetString("tls.cert")
key := viper.GetString("tls.key")
addrTLS := viper.GetString("tls.addr")
if cert != "" && key != "" {
  go func() {
    // 等待 http 服务正常启动
    <-wait
    logrus.Infof("启动服务器在 https address: %s", addrTLS)
    srv.Addr = addrTLS
    if err := srv.ListenAndServeTLS(cert, key); err != nil && err != http.ErrServerClosed {
      logrus.Fatalf("listen on https: %s\n", err)
    }
  }()
}
```

启动之后, 就可以通过 `https://127.0.0.1/v1/check/cpu` 验证一下了,
浏览器中打开肯定是会显示不安全的, 因为证书无法通过验证.

## 总结

HTTPS 是一种趋势, 也是未来. API 接口为了安全性, 一般都是需要上 HTTPS 的.

而且在 Go 中使用 HTTPS 也挺简单的, 换个 TLS 结尾的函数就行了.
也可以只使用 HTTPS, 禁止 HTTP 对外服务,
修改 HTTP 的配置参数 `addr` 为 `localhost:port` 就行.

## 当前部分的代码

作为版本 [v0.10.0](https://github.com/zhenhua32/go_web/tree/v0.10.0)
