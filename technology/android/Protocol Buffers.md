# Protocol Buffers

Google提供的一种跨语言、跨平台，具有良好兼容性且性能优异的序列化框架。

## Proto3

### proto编码



### .proto文件

使用 protobuf 首先需要定义 .proto 文件：

第一行指定语法版本，如果未指定，则默认为 **proto2** 的语法：

```protobuf
syntax = "proto3";
```



每个定义的字段需要指定一个唯一的数字，作为 proto 二进制数据编解码时的标识：



为了避免删除某个字段后导致新老版本不兼容，可以把需要删除的字段设置为保留字段，则其他人新增或者修改为这些保留字段时编译器就会报错：

```protobuf
message Foo {
  reserved 2, 15, 9 to 11; //如果有人定义了唯一标识为 2、15、9、10、11 的字段，则编译器报错
  reserved "foo", "bar"; //如果有人定义了名称为 "foo" 或 "bar" 的字段，则编译器报错
}
```

枚举定义：

```protobuf
enum EnumAllowingAlias {
    option allow_alias = true; //通过指定这个值，枚举可以设置两个字段一样的值来作为别名，即 STARTED 等价于 RUNNING
    UNKNOWN = 0;
    STARTED = 1;
    RUNNING = 1;
  }
```

import 其他的 .proto 文件

```protobuf
import public "new.proto"; //指定 public 则 import 可以传递
import "other.proto"; 
```

### 生成Java文件

foo_bar.proto -> FooBar.java，如果 .proto 文件有一个名为 FooBar 的 message，则会生成  FooBarOuterClass.java



## 官方文档

[Language Guide (proto3)](https://developers.google.com/protocol-buffers/docs/proto3)



