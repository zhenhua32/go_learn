<!-- TOC -->

- [起因](#%E8%B5%B7%E5%9B%A0)
- [尝试](#%E5%B0%9D%E8%AF%95)
- [原因](#%E5%8E%9F%E5%9B%A0)
- [解决](#%E8%A7%A3%E5%86%B3)
- [总结](#%E6%80%BB%E7%BB%93)

<!-- /TOC -->

## 起因

今天在新机器上启动 Django 的时候报了个数据库无法连接的错误.

```bash
django.db.utils.OperationalError: (2002, "Can't connect to local MySQL server through socket '/tmp/mysql.sock' (2)")
```

一开始的确很慌张, 尤其是脑袋中没有这个概念之后. 后来想起来的确是一个非常基础的概念.

## 尝试

数据库连不上嘛, 赶紧试试在命令行中连接, 喔, 一切正常?

再去看看 Django 的设置, 似乎也没什么问题.

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'user',
        'USER': 'root',
        'PASSWORD': '123456',
        'HOST': 'localhost',
        'PORT': 3306,
    }
}
```

经验告诉我, 可能是 HOST 设置德不对, 试试 `127.0.0.1`.

果然, 更改设置之后, 正常启动了.

## 原因

> A MySQL client on Unix can connect to the mysqld server in two different ways: By using a Unix socket file to connect through a file in the file system (default /tmp/mysql.sock), or by using TCP/IP, which connects through a port number. A Unix socket file connection is faster than TCP/IP, but can be used only when connecting to a server on the same computer. A Unix socket file is used if you do not specify a host name or if you specify the special host name localhost.

https://dev.mysql.com/doc/refman/8.0/en/can-not-connect-to-server.html

官方文档中的第一段话, 大意就是 Unix 中有两种连接 mysql 的方式.
一种是 Unix socket file, 另一种 TCP/IP 连接.
Unix socket file 会在使用 localhost 作为 host name 的时候启用.

> A Unix domain socket or IPC socket (inter-process communication socket) is a data communications endpoint for exchanging data between processes executing on the same host operating system.

https://en.wikipedia.org/wiki/Unix_domain_socket

Unix domain socket 是用于在同一主机上的操作系统的进程间交换数据.

所以, 当使用 localhost 的时候, 使用的是 Unix socket file 连接, mysql 默认的连接地址是 `/tmp/mysql.sock`.

明白了这点之后, 原因就显然易见了, 对 Django 的错误提示也更好理解了, sock 文件的路径不存在.

## 解决

mysqladmin 工具可以查看基本的信息, 包括 sock 的路径.

```bash
sudo mysqladmin version -p123456
```

没有指定 `-h 127.0.0.1` 会使用 socket 连接, 所以能看到 socket 文件地址.

可以选择在连接数据库的时候指定 socket file 路径.

另一种方式是修改 mysql 的配置, 设置为 `/tmp/mysql.sock`.

```ini
[mysqld]
socket=/usr/local/var/mysql.sock
[client]
socket=/usr/local/var/mysql.sock
```

## 总结

很多时候还是因为知识储备不足, 连错误提示都看不太明白, 解决问题就无从下手了.
提升基本功还是非常有必要的.
