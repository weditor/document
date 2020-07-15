
#############################
Protobuf
#############################

.. image:: /_static/images/serialize/protobuf-example.svg


简介
=================================

Protobuf 是 google 推出的一种序列化格式, 应用非常广泛，如 grpc、tfrecord。

* 内部编码: https://developers.google.com/protocol-buffers/docs/encoding
* 使用文档: https://developers.google.com/protocol-buffers/docs/proto3
* 支持的数据格式: https://developers.google.com/protocol-buffers/docs/reference/google.protobuf

proto schema 案例
=================================

:: 

    syntax = "proto3";

    message SearchRequest {
        string query = 1;
        int32 page_number = 2;
        int32 result_per_page = 3;
    }

字段后面的数字是 **Field Number** , 用于编码索引，其范围为 1~(2^29 - 1)。
推荐将 1~15 范围的数值预留用于频繁访问的属性, 这个范围的 *Field Number* 进行 Varint 
编码后只需要 1 字节, 16~2047 耗费 2 字节，越大的 *Field Number* 编码后字节越多。

probobuf 编码
=================================

+------+----------------------------+----------------------------------------------------------+
| 类型 |            含义            |                         适用类型                         |
+======+============================+==========================================================+
| 0    | Varint(变长数值)           | int32, int64, uint32, uint64, sint32, sint64, bool, enum |
+------+----------------------------+----------------------------------------------------------+
| 1    | 64-bit(64位数值)           | fixed64, sfixed64, double                                |
+------+----------------------------+----------------------------------------------------------+
| 2    | Length-delimited(指定长度) | string, bytes, embedded messages, packed repeated fields |
+------+----------------------------+----------------------------------------------------------+
| 3    | Start group                | groups (deprecated)                                      |
+------+----------------------------+----------------------------------------------------------+
| 4    | End group                  | groups (deprecated)                                      |
+------+----------------------------+----------------------------------------------------------+
| 5    | 32-bit(32位数值)           | fixed32, sfixed32, float                                 |
+------+----------------------------+----------------------------------------------------------+

每一个字段首字节表示类型, byte 末3位是主类型, 前5位是 *Field Number*, 
其中首位 msb (most significant bit) 有特殊用途。

.. image:: /_static/images/serialize/type-spec.svg

上图表示主类型为0(Varint)， Field Number 为 2 。

变长类型

    这里的变长类型特指变长数值 - Varint 。 Varint 数值区域中每个 byte 首位标识后面还有没有内容(此标识即 msb)。
    当首位为1，表示后面还有内容，为 0 表示当前已经是最后一字节。 

    这种变长策略是 Protobuf 的亮点, 使整数类型的存储非常紧凑，小整数高位为0，这些无效位可以省略，
    所以通常只会占 1 byte，大整数才会占据更多空间 。基于统计，越小的整数使用越频繁，这种策略是合理的。

    int32 类型的 -1 二进制是 ``0xffffffff`` ，由于补码的特性，绝对值很小的负数有效位非常多。
    也许你已经猜到了，对于这个小整数，使用 Varint 编码却需要占据 5 字节。
    所以，对于有符号整数， Protobuf 先采用 ZigZag 映射为无符号整数，再进行编码。

定长类型

    当 type 字段是定常类型, 只需要从后面取固定长度字节解释为对应类型，
    1、5 两种主类型都是定常类型。

指定长度

    字符串、结构体等类型长度没有上限，都需要手动指定长度，所以除了首字节type，还有第二段内容说明数据长度，
    再后面才是真正的数据。

Optional & Repeated

    由于每个字段都有 *Field Number* 作为索引, Optional 字段可以不出现在编码中，即缺失。
    
    Repeated 字段比较有意思。 Protobuf 存储的策略是相同的 *Field Number* 可以出现多次, 
    并且可以不连续， 如下是包含了2个元素的数组(Field Number索引为3); 

    .. image:: /_static/images/serialize/protobuf-repeated.svg


schema
=============================

Protobuf 编码有 type 的概念，但是作用很弱, 
数据解释很依赖于 schema , 甚至基于 schema 可以对相同的数据做出不同解释。

比如 protobuf 支持的 option、默认值、数组、结构体等概念，
都是在 .proto 文件中描述的，还有涉及到向后兼容性等功能， schema 层承载了大量业务逻辑。



如以下 schema (v1)::

    message TestMessage {
        string query = 1;
    }

此时新增一个字段 (v2)::

    message TestMessage {
        string query = 1;
        optional string page_number = 2 [default = 10];
    }

这种情况，如果 v2 客户端接收到 v1 协议数据, schema 层可以把这个字段补全。
v1 客户端接收到 v2 协议的消息，也能正确忽略这个字段.


总结
=================================

从各方面讲 Protobuf 比 json 更像是一种合格并且可靠的序列化格式。
从上面的描述可以看出，Protobuf 有一定的编解码的性能损耗，但是消耗不大。

同时 protobuf 对编程非常友好，强类型、跨平台跨语言、周边完善。
属于序列化方面的大一统框架，各方面比较平衡。

另一种古老而广泛应用的序列化格式是 `ASN.1 <https://zh.wikipedia.org/wiki/ASN.1>`_ 。
protobuf 借鉴了它的协议描述层，并且定义了数据编码格式， 而 ASN.1 并没有限定数据编码格式，
底层实现可以选择 json、xml 等任意编码格式，甚至是私有编码格式，所以不做过多讨论。
