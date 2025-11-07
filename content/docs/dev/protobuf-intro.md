---
title: "什么是 Protocol Buffers"
weight: 1
date: 2025-11-06
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# 什么是 Protocol Buffers

Protocol Buffers 是 Google 开发的数据序列化格式.
可以将结构化数据编码为紧凑的二进制进行传输.

## 为什么需要 Protocol Buffers

例如, 我们有一个类 `User`:

```go
type User struct {
    id   uint
    name string
}
```

有一个对象 `foo`
```go
foo := User{1, "bar"}
```

现在有一个 RPC 方法, 接受一个 `User` 作为参数.
我们想把 `foo` 传过去,
那么我们就需要想个办法把这个对象序列化.

一种简单的方法是序列化为 JSON:
```json
{
    "id": 1,
    "name": "bar"
}
```

但是代价是这样做空间占用很高.
这样一个消息至少需要 21 个字节来传输.

而 Protobuf 提供了一种二进制的方法来传输, 例如上面的消息用 Protobuf 来序列化得到的是一串二进制数:
`08 01 12 03 62 61 72`. 相较之下只有 7 个字节.

> [!IMPORTANT]
> 需要注意的是 protobuf **不保存** `"id"`, `"name"` 这些字段名

## 原理

Protocol Buffers 通过 `.proto` 文件对需要序列化的数据进行定义.
例如, 上面的 User 可以表示为 `user.proto`:
```proto
syntax = "proto3";

message User {
  int32 id = 1;
  string name = 2;
}
```

protobuf 内部使用 wire format 编码.
格式如下:
```
        |<-3 bit->|
[field] [wire_type] [value]
|<----- tag ----->|
```

其中每个字段的格式如下:
- `tag`: 标记部分, 格式是 `uint32 varint`, 由 `field` 与 `wire_type` 两个部分组成. 低三位为 `wire_type`.
  - `field`: 也就是 proto 文件里面的编号. 例如 User 的 id 的 `field` 就是 1
  - `wire_type`: 表示该字段的编码方式, 有以下的取值:
    - 0: `VARINT`, 对应 int32 / int64 / bool / enum
    - 1: `I64`, 对应 fixed64 / double
    - 2: `LEN`, 对应 string, bytes, message 等不定长的数据
    - 5: `I32`, 对应 fixed32 / float
- `value`: 具体的数据
  - 如果 `wire_type` 是 `LEN` 的话, 还需要额外加一个用 `varint` 表示的 `len-prefix` 来指示数据部分的长度.

其中 `tag` 是 `uint32` 格式, 这就导致了 field 不能超过 $2^{29}-1$

> [!INFO]
> `varint` 是 protobuf 的一种特殊编码格式.
> 通过在最高有效位放置 0 或 1 来说明是否还有后续数据.
> 如果最高有效位是 1 则说明还有后续数据.
> 
> 例如 150 转换为二进制是 `10010110`, 用 varint 来表示就是
> `10010110 00000001`

官方提供的参考如下:
```go
message    := (tag value)*

tag        := (field << 3) bit-or wire_type;
                encoded as uint32 varint
value      := varint      for wire_type == VARINT,
              i32         for wire_type == I32,
              i64         for wire_type == I64,
              len-prefix  for wire_type == LEN

varint     := int32 | int64 | uint32 | uint64 | bool | enum | sint32 | sint64;
                encoded as varints (sintN are ZigZag-encoded first)
i32        := sfixed32 | fixed32 | float;
                encoded as 4-byte little-endian (float is IEEE 754
                single-precision); memcpy of the equivalent C types (u?int32_t,
                float)
i64        := sfixed64 | fixed64 | double;
                encoded as 8-byte little-endian (double is IEEE 754
                double-precision); memcpy of the equivalent C types (u?int64_t,
                double)

len-prefix := size (message | string | bytes | packed);
                size encoded as int32 varint
string     := valid UTF-8 string (e.g. ASCII);
                max 2GB of bytes
bytes      := any sequence of 8-bit bytes;
                max 2GB of bytes
packed     := varint* | i32* | i64*,
                consecutive values of the type specified in `.proto`
```


例如, `"id": 1` 会被编码为 `08 01`, 转换为二进制就是:
```
00001 000 00000001
  |    |      |
  |    |      +--- value    : 1
  |    +---------- wire_type: 0 (VARINT)
  +--------------- field    : 1
```

而 `"name": "bar"` 会被编码为 `12 03 62 61 72`, 转换为二进制就是:
```
00010 010 00000011 01100010 01100001 01110010
  |    |     |         |        |        |
  |    |     |         |        |        +- ASCII    : r
  |    |     |         |        +---------- ASCII    : a
  |    |     |         +------------------- ASCII    : b
  |    |     +----------------------------- len      : 3
  |    +----------------------------------- wire_type: 2 (LEN)
  +---------------------------------------- field    : 2 
```

## 在 Go 中使用 Protocol Buffers

在 Go 中使用 protobuf 需要先定义 .proto 文件.

```proto
syntax = "proto3";

option go_package = "internal/gen/proto";

message User {
  int32 id = 1;
  string name = 2;
}
```

> [!INFO]
> 这里可以在 proto 中指定
> ```proto
> option go_package = "example.com/project/protos/fizz";
> ```
>
> 也可以调用 `protoc` 时指定:
> ```bash
> protoc --proto_path=src \
>   --go_opt=Mprotos/buzz.proto=example.com/project/protos/fizz \
>   protos/buzz.proto
> ```

之后使用 protoc 将其编译为 stub:
```
protoc --go_out=. proto/user.proto
```

会在对应路径生成 go stub 文件.
之后就可以调用.

例如这段代码
```go
u := &pb.User{Id: 1, Name: "bar"}
out, err := proto.Marshal(u)
if err != nil {
    log.Fatalln(err)
}
fmt.Println(out)
```
对应的输出就是 `[8 1 18 3 98 97 114]`, 转为 16 进制就是 `08 01 12 03 62 61 72`.

## 参考资料

- Encoding | Protocol Buffers Documentation: https://protobuf.dev/programming-guides/encoding/
