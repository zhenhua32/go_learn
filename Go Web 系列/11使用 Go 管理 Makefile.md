<!-- TOC -->

- [简介](#简介)
- [Makefile](#makefile)
- [实践](#实践)
- [总结](#总结)
- [当前部分的代码](#当前部分的代码)

<!-- /TOC -->

## 简介

很多时候, 我们需要运行多个命令来能完成一件事,
又或者某个命令需要指定很多参数.

这个时候, 就需要使用脚本来取代这些复杂的命令,
减少输错命令的可能, 也可以为后来者指明常用的操作.

## Makefile

Makefile 就是为此而生的, 相对于用途广泛的 shell 脚本,
Makefile 专注于构建自动化过程, 通常用于编译源码等.
很多项目都会提供 Makefile 文件, 只需要简单地运行
`make` 就能轻松完成编译构建的过程.

简单介绍下 Makefile 的规则.

```
target: dependencies
    system command(s)
```

target 通常是程序要生成的目标文件的名字. 但也可以是一个动作的名字.

dependencies 是依赖, 通常是文件, 完成 target 所需要的输入.

system command(s) 是完成 target 所需要运行的指令, 即 shell 命令.
一条语句一行, 使用单个 tab 缩进.

使用 make 命令可以运行各种 target. 如果不带 target 参数,
第一个 target 会被作为默认目标.

很多时候, Makefile 不是为了编译, 也不再引用任何文件,
仅仅是为了整合多个命令, 比写脚本方便多.
这个时候涉及到一个叫做伪目标的指令 `.PHONY`.
`.PHONY` 后面跟着的多个 target 都不是要生成的文件的名字,
而是指代一个动作, 一个行为. 比如 test 指运行测试, clean 清理文件等.

```
.PHONY: all test clean doc ci
```

更多内容可以参考
[跟我一起写 Makefile](https://seisman.github.io/how-to-write-makefile/index.html)

## 实践

**注意, windows 下没有 make 命令, 所以 Makefile 也就无法使用.**

你可以在 docker 容器中运行命令, 可以参考另一篇文章
`在 VS Code 中使用容器开发`.

在项目的根目录添加 `Makefile` 文件:

```makefile
all: gotool build
build:
	@go build ./
run:
	@go run ./
clean:
	rm -f web
	find . -name "[._]*.s[a-w][a-z]" | xargs -i rm -f {}
gotool:
	go fmt ./
	go vet ./
ca:
	MSYS_NO_PATHCONV=1 openssl req -new -nodes -x509 -out conf/server.crt -keyout conf/server.key -days 3650 -subj "/C=CN/ST=SH/L=SH/O=CoolCat/OU=CoolCat Software/CN=127.0.0.1/emailAddress=coolcat@qq.com"
mysql:
	docker-compose up -d mysql
dbcli:
	docker-compose run --rm dbclient

help:
	@echo "make - 格式化 Go 代码, 并编译生成二进制文件"
	@echo "make build - 编译 Go 代码, 生成二进制文件"
	@echo "make run - 直接运行 Go 代码"
	@echo "make clean - 移除二进制文件和 vim swap files"
	@echo "make gotool - 运行 Go 工具 'fmt' and 'vet'"
	@echo "make ca - 生成证书文件"
	@echo "make mysql - 启动 mysql 服务器"
	@echo "make dbcli - 连接到 mysql 命令行"

.PHONY: all build run clean gotool ca mysql dbcli help
```

这里的所有 target 都是伪目标.用来包装一些简单的 shell 命令.

可以在项目根目录下运行以下命令:

- make - 格式化 Go 代码, 并编译生成二进制文件
- make build - 编译 Go 代码, 生成二进制文件
- make run - 直接运行 Go 代码
- make clean - 移除二进制文件和 vim swap files
- make gotool - 运行 Go 工具 'fmt' and 'vet'
- make ca - 生成证书文件
- make mysql - 启动 mysql 服务器
- make dbcli - 连接到 mysql 命令行
- make help - 查看帮助信息

有了 Makefile 的帮助, 很多事情变得简单起来了,
比如要生成证书文件, 只需要运行 `make ca` 就行了,
不用输入一大行命令了.

## 总结

Makefile 是 linux 下常用的工具, 对于提升效率是非常有效的.

## 当前部分的代码

作为版本 [v0.11.0](https://github.com/zhenhua32/go_web/tree/v0.11.0)
