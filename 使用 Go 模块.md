<!-- TOC -->

- [简介](#简介)
- [使用 Go 模块](#使用-go-模块)
- [模块的常见操作](#模块的常见操作)
  - [创建一个新的模块](#创建一个新的模块)
  - [添加一个依赖](#添加一个依赖)
  - [升级依赖](#升级依赖)
  - [添加对新的主版本的依赖](#添加对新的主版本的依赖)
  - [升级依赖到新的主版本](#升级依赖到新的主版本)
  - [移除未使用的依赖](#移除未使用的依赖)
  - [结论](#结论)
- [迁移到 Go 模块](#迁移到-go-模块)
  - [迁移你的项目到 Go 模块](#迁移你的项目到-go-模块)
    - [已有依赖管理器](#已有依赖管理器)
    - [没有使用包管理器](#没有使用包管理器)
  - [在模块下进行测试](#在模块下进行测试)
  - [发布一个版本](#发布一个版本)
  - [导入和规范模块路径](#导入和规范模块路径)
  - [总结](#总结)

<!-- /TOC -->

## 简介

Go 终于要有自己的模块了, 以前只有包, 而模块是包的上一级.

以下是阅读官网上的两篇文章的总结.

- https://blog.golang.org/using-go-modules
- https://blog.golang.org/migrating-to-go-modules

## 使用 Go 模块

一个模块是存储在文件树中的一系列的 Go 包的集合, 根目录有一个 `go.mod` 文件.

`go.mod` 文件定义了模块的 _module path_, 这也是用于根目录的导入路径.
还定义了 _dependency requirements_, 这是构建需要的外部模块.

## 模块的常见操作

### 创建一个新的模块

在 `$GOPATH/src` 之外, 创建一个目录, 并添加文件 `hello.go`:

```go
package hello

func Hello() string {
    return "Hello, world."
}
```

添加一个测试文件, `hello_test.go`:

```go
package hello

import "testing"

func TestHello(t *testing.T) {
    want := "Hello, world."
    if got := Hello(); got != want {
        t.Errorf("Hello() = %q, want %q", got, want)
    }
}
```

到这里, 该目录包含一个包, 但还不是一个模块, 因为还没有 `go.mod` 文件.

如果在该目录下运行 `go test`:

```bash
$ go test
PASS
ok      _/home/gopher/hello    0.020s
$
```

最后一行总结了所有的包的测试. 因为我们在 `$GOPATH` 目录外, 也在任何模块之外,
所有 `go` 命令对当前目录不知道任何导入路径, 这会制造一个假的基于当前目录名的模块:
`_/home/gopher/hello`.

现在, 使用 `go mod init` 将该目录初始化为一个模块, 并重新运行 `go test`.

```bash
$ go mod init example.com/hello
go: creating new go.mod: module example.com/hello
$ go test
PASS
ok      example.com/hello    0.020s
$
```

这样, 就创建一个 Go 模块, 并运行了测试. 注意到, 模块名字已经变成了 `example.com/hello`.

`go mod init` 命令产生了一个 `go.mod` 的文件.

```bash
$ cat go.mod
module example.com/hello

go 1.12
$
```

`go.mod` 文件只出现在模块的根目录.
在子目录中的包的导入路径为 模块路径 加上 子目录 路径.
如果我们创建了一个叫做 `world` 的子目录, 这个包会自动被识别为
模块 `example.com/hello`的一部分, 它的导入路径为 `example.com/hello/world`.

### 添加一个依赖

Go 模块的主要动力是提高代码被其他开发者作为依赖的体验.

更新 `hello.go`, 导入 `rsc.io/quote` 并使用它来实现 Hello 函数.

```go
package hello

import "rsc.io/quote"

func Hello() string {
    return quote.Hello()
}
```

重新运行测试:

```bash
$ go test
go: finding rsc.io/quote v1.5.2
go: downloading rsc.io/quote v1.5.2
go: extracting rsc.io/quote v1.5.2
go: finding rsc.io/sampler v1.3.0
go: finding golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c
go: downloading rsc.io/sampler v1.3.0
go: extracting rsc.io/sampler v1.3.0
go: downloading golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c
go: extracting golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c
PASS
ok      example.com/hello    0.023s
$
```

`go` 命令使用在 `go.mod` 中列出的特定依赖来解析导入.
当它遇到任何不在 `go.mod` 中的 `import` 导入的包时,
go 命名会自动查找包含这个包的模块, 并将它添加到 `go.mod` 文件中,
并使用最新的版本号.

```bash
$ cat go.mod
module example.com/hello

go 1.12

require rsc.io/quote v1.5.2
$
```

再次运行 `go test` 不会重复上面的过程, 因为 `go.mod` 是最新的,
而且下载的模块已经缓存在本地了, 在 `$GOPATH/pkg/mod`目录下.

添加一个直接的依赖通常会带来一些间接的依赖, 命令 `go list -m all`
列出当前模块和它的所有依赖.

```bash
$ go list -m all
example.com/hello
golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c
rsc.io/quote v1.5.2
rsc.io/sampler v1.3.0
$
```

通常第一行是主模块, 下面都是它的依赖, 按模块路径排序.

`golang.org/x/text` 的版本是 `v0.0.0-20170915032832-14c0d48ead0c`,
这是一个 _伪版本(pseudo-version)_, 这是 go 命令对没有标签的提交的版本语法.

除了 `go.mod` 之外, go 命令还维护了一个叫做 `go.sum` 的文件,
包含每一个特定版本的模块的内容的加密哈希.

```bash
$ cat go.sum
golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c h1:qgOY6WgZO...
golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c/go.mod h1:Nq...
rsc.io/quote v1.5.2 h1:w5fcysjrx7yqtD/aO+QwRjYZOKnaM9Uh2b40tElTs3...
rsc.io/quote v1.5.2/go.mod h1:LzX7hefJvL54yjefDEDHNONDjII0t9xZLPX...
rsc.io/sampler v1.3.0 h1:7uVkIFmeBqHfdjD+gZwtXXI+RODJ2Wc4O7MPEh/Q...
rsc.io/sampler v1.3.0/go.mod h1:T1hPZKmBbMNahiBKFy5HrXp6adAjACjK9...
$
```

go 命令通过使用 `go.sum` 文件来未来的下载和第一次下载一样, 有相同的 bit.
保证你的项目依赖的模块不会发生意外的变化. 这两个文件 `go.mod` 和 `go.sum`
都应该保存在版本控制之中.

### 升级依赖

使用 Go 模块, 版本号使用语义版本标签. 一个语义版本有三个部分: 主版本, 次版本,
补丁版本.

因为前面通过 `go list -m all` 看到 `golang.org/x/text` 使用了一个未标记的版本,
让我们升级到最新的版本, 并测试一切都正常.

```bash
$ go get golang.org/x/text
go: finding golang.org/x/text v0.3.0
go: downloading golang.org/x/text v0.3.0
go: extracting golang.org/x/text v0.3.0
$ go test
PASS
ok      example.com/hello    0.013s
$
```

一切正常, 重新看看 `go list -m al` 的输出, 以及 `go.mod` 文件.

```bash
$ go list -m all
example.com/hello
golang.org/x/text v0.3.0
rsc.io/quote v1.5.2
rsc.io/sampler v1.3.0
$ cat go.mod
module example.com/hello

go 1.12

require (
    golang.org/x/text v0.3.0 // indirect
    rsc.io/quote v1.5.2
)
$
```

`golang.org/x/text` 包已经升级到最新版本了. `go.mod` 文件也更新了.
`indirect` 注释表示这不是一个直接被该模块使用的依赖, 而是间接依赖.

让我们试图更新 `rsc.io/sampler` 的次要版本, 使用相同的方式, 并运行测试:

```bash
$ go get rsc.io/sampler
go: finding rsc.io/sampler v1.99.99
go: downloading rsc.io/sampler v1.99.99
go: extracting rsc.io/sampler v1.99.99
$ go test
--- FAIL: TestHello (0.00s)
    hello_test.go:8: Hello() = "99 bottles of beer on the wall, 99 bottles of beer, ...", want "Hello, world."
FAIL
exit status 1
FAIL    example.com/hello    0.014s
$
```

测试失败了, 这个版本不兼容我们的用例. 看一看这个模块有哪些版本:

```bash
$ go list -m -versions rsc.io/sampler
rsc.io/sampler v1.0.0 v1.2.0 v1.2.1 v1.3.0 v1.3.1 v1.99.99
$
```

换个版本试试, 我们已经使用 `1.3.0` 了, 或许 `1.3.1` 可以兼容.

```go
$ go get rsc.io/sampler@v1.3.1
go: finding rsc.io/sampler v1.3.1
go: downloading rsc.io/sampler v1.3.1
go: extracting rsc.io/sampler v1.3.1
$ go test
PASS
ok      example.com/hello    0.022s
$
```

通过在 `go get` 中指明版本号, 默认是 `@latest`.

### 添加对新的主版本的依赖

添加一个新的功能, 修改 `hello.go` 文件:

```go
package hello

import (
    "rsc.io/quote"
    quoteV3 "rsc.io/quote/v3"
)

func Hello() string {
    return quote.Hello()
}

func Proverb() string {
    return quoteV3.Concurrency()
}
```

然后添加一个新的测试, 在 `hello_test.go` 中:

```go
func TestProverb(t *testing.T) {
    want := "Concurrency is not parallelism."
    if got := Proverb(); got != want {
        t.Errorf("Proverb() = %q, want %q", got, want)
    }
}
```

然后, 重新测试:

```bash
$ go test
go: finding rsc.io/quote/v3 v3.1.0
go: downloading rsc.io/quote/v3 v3.1.0
go: extracting rsc.io/quote/v3 v3.1.0
PASS
ok      example.com/hello    0.024s
$
```

注意到, 我们的模块现在依赖 `rsc.io/quote` 和 `rsc.io/quote/v3`.

```bash
$ go list -m rsc.io/q...
rsc.io/quote v1.5.2
rsc.io/quote/v3 v3.1.0
$
```

每一个 GO 模块的主要版本, 都会使用一个不同的路径: 从 v2 开始, 路径必须以主版本号结尾.
比如 `rsc.io/quote` 的 v3 版本, 不再是 `rsc.io/quote`, 而是独立的名字 `rsc.io/quote/v3`.
这种用法叫做 **语义导入版本化**, 它给了不兼容的包(不同主版本的包)一个不同的名字.

go 命令允许构建包含任何特定模块路径的至多一个版本, 意味着每一个主版本最多一个路径:
一个 `rsc.io/quote`, 一个 `rsc.io/quote/v2`, 一个 `rsc.io/quote/v3`等等.
这给予了模块作者一个清晰的规则, 一个模块路径是否可能出现副本: 一个程序不可能同时使用
`rsc.io/quote v1.5.2` 和 `rsc.io/quote v1.6.0`. 同时, 一个模块的不同主版本允许共存,
能帮助模块的消费者可以渐进地升级到新的主版本上.

在这个例子中, 我们想要使用 `rsc/quote/v3 v3.1.0` 中的 `quote.Concurrency`, 但我们还没
做好准备迁移对 `rsc.io/quote v1.5.2` 的使用.

在大型程序或代码库中, 逐步迁移的能力尤其重要.

### 升级依赖到新的主版本

让我们继续完成迁移, 开始只使用 `rsc.io/quote/v3`. 因为主版本的改变, 我们预计到某些 APIs
可能已经被移除, 重命名, 或者以其他不兼容的方式改变了.

通过阅读文档, 我们发现 `Hello` 已经变成了 `HelloV3`:

```bash
$ go doc rsc.io/quote/v3
package quote // import "rsc.io/quote/v3"

Package quote collects pithy sayings.

func Concurrency() string
func GlassV3() string
func GoV3() string
func HelloV3() string
func OptV3() string
$
```

我们通过升级将 `hello.go` 中的 `quote.Hello()` 改变为 `quoteV3.HelloV3()`.

```go
package hello

import quoteV3 "rsc.io/quote/v3"

func Hello() string {
    return quoteV3.HelloV3()
}

func Proverb() string {
    return quoteV3.Concurrency()
}
```

在这个节点上, 我们无需重命名导入了, 所以可以这样做:

```go
package hello

import "rsc.io/quote/v3"

func Hello() string {
    return quote.HelloV3()
}

func Proverb() string {
    return quote.Concurrency()
}
```

重新运行测试, 一切正常.

### 移除未使用的依赖

我们已经移除了对 `rsc.io/quote` 的使用, 但它依然出现在 `go list -m all`
和 `go.mod` 文件中.

为什么? 因为 `go build` 和 `go test` 可以轻易告诉我们某些东西丢失了,
需要添加, 但无法告诉我们某些东西可以安全地被移除.

移除一个依赖, 只有在检查过模块中的所有包, 和这些包所有可能的构建标签组合
之后才能确定. 一个普通的构建命令是不会加载这些信息的, 所以它不能安全地移除依赖.

`go mod tidy` 可以清除这些未使用的包.

### 结论

在 Go 中 Go 模块是依赖管理的未来. 模块功能在所有支持的 Go 版本中是可用的了(GO 1.11 之后).

总结一下使用 Go 模块的工作流:

- `go mod init` 创建一个新的模块, 初始化 `go.mod` 文件
- `go build, go test` 和其他的包构建命令会添加必要的依赖到 `go.mod` 文件中
- `go list -m all` 打印出当前模块的依赖
- `go get` 改变一个依赖的必要版本
- `go mod tidy` 移除未使用的依赖

## 迁移到 Go 模块

Go 项目有各式各样的依赖管理策略. Vendoring 工具, 比如 dep 或 glide 是
非常流行的, 但它们在行为上非常不同, 且不能很好地兼容.
一些项目在 Git 仓库中存储它们的整个 GOPATH 目录.
其他一些可能仅依赖 `go get` 并期望在 `GOPATH` 中安装最新的依赖.

Go 的模块系统, 在 Go 1.11 中引入, 提供了一个基于 go 命令的官方依赖解决方案.

下面主要讲述了这个工具, 以及将一个项目转变为模块的技术.

注意: 如果你的项目已经在版本 2.0.0 或以上, 当你添加一个 `go.mod` 文件时,
需要升级你的模块路径.

### 迁移你的项目到 Go 模块

一个即将转换到 Go 模块的项目可能处于下面三种情况下:

- 一个全新的 Go 项目
- 一个使用依赖管理器的 Go 项目
- 一个没有任何依赖管理器的 Go 项目

这里将讨论后面两种情况.

#### 已有依赖管理器

转换一个已经有依赖管理器的项目, 使用下面的命令:

```bash
$ git clone https://github.com/my/project
[...]
$ cd project
$ cat Godeps/Godeps.json
{
    "ImportPath": "github.com/my/project",
    "GoVersion": "go1.12",
    "GodepVersion": "v80",
    "Deps": [
        {
            "ImportPath": "rsc.io/binaryregexp",
            "Comment": "v0.2.0-1-g545cabd",
            "Rev": "545cabda89ca36b48b8e681a30d9d769a30b3074"
        },
        {
            "ImportPath": "rsc.io/binaryregexp/syntax",
            "Comment": "v0.2.0-1-g545cabd",
            "Rev": "545cabda89ca36b48b8e681a30d9d769a30b3074"
        }
    ]
}
$ go mod init github.com/my/project
go: creating new go.mod: module github.com/my/project
go: copying requirements from Godeps/Godeps.json
$ cat go.mod
module github.com/my/project

go 1.12

require rsc.io/binaryregexp v0.2.1-0.20190524193500-545cabda89ca
$
```

`go mod init` 会创建一个新的 `go.mod` 文件, 并自动导入依赖,
从 `Godeps.json, Gopkg.lock` 或[其他支持的格式](https://go.googlesource.com/go/+/362625209b6cd2bc059b6b0a67712ddebab312d9/src/cmd/go/internal/modconv/modconv.go#9)中.

`go mod init` 的参数是模块路径, 模块从哪里发现的位置.

在继续之前, 先运行 `go build` 或 `go test`. 后续的步骤可能会修改你的 `go.mod` 文件,
如果你想要迭代更新, 这是最接近你的模块依赖规范的 `go.mod` 文件.

```bash
$ go mod tidy
go: downloading rsc.io/binaryregexp v0.2.1-0.20190524193500-545cabda89ca
go: extracting rsc.io/binaryregexp v0.2.1-0.20190524193500-545cabda89ca
$ cat go.sum
rsc.io/binaryregexp v0.2.1-0.20190524193500-545cabda89ca h1:FKXXXJ6G2bFoVe7hX3kEX6Izxw5ZKRH57DFBJmHCbkU=
rsc.io/binaryregexp v0.2.1-0.20190524193500-545cabda89ca/go.mod h1:qTv7/COck+e2FymRvadv62gMdZztPaShugOCi3I+8D8=
$
```

`go mod tidy` 查找在你的模块中, 确实被导入的包. 它会为不在任何已知模块中的包添加模块依赖,
并删除任何没有提供导入包的模块. 如果一个模块提供的包只被还没有迁移到模块系统的项目使用,
这个模块依赖会使用 `// indirect` 注释标记.

当提交对 `go.mod` 的改动到版本控制中, 一个好习惯是先运行 `go mod tidy`.

最后, 检查代码构建成功, 并通过测试.

```bash
$ go build ./...
$ go test ./...
[...]
$
```

注意, 其他的依赖管理器可能在个人包或者整个仓库的级别上指定依赖关系,
并且通常无法识别 `go.mod` 中指定的依赖. 因此, 你可能无法获得
和以前一样的每个包的完全相同的版本, 并且升级可能发生破坏性的变化.
因此, 通过审计生成的依赖来执行上面的命令非常重要.

运行下面的命令:

```bash
$ go list -m all
go: finding rsc.io/binaryregexp v0.2.1-0.20190524193500-545cabda89ca
github.com/my/project
rsc.io/binaryregexp v0.2.1-0.20190524193500-545cabda89ca
$
```

并通过对比你的旧依赖管理文件来确定选择的版本是恰当的.
如果你发现一个版本不是你想要的, 可以通过 `go mod why -m`
或 `go mod graph` 命令来发现为什么, 并使用 `go get`
升级或降级到正确的版本.

如果需要降级, `go get` 也可能会降级其他依赖来满足最小兼容性.

```bash
$ go mod why -m rsc.io/binaryregexp
[...]
$ go mod graph | grep rsc.io/binaryregexp
[...]
$ go get rsc.io/binaryregexp@v0.2.0
$
```

#### 没有使用包管理器

对于没有使用包管理器的 Go 项目, 从创建 `go.mod` 文件开始:

```bash
$ git clone https://go.googlesource.com/blog
[...]
$ cd blog
$ go mod init golang.org/x/blog
go: creating new go.mod: module golang.org/x/blog
$ cat go.mod
module golang.org/x/blog

go 1.12
$
```

没有先前的依赖管理文件可以参考, 所以 `go mod init` 会创建
一个只有 `module` 和 `go` 指令的 `go.mod` 文件.
在这个例子中, 我们设置模块路径为 `golang.org/x/blog`, 是因为
那是它的 [自定义导入路径](https://golang.org/cmd/go/#hdr-Remote_import_paths).
用户可能使用这个路径导入包, 所以我们必须小心对待, 不要改变它.

`module` 指令声明了模块路径, `go` 指令声明了用来兼容
这个模块的 Go 语言版本.

现在, 运行 `go mod tidy` 添加依赖:

```bash
$ go mod tidy
go: finding golang.org/x/website latest
go: finding gopkg.in/tomb.v2 latest
go: finding golang.org/x/net latest
go: finding golang.org/x/tools latest
go: downloading github.com/gorilla/context v1.1.1
go: downloading golang.org/x/tools v0.0.0-20190813214729-9dba7caff850
go: downloading golang.org/x/net v0.0.0-20190813141303-74dc4d7220e7
go: extracting github.com/gorilla/context v1.1.1
go: extracting golang.org/x/net v0.0.0-20190813141303-74dc4d7220e7
go: downloading gopkg.in/tomb.v2 v2.0.0-20161208151619-d5d1b5820637
go: extracting gopkg.in/tomb.v2 v2.0.0-20161208151619-d5d1b5820637
go: extracting golang.org/x/tools v0.0.0-20190813214729-9dba7caff850
go: downloading golang.org/x/website v0.0.0-20190809153340-86a7442ada7c
go: extracting golang.org/x/website v0.0.0-20190809153340-86a7442ada7c
$ cat go.mod
module golang.org/x/blog

go 1.12

require (
    github.com/gorilla/context v1.1.1
    golang.org/x/net v0.0.0-20190813141303-74dc4d7220e7
    golang.org/x/text v0.3.2
    golang.org/x/tools v0.0.0-20190813214729-9dba7caff850
    golang.org/x/website v0.0.0-20190809153340-86a7442ada7c
    gopkg.in/tomb.v2 v2.0.0-20161208151619-d5d1b5820637
)
$ cat go.sum
cloud.google.com/go v0.26.0/go.mod h1:aQUYkXzVsufM+DwF1aE+0xfcU+56JwCaLick0ClmMTw=
cloud.google.com/go v0.34.0/go.mod h1:aQUYkXzVsufM+DwF1aE+0xfcU+56JwCaLick0ClmMTw=
git.apache.org/thrift.git v0.0.0-20180902110319-2566ecd5d999/go.mod h1:fPE2ZNJGynbRyZ4dJvy6G277gSllfV2HJqblrnkyeyg=
git.apache.org/thrift.git v0.0.0-20181218151757-9b75e4fe745a/go.mod h1:fPE2ZNJGynbRyZ4dJvy6G277gSllfV2HJqblrnkyeyg=
github.com/beorn7/perks v0.0.0-20180321164747-3a771d992973/go.mod h1:Dwedo/Wpr24TaqPxmxbtue+5NUziq4I4S80YR8gNf3Q=
[...]
$
```

`go mod tidy` 为所有在你的模块中直接导入的包添加模块依赖, 并且创建一个 `go.sum` 文件
保存每一个库在特定版本下的校验和. 检查是否通过构建和测试:

```bash
$ go build ./...
$ go test ./...
ok      golang.org/x/blog    0.335s
?       golang.org/x/blog/content/appengine    [no test files]
ok      golang.org/x/blog/content/cover    0.040s
?       golang.org/x/blog/content/h2push/server    [no test files]
?       golang.org/x/blog/content/survey2016    [no test files]
?       golang.org/x/blog/content/survey2017    [no test files]
?       golang.org/x/blog/support/racy    [no test files]
$
```

注意, 当 `go mod tidy` 添加一个依赖时, 它会添加这个模块的最新版本.
如果你的 GOPATH 中包含了一个旧版本的依赖, 那么新版本可能会带来
破坏性的变化, 你可能在 `go mod tidy`, `go build` 或 `go test` 中
看到错误.

如果发生了这种事, 试着用 `go get` 降级版本, 或者将你的模块兼容
每个依赖的最新版本.

### 在模块下进行测试

一些测试可能在迁移到 Go 模块后需要调整.

如果一个测试需要在包目录下写入文件, 当这个包目录位于模块缓存中时可能失败,
因为它是只读的. 在特定情况下, 这可能导致 `go test all` 失败.
测试应该复制它需要的文件并写到到一个临时目录中.

如果一个测试依赖相对路径(../package-in-another-module) 来定位并
读取在另一个包中的文件, 如果包在另一个模块中, 测试会失败.
这个路径会被定位在模块缓存的版本化子目录中, 或者一个被 `replace`
指令指定的路径中.

在这种情况下, 你可能需要复制测试输入到你的模块中, 或者将测试输入
从原始文件转为嵌入到 .go 源文件的数据.

如果测试需要 go 命令在 GOPATH 模式下运行测试, 它可能失败.
在这种情况下, 你可能需要添加一个 `go.mod` 文件到要被测试的源代码树中,
或者明确地设置 `GO111MODULE=off`.

### 发布一个版本

最终, 你应该为你的模块标记一个版本并发布. 如果你还没有发布任何版本, 这是可选的,
但是没有正式发布的版本, 下游的使用者在特定的提交上使用 **伪版本**,
这可能更难以支持.

```bash
$ git tag v1.2.0
$ git push origin v1.2.0
```

你的新 `go.mod` 文件为你的模块定义了一个规范的导入路径, 并添加了最低的依赖版本.
如果你的用户已经使用正确的导入路径, 你的依赖也没有破坏性的改动,
那么添加 `go.mod` 文件是向后兼容的. 但它是一个重大的改变, 可能暴露已存在的问题.

如果你已经现存的版本了, 你应该增加主版本号.

### 导入和规范模块路径

每一个模块在 `go.mod` 中声明了它的模块路径.
引用模块中的包的每个 import 语句必须将模块路径作为包路径的前缀.
但是, go 命令可能遇到一个包含许多不同
[远程导入路径](https://golang.org/cmd/go/#hdr-Remote_import_paths) 的仓库.
举个例子, `golang.org/x/lint` 和 `github.com/golang/lint` 都可以
解析为托管在 `go.googlesource.com/lint` 中的代码仓库.
`go.mod` 文件包含的仓库声明为 `golang.org/x/lint`, 所以只有这条路径
对应了一个有效的模块.

Go 1.4 提供了一个机制, 用于声明规范的导入路径, 通过
[`// import` 注释](https://golang.org/cmd/go/#hdr-Import_path_checking),
但包作者并不总是提供它们.
因此, 在模块出现之前编写的代码可能使用了不规范的模块导入路径,
却没有因为路径不匹配而出现错误.
当使用模块后, 导入路径必须匹配规范的模块路径, 所以你需要更新 import 语句.
举个例子, 你可能需要将 `import "github.com/golang/lint"` 改为
`import "golang.org/x/lint"`.

另一个问题是, 对于主版本大于等于 2 的模块, 模块的规范路径和它的存储库路径不一致.
一个主版本大于 1 的 Go 模块, 必须在它的模块路径之后加上主版本号作为后缀.
比如版本 `2.0.0` 必须添加后缀 `/v2`. 但是 import 语句可能引用了
模块中的包, 但没有提供版本后缀.
举个例子, 非模块用户在可能在版本 v2.0.1 中使用了
`github.com/russross/blackfriday`
而不是
`github.com/russross/blackfriday/v2`,
需要更新导入路径包含后缀 `/v2`.

### 总结

对大多数用户而言, 转换为 Go 模块应该是一个简单的过程.
因为非规范的导入路径或者破坏性的依赖可能偶然导致出错.
