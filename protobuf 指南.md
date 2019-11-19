<!-- TOC -->

- [简介](#简介)
- [安装](#安装)
- [语言定义](#语言定义)
  - [特殊指令](#特殊指令)
  - [定义服务](#定义服务)
  - [JSON 支持](#json-支持)
  - [选项](#选项)
  - [生成代码](#生成代码)
  - [基础类型](#基础类型)
  - [更新 message](#更新-message)
- [Golang 下使用](#golang-下使用)
- [参考](#参考)

<!-- /TOC -->

## 简介

Protocol Buffers 是 google 出品的一种数据交换格式, 缩写为 protobuf.

主要介绍 proto3 版本和 Golang 下的使用.

## 安装

protobuf 分为编译器和运行时两部分. 编译器直接使用预编译的二进制文件即可,
可以从 [releases](https://github.com/protocolbuffers/protobuf/releases) 上下载.

protobuf 运行时就是不同语言对应的库, 以 Golang 为例:

```bash
go get github.com/golang/protobuf/protoc-gen-go
```

## 语言定义

protobuf 现在有两个版本, proto2 和 proto3. 本着学新不学旧的原则, 这里只介绍 proto3.

既然是一种数据交换格式, 必然是要学习它的语法的, 就像学习 JSON 一样, 你总得知道如何定义.

默认的文件是 `.proto`.

下面是一个简单的例子, 来自于官方文档.

```protobuf
syntax = "proto3";

message SearchRequest {
  string query = 1;
  int32 page_number = 2;
  int32 result_per_page = 3;
}
```

首行定义了语法版本, 即使用 proto3 版本. 然后定义了一个名为 SearchRequest 的 message, 以及它包含的字段(键值对).
message 的结构非常类似于各种语言中的 struct, dict 或者 map.

**每个字段包括三个部分, 类型, 字段名和字段编号**. 前两个部分非常易懂, 主要解释一下字段编号.
**在同一个 message 中字段编号应该是唯一的**, 用于在 message 的二进制格式(message binary format)中标识字段.
因此数字的大小决定了编码的长度, 1-15 的数字只占用一个字节.

注释语法是 `//` 和 `/* ... */`.

### 特殊指令

使用 `reserved` 注明已被废弃的字段编号和字段名称.

```protobuf
message Foo {
  reserved 2, 15, 9 to 11;
  reserved "foo", "bar";
}
```

使用 `repeated` 可以指定重复字段, 即数组.

```protobuf
message Test4 {
  repeated int32 d = 4 [packed=true];
}
```

使用 `enum` 定义枚举类型. 每个枚举定义都必须包含一个映射值为 0 的常量作为第一个元素.

```protobuf
message SearchRequest {
  string query = 1;
  int32 page_number = 2;
  int32 result_per_page = 3;
  enum Corpus {
    UNIVERSAL = 0;
    WEB = 1;
    IMAGES = 2;
    LOCAL = 3;
    NEWS = 4;
    PRODUCTS = 5;
    VIDEO = 6;
  }
  Corpus corpus = 4;
}
```

定义枚举时, 可以使用 `allow_alias` 选项允许枚举值的别名.

```protobuf
enum EnumAllowingAlias {
  option allow_alias = true;
  UNKNOWN = 0;
  STARTED = 1;
  RUNNING = 1;  // RUNNING 是 STARTED 的别名
}
```

类型也可以嵌套使用.

```protobuf
message SearchResponse {
  repeated Result results = 1;
}

message Result {
  string url = 1;
  string title = 2;
  repeated string snippets = 3;
}
```

使用 `import` 可以导入外部定义.

```protobuf
import "myproject/other_protos.proto";
```

使用 `import public` 可以传递导入依赖, 通常用于被导入的 proto 文件需要更改的情况下.

```protobuf
import public "new.proto";
```

protocol 编译器搜索的位置是命令行参数中的 `-I/--proto_path` flag. 如果没有提供, 则搜索编译器被调用时所在的目录.

`Any` 类型可以让你可以在没有它们的 proto 定义时, 将 messages 用作内嵌的类型. 一个 `Any` 包括任意的二进制序列化的
message, 就像 bytes 类型那样, 以及用作该类型的全局唯一的标识符 URL.

```protobuf
import "google/protobuf/any.proto";

message ErrorStatus {
  string message = 1;
  repeated google.protobuf.Any details = 2;
}
```

默认的类型 URL 是 `type.googleapis.com/packagename.messagename`.

如果你有一个 message 包括许多字段, 但同时最多只有一个字段会被设置, 可以使用 `Oneof` 特性来节省内存.
Oneof 字段和普通的字段没有区别, 除了所有这些 Oneof 字段共享内存, 且同时只能由一个字段被设置.
你可以使用特殊的 `case()` 或 `WhichOneof()` 方法检查哪个字段被设置了, 具体方法名称取决于实现的语言.

```protobuf
message SampleMessage {
  oneof test_oneof {
    string name = 4;
    SubMessage sub_message = 9;
  }
}
```

可以在 Oneof 中添加任何类型的字段, 但不能使用 `repeated`.

使用 `map` 可以创建关联映射.

```protobuf
map<key_type, value_type> map_field = N;
map<string, Project> projects = 3;
```

key_type 可以是 integral 或 string 类型, 即 scalar 类型中除了 floating point types 和 bytes 以外的类型.
value_type 可以是除 map 以外的任何类型.

使用 `package` 可以设置命名空间, 防止 message 类型冲突.

```protobuf
package foo.bar;
message Open { ... }
```

在另一个 proto 文件中使用.

```protobuf
message Foo {
  ...
  foo.bar.Open open = 1;
  ...
}
```

package 说明符会影响不同语言生成代码的方式:

- Python: 忽略 package 指令
- Golang: package 用作 Go 包名, 除了你使用 `option go_package` 显式声明包名

### 定义服务

如果你想要和 RPC 系统集成使用, 你可以定义 RPC 服务接口, 编译器会根据选择的语言, 生成对应的服务接口代码和存根.

```protobuf
service SearchService {
  rpc Search (SearchRequest) returns (SearchResponse);
}
```

最常见的是和 gRPC 一起使用.

### JSON 支持

proto3 支持 JSON 中的规范编码, 具体的类型转换关系, 查看[官方文档](https://developers.google.com/protocol-buffers/docs/proto3#json).

### 选项

有很多选项可以调节特性的行为, 完整的选项列表定义在 `google/protobuf/descriptor.proto`.

### 生成代码

要生成代码, 需要使用下面的命令, 对于 Golang 需要安装额外的插件.

```bash
protoc --proto_path=IMPORT_PATH --cpp_out=DST_DIR --java_out=DST_DIR --python_out=DST_DIR --go_out=DST_DIR --ruby_out=DST_DIR --objc_out=DST_DIR --csharp_out=DST_DIR path/to/file.proto
```

### 基础类型

标量值类型和各种语言间的转换可以参考 [官方文档](https://developers.google.com/protocol-buffers/docs/proto3#scalar).

包括以下类型:

```
double
float
int32
int64
unit32
unit64
sint32  编码负数比 int32 更高效
sint64
fixed32 固定四字节. 如果数字通常大于 2^28, 比 uint32 更高效
fixed64
sfixed32 负数优化版本的 fixed32
sfixed64
bool
string
bytes
```

### 更新 message

官方指南翻译

- 不要更改任何现有字段的字段编号.
- 如果添加新的字段, 仍然可以使用新生成的代码解析旧的 message 格式. 但应该自行处理新加字段的默认值.
  使用新代码创建的 message 也可以被旧代码解析, 那些新字段会被忽略.
- 字段可以被删除, 只要不要复用那些字段编号. 要重命名字段, 可以给旧字段添加 `OBSOLETE_` 前缀,
  或者将字段编号设置为 `reserved`.
- int32, uint32, int64, uint64, bool 是相互兼容的类型.
- sint32 和 sint64 是相互兼容的, 但不兼容其他 int 类型.
- string 和 bytes 可以是兼容的, 只要 bytes 是有效的 UTF-8.
- 嵌入的 message 可以和 bytes 兼容, 如果 bytes 包含 message 的编码后的版本.
- fixed32 和 sfixed32 兼容, fixed64 和 sfixed64 兼容.
- enum 在 wire 格式上兼容 int32, uint32, int64, uint64(如果值不合适会被截断). 当 message 被反序列化时,
  客户端代码可以用不同的方式对待它们. 比如, 未识别的 proto3 的 enum 类型将会保存在 message 中, 但它如何
  表示是语言特定的. int 字段总是会保留它的值.
- 将一个单个值更改为新 `oneof` 的成员是安全的且二进制兼容的. 移动多个字段到一个新的 `oneof` 可能是安全的,
  如果你确保没有 code 被多次设置. 移动任何字段到一个已存在的 `oneof` 是不安全的.

## Golang 下使用

定义 protobuf 文件 `hello.proto` 的内容为:

```protobuf
syntax = "proto3";

import "google/protobuf/any.proto";

package hello;
option go_package = "hello";

message HelloReq {
  string name = 1;
}

message HelloResp {
  int32 code = 1;
  string greet = 2;
  google.protobuf.Any details = 3;
}

service HelloService {
  rpc Greet(HelloReq) returns (HelloResp);
}
```

初始化项目, 并生成文件:

```powershell
go mod init tzh.com/app
go get github.com/golang/protobuf/protoc-gen-go
mkdir hello
# 假设 protoc3 已经解压好了
.\protoc3\bin\protoc.exe  --proto_path=. --go_out=./hello hello.proto
```

main.go 如下:

```go
package main

import (
	"crypto/rand"
	"fmt"

	"github.com/golang/protobuf/proto"
	"github.com/golang/protobuf/ptypes/any"
	hello "tzh.com/app/hello"
)

func main() {
	req := &hello.HelloReq{
		Name: "hello",
	}
	details := make([]byte, 10)
	rand.Read(details)
	resp := &hello.HelloResp{
		Code:    1,
		Greet:   "hello name",
		Details: &any.Any{Value: details},
	}
	fmt.Println(req.String())
	fmt.Println(resp.String())

	// 序列化
	data, _ := proto.Marshal(req)
	fmt.Println(data)

	// 反序列化
	newReq := &hello.HelloReq{}
	proto.Unmarshal(data, newReq)
	fmt.Println(newReq)

	fmt.Println("text format", proto.MarshalTextString(req))

}
```

如果你去看生成的代码, 会发现没有 `HelloService` 相关的内容, 这是因为没有使用 gRPC.

## 参考

- [github](https://github.com/protocolbuffers/protobuf)
- [go protobuf](http://github.com/golang/protobuf)
- [proto3](https://developers.google.com/protocol-buffers/docs/proto3)
- [Go Generated Code](https://developers.google.com/protocol-buffers/docs/reference/go-generated)
