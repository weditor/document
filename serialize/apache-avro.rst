
##########################################
Apache Avro
##########################################

* 官网: https://avro.apache.org
* 编码格式: https://avro.apache.org/docs/current/spec.html

Avro 类似 thrift, 也是 Apache 旗下的 RPC 框架，主要面向大数据场景。
schema定义非常灵活，属于半 schema-less。

Avro 内部支持两种序列化方式：

1. Json。用于开发环境或者其他要求数据可阅读的场合(如 Web )
2. 二进制。常用于生产环境

Schema
=======================

作为一个带 schema 的序列化框架，为了应对真实大数据场景中的数据多样性，Avro 对 Schema 做了几个改进：

1. schema 直接使用 json 定义好就可直接使用，且不需要像 thrift、Protobuf 一样编译。实现运行时定义 schema 。
2. 每次序列化数据时，头部都要带上 schema。即每条数据都是自描述的，实现类似 schema-less 的功能。
3. 传递数据时可以传递单条数据或者多条数据，传递多条数据可以提高 schema 的利用效率。
4. 对于 schema 本身比较大的情况，为了减少每条数据的体积，可以只传递 schema 的 8-bit 指纹。

schema 案例

.. code-block:: json

    {
        "type": "record",
        "name": "Test",
        "fields" : [
            {"name": "a", "enum": "long"},
            {"name": "b", "type": "string"},
            {"name": "c", "type": "enum", "symbols": ["A", "B", "C"]}
        ]
    }

数据
=======================

由于每条数据都会带有单独的schema, 数据部分就不用再维护 schema , 
只负责按照schema中fields定义的顺序存储数据就可以了。

如以下fields::

    [
        {"name": "a", "enum": "long"},
        {"name": "b", "type": "string"}
    ]

如果需要序列化的数据是 ``{"a": 27, "b": "foo"}`` , 存储的格式如下:

    0x36 0x06 f o o

    * 0x36: 27 进行 zig-zag 编码后的16进制。
    * 0x06: 十进制3，即 foo 字符串长度。也是 zig-zag 编码。

解码的时候，如果缺少 schema , 并不知道 0x36 应该如何解释, 有 schema 后，才知道需要解析成整数 27。
所以数据对 schema 的依赖性非常强，而且schema的独立存储让数据部分变得比 protobuf 还紧凑。

对于 enum ，Avro 根据 schema 中列举的枚举值对他们进行顺序编号，变成 int ，
如上面的枚举值 ``["A", "B", "C"]`` ,分别编码成 0/1/2 。

对于 float，由于 float 在内存中固定占用 4 字节，先把它内存中的 buffer 强制解释为 int32 , 然后使用 int 的编码方法处理。
