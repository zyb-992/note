# gRPC

> [语言指南 (proto3)  | Protocol Buffers  | Google Developers](https://developers.google.cn/protocol-buffers/docs/proto3#specifying_field_types)

## 使用前提

### 安装二进制文件以及下载需要的包

#### 相关命令

```go
//
go get google.golang.org/grpc

//
go get google.golang.org/protobuf

//
go install google.golang.org/protobuf/cmd/protoc-gen-go

//
go get google.golang.org/grpc/cmd/protoc-gen-go-grpc

// protoc-gen-go-grpc用于*.proto-->*_grpc.pb.go
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc

```

- 安装protocol buffers
  - https://github.com/protocolbuffers/protobuf/releases

- 将`$GOPATH/bin1`以及`protocol buffers`下的`bin`目录添加到系统环境变量中

#### 可能出现的错误

1. 使用`protoc --go_out=plugins=grpc:. helloworld.proto`，
   1. 这种生成方式，使用的就是`github`版本的`protoc-gen-go`，而目前这个项目已经由Google接管了。
   2. 如果使用这种生成方式的话，并不会生成`xxx_grpc.pb.go`与`xxx.pb.go`两个文件，只会生成`xxx.pb.go`文件。

#### 相关数据类型

1. `enum`枚举类型会被编译生成为常量以及对应的两种`map`类型

   ```go
   enum I {
       OK = 0;
       NOK = 1;
     }
   
   // ----->
   type MessageRequest_I int32
   
   const (
   	MessageRequest_OK  MessageRequest_I = 0
   	MessageRequest_NOK MessageRequest_I = 1
   )
   
   // Enum value maps for MessageRequest_I.
   var (
   	MessageRequest_I_name = map[int32]string{
   		0: "OK",
   		1: "NOK",
   	}
   	MessageRequest_I_value = map[string]int32{
   		"OK":  0,
   		"NOK": 1,
   	}
   )
   
   func (x MessageRequest_I) Enum() *MessageRequest_I {
   	p := new(MessageRequest_I)
   	*p = x
   	return p
   }
   
   func (x MessageRequest_I) String() string {
   	return protoimpl.X.EnumStringOf(x.Descriptor(), protoreflect.EnumNumber(x))
   }
   
   func (MessageRequest_I) Descriptor() protoreflect.EnumDescriptor {
   	return file_yb_proto_enumTypes[0].Descriptor()
   }
   
   func (MessageRequest_I) Type() protoreflect.EnumType {
   	return &file_yb_proto_enumTypes[0]
   }
   
   func (x MessageRequest_I) Number() protoreflect.EnumNumber {
   	return protoreflect.EnumNumber(x)
   }
   
   // Deprecated: Use MessageRequest_I.Descriptor instead.
   func (MessageRequest_I) EnumDescriptor() ([]byte, []int) {
   	return file_yb_proto_rawDescGZIP(), []int{0, 0}
   }
   ```

   

## 使用其他proto文件的消息类型

​	例如我这里有一个`old.proto`文件，里面有一个`message`需要引用到`new.proto`文件里面的一个`message`，则需要在`old.proto`文件里面导入`new.proto`文件

```go
// old.proto
import "myproject/new.proto"
message old {
}

// new.proto
message new {
}
```

![](D:\Program Files\电子书\go\md\图片\image-20221217155425626.png)



## 编译命令

```shell
protoc --proto_path=IMPORT_PATH --go_out=DST_DIR path/to/file.proto
protoc --proto_path=IMPORT_PATH --go-grpc_out=plugins=grpc:DST_DIR path/to/file.proto
```

- *proto_path=IMPORT_PATH*,IMPORT_PATH是 .proto 文件所在的路径,如果忽略则默认当前目录

- --go_out生成`pb.go`文件

- --go-grpc_out生成`file_grpc.pb.go`文件
- DST_DIR：生成`pb.go`文件的文件路径（可使用相对路径）
- path/to/file.proto：源proto文件

## 定义服务

如果要将消息类型与 RPC（远程过程调用）系统配合使用，您可以在 `.proto` 文件中定义 RPC 服务接口，协议缓冲区编译器会生成以您选择的语言编写的服务接口代码和存根。例如，如果您想使用接受 `SearchRequest` 并返回 `SearchResponse` 的方法定义 RPC 服务，可以在 `.proto` 文件中定义它，如下所示：

```go
service SearchService {
  rpc Search(SearchRequest) returns (SearchResponse);
}
```

