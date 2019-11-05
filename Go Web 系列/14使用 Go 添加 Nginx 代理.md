<!-- TOC -->

- [简介](#简介)
- [反向代理](#反向代理)
- [负载均衡](#负载均衡)
  - [轮询](#轮询)
  - [加权轮询](#加权轮询)
  - [最少连接](#最少连接)
  - [iphash](#iphash)
  - [通用 hash](#通用-hash)
- [总结](#总结)
- [当前部分的代码](#当前部分的代码)

<!-- /TOC -->

## 简介

Nginx 是一个高性能的 HTTP 服务器和反向代理服务器.

最常用的两个功能是反向代理和负载均衡.

## 反向代理

反向代理是正向代理的反面.

普通的代理服务器是需要用户主动去设置的, 用户在自己的电脑上设置并连接代理服务器,
从而可以隐藏自己的 IP, 使得应用服务器不知道客户端的 IP 地址.

而反向代理是作为应用服务器的代理, 安装在服务器上. 客户端实际上访问的反向代理服务器,
反向代理服务器再去访问实际的应用服务器, 然后将获取到的响应传送给客户端.

![反向代理](https://upload.wikimedia.org/wikipedia/commons/thumb/6/67/Reverse_proxy_h2g2bob.svg/420px-Reverse_proxy_h2g2bob.svg.png)

使用 Nginx 配置反向代理非常简单, 基础配置如下:

```nginx
upstream web {
  server 127.0.0.1:8081;
}

server {
  listen 80;
  server_name web.coolcat.com;

  location / {
    proxy_set_header Host $http_host;
    proxy_set_header X-Forwarded-Host $http_host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Real-IP $remote_addr;

    client_max_body_size 5m;

    proxy_pass http://web;
  }
}
```

上面的配置文件中设置了一个域名 `web.coolcat.com`,
对这个域名的所有请求都会转发到 `http://web` 上.

通过配置 `upstream`, 我们可以发现, 实际上的流量都被转发到了
`127.0.0.1:8081` 上了.

如此一来, 就实现了反向代理.

## 负载均衡

负载均衡和反向代理是分不开的, 负载均衡通常都是基于反向代理做的.

所谓的负载均衡, 指的是将多个请求转发到不同的后端服务器上.

```nginx
upstream web {
  server 127.0.0.1:8081;
}
```

在上面的反向代理配置中, 只设置了一个后端服务器地址,
如果再添加几个, 就实现了最简单的负载均衡了.

### 轮询

轮询策略按顺序分配请求.

```nginx
upstream web {
  server 192.168.1.1:8081;
  server 192.168.1.2:8081;
}
```

### 加权轮询

加权策略按比例分配请求.

```nginx
upstream web {
  server 192.168.1.1:8081 weight=4;
  server 192.168.1.2:8081 weight=6;
}
```

上面的两个服务器的访问概率就是四六开.

### 最少连接

最少连接将请求分配给当前连接数最少的服务器.

```nginx
upstream web {
  least_conn;
  server 192.168.1.1:8081;
  server 192.168.1.2:8081;
}
```

### ip_hash

来自同一个 IP 的连接都会分配给同一个服务器, 通常用于 `会话保持`.

```nginx
upstream web {
  ip_hash;
  server 192.168.1.1:8081;
  server 192.168.1.2:8081;
}
```

### 通用 hash

使用 hash 自定义要计算的 key. 示例中使用请求地址.
可以选择 `consistent` 参数可以指定使用一致性哈希算法.

```nginx
upstream web {
  hash $request_uri;
  # hash $request_uri consistent;
  server 192.168.1.1:8081;
  server 192.168.1.2:8081;
}
```

参考:

- [Using nginx as HTTP load balancer](https://nginx.org/en/docs/http/load_balancing.html)
- [Module ngx_http_upstream_module](https://nginx.org/en/docs/http/ngx_http_upstream_module.html)

## 总结

Nginx 是很常用的代理服务器, 它的功能非常强大, 性能也很好.
更多的资料请参考 [官方文档](https://nginx.org/en/docs/).

## 当前部分的代码

作为版本 [v0.14.0](https://github.com/zhenhua32/go_web/tree/v0.14.0)
