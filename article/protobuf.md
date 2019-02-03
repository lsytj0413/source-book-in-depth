# protobuf #

## 简介 ##

[protobuf](https://developers.google.com/protocol-buffers/) 是由 Google 开源的一种语言中立, 平台无关, 可扩展的序列化结构化数据的方式, 用于通信协议, 数据存储等.

protobuf 是用于序列化结构化数据的灵活的, 高效的自动化机制. 在定义一次数据的结构后, 就可以使用工具生成各种语言的结构代码(用于写入数据流或从数据流中读取), 而且可以在不破坏旧的格式使用代码的情况下升级结构定义.

本文章只介绍 proto3 格式.

## 定义一个消息结构 ##

假设我们有如下的一个需求: 定义一个用于搜索的消息结构, 包含搜索关键字, 页码和每页的数量. 一个简单的 .proto 文件如下:

```
syntax = "proto3";

message SearchRequest {
    string query = 1;
    int32 page_number = 2;
    int32 result_per_page = 3;
}
```

- 第一行指示使用 proto3 的语法(编译器默认使用 proto2), 该语句必须是文件中的第一行有效代码(非空白或注释)
- SearchRequest 包含 3 个字段, 每个字段都包括一个字段名称和类型

### 字段类型 ###

proto3 支持多种标量数据类型, 也可以使用组合类型--包括枚举和自定义类型.

### 字段标签 ###

可以为每一个字段指定一个唯一的数字标签, 该标签用于在消息的二进制格式中定位字段, 在指定之后不应该进行修改. 对标签在 1-15 的字段, protobuf 使用一个自己来编码(包含标签和数据类型), 在 16-2047 的字段使用两个字节. 所以可以将 1-15 用于常见的字段.

最大的标签值为 2^29 - 1, 即 536870911. 不应该使用 19000-19999 范围(FieldDescriptor::kFirstReservedNumber, FieldDescriptor::kLastReservedNumber)内的值, 这个范围是预留给 protobuf 内部使用的.

### 特殊规则 ###

字段可以是一下一种类型:

- 单数: 0个或1个
- 重复的(repeated): 0个或多个(即数组)

在 proto3 中, repeated 的标量类型默认使用 packed 方式编码.

### 增加消息类型 ###

可以在一个 .proto 文件中定义多个消息类型.

```
syntax = "proto3";

message SearchRequest {
    string query = 1;
    int32 page_number = 2;
    int32 result_per_page = 3;
}

message SearchResponse {
    ....
}
```

### 注释 ###

在 .proto 文件中可以使用 C/C++ 风格的注释:

```
syntax = "proto3";

/* SearchRequest represents a search query, with pagination options to
 * indicate which results to include in the response. */

message SearchRequest {
  string query = 1;
  int32 page_number = 2;  // Which page number do we want?
  int32 result_per_page = 3;  // Number of results to return per page.
}
```

### 保留字段 ###

如果删除或注释掉某些字段, 那么该字段的标签可以被充用, 再重用后可能导致使用旧格式解析出现问题. 为了避免这种错误, 可以将字段名或者标签声明为保留字段:

```
message Foo {
    reserved 2, 15, 9 to 11;
    reserved "foo", "bar";
}
```

在同一行 reserved 声明中不能混合使用字段名和标签.

### 可以从 .proto 文件编译得到什么? ###

当使用 protobuf 编译器便宜一个 .proto 文件, 编译器会生成所选择的语言的对应代码文件:

- C++: 对每一个 .proto 文件生成一个 .h 和 .cc 文件
- Java: 对每一个消息类型生成一个 .java 文件, 并生成一个 Builder 类
- Python: 对每一个消息类型生成一个 module, 并通过 metaclass 生成对应的 class
- Go: 对每一个消息类型生成一个 .pb.go 文件
- Ruby: 生成一个 .rb 文件
- Objective-C: 对每一个 .proto 文件生成一个 pbobjc.h 和 pbobjc.m 文件
- C#: 对每一个 .proto 文件生产一个 .cs 文件

## 标量类型 ##

protobuf 支持的标量类型如下:

| .proto 类型 | 备注                                              | C++    | Java       | Python      | Go      | Ruby                 | C#         | PHP            |
| :--         | :--                                               | :--    | :--        | :--         | :--     | :--                  | :--        | :--            |
| double      |                                                   | double | double     | float       | float64 | Float                | double     | float          |
| float       |                                                   | float  | float      | float       | float32 | Float                | float      | float          |
| int32       | 使用可变长编码, 处理负数效率不高(负数使用 sint32) | int32  | int        | int         | int32   | Fixnum or Bignum     | int        | integer        |
| int64       | 使用可变长编码, 处理负数效率不高(负数使用 sint64) | int64  | long       | int/long    | int64   | Bignum               | long       | integer/string |
| uint32      | 使用可变长编码                                    | uint32 | int        | int/long    | uint32  | Fixnum or Bignum     | uint       | integer        |
| uint64      | 使用可变长编码                                    | uint64 | long       | int/long    | uint64  | Bignum               | ulong      | integer/string |
| sint32      | 使用可变长编码                                    | int32  | int        | int/long    | int32   | Fixnum or Bignum     | uint       | integer        |
| sint64      | 使用可变长编码                                    | int64  | long       | int/long    | int64   | Bignum               | long       | integer/string |
| fixed32     | 总是使用4字节, 如果值大于 2^28 比 uint32 效率更高 | uint32 | int        | int/long    | uint32  | Fixnum or Bignum     | uint       | integer        |
| fixed64     | 总是使用8字节, 如果值大于 2^56 比 uint64 效率更高 | uint64 | long       | int/long    | uint64  | Bignum               | ulong      | integer/string |
| sfixed32    | 总是使用4字节                                     | int32  | int        | int/long    | int32   | Fixnum or Bignum     | int        | integer        |
| sfixed64    | 总是使用8字节                                     | int64  | long       | int/long    | int64   | Bignum               | long       | integer/string |
| bool        |                                                   | bool   | boolean    | bool        | bool    | TrueClass/FalseClass | bool       | boolean        |
| string      | 必须是 utf8 或 7-bit ASCII 编码                   | string | String     | str/unicode | string  | String(UTF-8)        | string     | string         |
| bytes       | 可包含任意字节序列                                | string | ByteString | str         | []byte  | String(ASCII-8BIT)   | ByteString | string         |

## 默认值 ##

如果没有对字段赋值, 那么该字段的值即使用默认值:

- 如果是字符串, 默认值是空字符串
- 如果是 bytes, 默认值是空 bytes
- 如果是 bool, 默认值是 false
- 如果是数字类型, 默认值是 0
- 如果是枚举类型, 默认值是枚举中的第一个值, 且该值必须是 0
- 如果是消息类型, 则没有默认值, 具体取值与编程语言相关
- 如果是 repeated 字段, 那么默认值为空(数组长度为0)

需要注意的是, 对于标量类型不能区分该字段是被显示设置为默认值或是没有设置因而采用的默认值.

## 枚举类型 ##

定义一个简单的枚举类型的语法如下:

```
syntax = "proto3";

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

枚举中的第一个值必须为 0, 这是因为:

1. 0 是默认值
2. 兼容 proto2 中第一个值是默认值的语义

如果你需要将不同的定义设置为相同的值, 需要使用 option 语法:

```
enum EnumAllowingAlias {
    option allow_alias = true;
    UNKNOWN = 0;
    STARTED = 1;
    RUNNING = 1;
}
```

枚举的值是 32 位的整数, 使用负数的效率较低(不推荐). 在反序列化的时候, 未知的枚举值会保留, 具体保留方式视不同的语言而定.

### 保留值 ###

在 enum 中也可以设置保留值, 以避免移除值之后导致错误:

```
enum Foo {
    reserved 2, 15, 9 to 11, 40 to max;
    reserved "FOO", "BAR";
}
```

## 使用消息类型 ##

可以将字段定义为其他的消息类型:

```
syntax = "proto3";

message SearchResponse {
    repeated Result results = 1;
}

message Result {
    string url = 1;
    string title = 2;
    repeated string snippets = 3;
}
```

### 导入定义 ###

可以在 proto 文件中导入其他 proto 文件定义的消息类型:

```
import "myproject/other_protos.proto"
```

默认情况下 import 只在当前文件生效, 可以使用 import public 将类型导入到引入当前文件的文件:

```
// new.proto
// All definitions are moved here

// old.proto
// This is the proto that all clients are importing.
import public "new.proto";
import "other.proto";

// client.proto
import "old.proto";
```

protobuf 编译器会在 -I/--proto_path 指定的目录中搜索 import 文件, 如果没有设置则从当前目录中进行查找.

### 使用 proto2 的消息类型 ###

可以在 proto3 中导入 proto2 语法的文件, 但是 proto2 的 enum 语法不能在 proto3 中工作.

## 嵌套类型 ##

可以在一个类型中定义新的类型:

```
message SearchResponse {
  message Result {
    string url = 1;
    string title = 2;
    repeated string snippets = 3;
  }
  repeated Result results = 1;
}
```

在其他地方可以使用 SearchResponse.Result 引用嵌套的类型. 嵌套可以是任意层次的.

## 更新类型定义 ##

在更新时最好遵守以下规则:

1. 不要修改已有字段的标签值
2. 新增字段时需要关注默认值
3. 被移除的字段的标签不应该重复使用
4. int32, uin32, int64, uint64 和 bool 是兼容的(可以修改类型)
5. sint32 和 sint64 是互相兼容的, 但是不兼容其他的数字类型
6. string 和 bytes 是兼容的(当 bytes 是 utf8 时)
7. 嵌入的消息和 bytes 是兼容的, 当 bytes 包含编码版本时
8. fixed32 和 sfixed32 是兼容的, fixed64 和 sfixed64 是兼容的
9. enum 和 int32, uin32, int64 以及 uint64 是兼容的
10. 将一个字段转到 oneof 中是兼容的, 将多个字段转到 oneof 可能是兼容的(当不会同时设置多个时), 将任意字段转到已经存在的 oneof 字段中是不安全的

## 未知字段 ##

protobuf 可以解析含有未知字段的数据.

## Any ##

```
import "google/protobuf/any.proto"

message ErrorStatus {
    string message = 1;
    repeated google.protobuf.Any details = 2;
}
```

## Oneof ##

当有多个字段不会被同时设置时, 可以使用 Oneof 来提升性能.

### 如何使用 ###

```
message SampleMessage {
    oneof test_oneof {
        string name = 4;
        SubMessage sub_message = 9;
    }
}
```

在 oneof 中不能使用 repeated 字段.

### 特性 ###

- 当设置 oneof 字段的值时会清空之前的值
- 解析 oneof 字段时只会保存最后的值
- 不能包含 repeated 字段
- 反射 api 可以在 oneof 字段上工作
- 如果你使用的是 C++, 确保不会导致 memory crash
- 在 C++ 中, 交换两个 oneof 字段会交换设置的值

### 向后兼容性 ###

向 oneof 中增加或减少字段需要小心. 这可能导致 None/NOT_SET 不能探测到是否未设置/或未包含该字段定义.

### 标签复用 ###

- 向 oneof 添加或删除字段可能丢失一些字段信息
- 删除字段后重新添加, 可能导致解析之前序列化的数据丢失字段信息
- 拆分或组合 oneof, 也可能导致丢失字段信息

## 字典 ##

protobuf 包含了定义字典的语法:

```
map<key_type, value_type> map_field = N;
map<string, Project> projects = 3;
```

- key_type 可以是数字和string类型, 不能是枚举
- value_type 可以是任意类型, 除了 map
- map 字段不能是 repeated
- map 的字段顺序是未定义的
- 将生成 text 信息时 map 的 key 是有序的
- 当反序列化或合并 map 时重复的 key 取最后一个值; 当从 text 数据中反序列化时如果有相同的 key 值会失败

### 向后兼容性 ###

map 类型的等价定义如下:

```
message MapFieldEntry {
    key_type key = 1;
    value_type value = 2;
}

repeated MapFieldEntry map_field = N;
```

## 包 ##

可以在 .proto 文件中增加可选的 package 指示符来避免在不同 .proto 文件中定义相同名字的消息类型引起的冲突:

```
pacakge foo.bar;
message Open{}
```

```
message Foo {
    foo.bar.Open open = 1;
}
```

使用 package 之后生成的代码与使用的编程语言相关:

- C++: 使用命名空间来实现 package
- Java: 使用 Java pacakge, 除非提供了 option java_package 指示符
- Python: 忽略
- Ruby: 使用 Ruby 命名空间
- C#: 使用命名空间, 除非提供了 option\_csharp\_namespace 指示符

### 包和名字解析 ###

protobuf 中的名字解析的和 C++ 中一样.

## 定义服务 ##

如果你需要在 RPC 框架中使用 protobuf 定义的消息类型, 可以在 .proto 文件中定义 RPC 服务, 然后使用 protobuf 编译器来生成对应的代码:

```
service SearchService {
   rpc Search(SearchRequest) returns (SearchResponse);
}
```

## JSON 映射 ##

proto3 支持 JSON 编码规范. 如果一个值在 JSON 中不存在或者为 null, 它会被序列化为该类型的默认值; 如果该值是默认值, 序列化时默认的情况下会被忽略(以节省空间).

| proto3                 | JSON          | JSON example               | 备注                                                        |
| :--                    | :--           | :--                        | :--                                                         |
| message                | object        | {"foobar": v}              | 默认 key 使用小驼峰命名, 可以使用 json_name 选项指定名称    |
| enum                   | string        | "FOO_BAR"                  |                                                             |
| map<K,V>               | object        | {"k": v}                   | 所有的key都会转换为字符串                                   |
| repeated V             | array         | [v...]                     | null 转换为 []                                              |
| bool                   | true, false   | true, false                |                                                             |
| string                 | string        |                            |                                                             |
| bytes                  | base64 string |                            |                                                             |
| int32, fixed32, uint32 | number        |                            |                                                             |
| int64, fixed64, uint64 | string        |                            |                                                             |
| float, double          | number        |                            |                                                             |
| Any                    | object        | {"@type": "url", "f": v}   | 如果 Any 是一个 JSON, 会转换为 {"@type": xxx, "value": yyy} |
| Timestamp              | string        | "1972-01-01T10:00:20.021Z" | RFC3339                                                     |
| Duration               | string        | "1s"                       |                                                             |
| Struct                 | object        | {...}                      | 见 struct.proto                                             |
| Wrapper types          | various types | 2, "2"                     |                                                             |
| FieldMask              | string        |                            | 见 fieldmask.proto                                          |
| ListValue              | array         |                            |                                                             |
| Value                  | value         |                            |                                                             |
| NullValue              | null          |                            |                                                             |

### JSON选项 ###

- 值未默认值的key被忽略
- 忽略未知字段
- 可选的字段名
- 枚举中使用整形值

## 可选配置 ##

所有的配置项可以子啊 google/protobuf/descriptor.proto 查看.

- java\_package: 指定生成的代码的 java package 名称.

```
option java_package = "com.example.foo";
```

- java\_multiple\_files: 指定顶层的消息类型, 枚举和服务定义在 package层, 否则默认定义在以 proto 文件名命名的 class 中.

```
option java\_multiple\_files = true;
```

- java\_outer\_classname: 指定外层的 class 名称.

```
option java\_outer\_classname = "Ponycopter";
```

- optimize\_for: 可以设置为 SPEED, CODE\_SIZE, LITE\_RUNTIME, 影响生成的 c++ 和 java 代码:
    - SPEED: 默认值, 序列化和反序列化速度优先
    - CODE\_SIZE: 最小代码量
    - LITE\_RUNTIME: 使用 libprotobuf-lite 代替 libprotobuf, 缺少某些特性, 常用于手机等平台, 也生成速度优先的代码, 所有消息实现 MessageLite 接口而非 Message 接口

- cc\_enable\_arenas: 对 c++ 使用 arenas 分配器
- objc\_class\_prefix: 设置 oc 的类前缀
- deprecated: 指示消息类型被废弃

```
int32 old_field = 6 [deprecated=true];
```

### 自定义选项 ###

高级特性, 使用扩展实现.

## 生成代码 ##

使用 protoc 编译器生成代码的命令如下:

```
protoc --proto_path=IMPORT_PATH --cpp_out=DST_DIR --java_out=DST_DIR --python_out=DST_DIR --go_out=DST_DIR --ruby_out=DST_DIR --objc_out=DST_DIR --csharp_out=DST_DIR path/to/file.proto
```

- IMPORT\_PATH: 指定包含目录, import 文件时会从该目录进行查找, 默认为当前目录. 可以缩写为 -I=IMPORT_PATH
- xxx\_out: 指定代码生成目录. 如果以 .zip 或 .jar 结尾则打包到对应的文件.

## 风格指南 ##

### 消息和字段名称 ###

为消息名称使用驼峰命名, 字段名称使用下划线命名:

```
message SongServerRequest {
    string song_name = 1;
}
```

### 枚举 ###

为枚举名称使用驼峰命名, 具体字段使用大写的下划线命名:

```
enum Foo {
    FIRST_VALUE = 0;
    SECOND_VALUE = 1;
}
```

### 服务 ###

对服务名称和方法名称使用驼峰命名:

```
service FooService {
    rpc GetSomething(FooRequest) returns (FooResponse);
}
```

## 编码 ##

本节定义二进制格式的 protobuf 序列的编码方式.

### 一个简单的消息 ###

假设有一个消息类型如下:

```
message Test1 {
    int32 a  = 1;
}
```

假设你在代码中将 a 的值设置为 150, 然后序列化得到的字节如下:

```
08 96 01
```

### Base 128 Varints ###

varints 是使用一个或多个字节序列化整形的方法. 在 varints 的每个字节(除了最后一个字节), 都设置有 msb 标志位: 当还有后续字节时设置为 1. 假设有一个数字1, 表示如下:

```
# 只有一个字节, 不设置 msb
0000 0001
```

300 的表示如下:

```
1010 1100 0000 0010
# 如何转换为300
-> 010 1100 000 0010    # 首先去除 msb
-> 000 0010 010 1100    # 然后以 7bit 为单位进行反转
-> 100101100            # 然后连接起来求值
-> 256 + 32 + 8 + 4 = 300
```

### 结构体 ###

在 protobuf 中, 每一个消息中的 key 包含两个值: 字段标签数字和类型(包含足够的信息来找到值的长度). 可用的类型如下:

| Type | Meaning          | Used For                                                 |
|  :-- | :--              | :--                                                      |
|    0 | Varint           | int32, int64, uint32, uint64, sint32, sint64, bool, enum |
|    1 | 64-bit           | fixed64, sfixed64, double                                |
|    2 | Length-delimited | string, bytes, embedded messages, packed repeated fields |
|    3 | Start group      | groups(废弃)                                             |
|    4 | End group        | groups(废弃)                                             |
|    5 | 32-bit           | fixed32, sfixed64, float                                 |

每一个 key 的数据都是一个 varint, 值是 (field\_number << 3) | wire\_type. 假设有如下的一个 key:

```
# 去掉了 msb
000 1000
```

取后三位可以得到类型(0), 然后右移三位得到字段签名(1). 从类型可知接下来是一个 varint.

### 更多值类型 ###

#### 有符号整数 ####

对于 sint32 和 sint64 类型, 它们的 wire\_type 也是 0, 但是编码方式和 int32 与 int64 有些不同. 因为它们负数的符号位为1, 会导致即使 -1 这样的数也会占用更多的字节. protobuf 使用 ZigZag 编码方式来处理有符号整数.

对 32 位数, 编码方式如下:

```
(n << 1) ^ (n >> 31)
```

对 64 位数, 编码方式如下:

```
(n << 1) ^ (n >> 63)
```

即将符号位转换到最低位.

#### 非 varint 数值 ####

对非 varint 数值的处理很简单. double 和 fixed64 的 wire\_type 是 1, 使用固定的 8 字节; float 和 fixed32 的 wire\_type 是 5, 使用固定的 4 字节. 并且它们都使用小端序(Little-endian)存储.

#### 字符串 ####

字符串的 wire\_type 是 2, 表示编码为一个长度加上值的字节表示.

```
message Test2 {
  string b = 2;
}
```

对于如上的消息, 如果将 b 设置为 "tesing", 得到的字节如下:

```
12 07 74 65 73 74 69 6e 67

- 12: tag=2, type=2
- 07: length
- 剩下的: testing 的 utf-8 编码
```

#### 内嵌的消息 ####

假设有一个消息定义如下:

```
message Test3 {
  Test1 c = 3;
}
```

如果将 c.a 设置为 150, 得到的字节如下:

```
1a 03 08 96 01

- 1a: tag=3, type=2
- 03: length
- 剩下的: 即 Test1.a = 150 时的编码
```

#### 可选和重复元素 ####

在 proto3 中, repeated 元素使用 packed 编码.

在 proto3 中, 对于非 repeated 的字段, 被编码的消息中可能不存在该字段. 正常情况下, 对于非 repeated 的字段编码的结果中只应该含有该字段一次, 但是反序列化时可以处理含有多次的情况: 对于数值类型和字符串类型, 取最后一次出现的值; 对于内嵌的消息类型, 会合并该消息的同一字段(对标量类型去最后一次, 对 repeated 字段连接起来). 这种方式和解析两个消息然后 merge 他们的表现一样:

```
MyMessage message;
message.ParseFromString(str1 + str2)

# 等价与
MyMessage message1, message2;
message1.ParseFromString(str1);
message2.ParseFromString(str2);
message1.MergeFrom(message2);
```

#### packed repeated 字段 ####

repeated 字段被打包到一个键值对中, 其中 wire\_type 为 2, 每个元素和平时一样编码:

```
message Test4 {
  repeated int32 d = 4;
}
```

假设将 d 的值设置为 3, 270, 86942, 得到的字节如下:

```
22 06 03 8E 02 9E A7 05

- 22: tag=4, type=2
- 06: length
- 03: 第一个值, 3
- 8E 02: 第二个值, 270
- 9E A7 05: 第三个值, 86942
```

只有原生字段类型的重复字段才能定义为 packed.

### 字段顺序 ###

当消息被序列化时, 已知的字段会按照标签顺序序列化. protobuf 解析器必须能够解析任何顺序的值. 如果有未知的字段, java 和 c++ 会在已知字段全部写入后按照任意顺序写入未知字段, 而 python 会忽略它们.

## 技巧 ##

本节描述一些在 protobuf 中广泛使用的设计模式.

### 流化多个消息 ###

如果想要写入多个消息到一个文件或流中, 需要自己保持消息的开始和结束记录. protobuf 的格式不是自描述的.

### 大数据集 ###

protobuf 不善于处理大型的消息: 通常应该小于 1M.

### 自描述消息 ###

一个 protobuf 的序列化结果不包含该类型的信息: 即不知道是由哪个消息类型序列化而来. 可以创建一个包裹消息, 用来指明该序列化结果的类型:

```
message SelfDescribingMessage {
  FileDescriptorSet proto_files = 1;
  string type_name = 2;
  bytes message_data = 3;
}
```

文件 src/google/protobuf/descriptor.proto 定义了消息类型的描述, protoc 可以通过 --descriptor\_set\_out 选项输出一个 FileDescriptorSet.

通过使用类似 DynamicMessage 的类(C++ 和 Java 可用), 可以操作 SelfDescribingMessage.
