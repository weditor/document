
MessagePack
=================================

官网的介绍: It's like JSON. but fast and small.

.. image:: /_static/images/serialize/messagepack-intro.png

* 官网: https://msgpack.org/
* 编码格式: https://github.com/msgpack/msgpack/blob/master/spec.md

MessagePack 主要的编码范式也是 `type length data`。 但内部编码方式和 Protobuf 很不一样。

MessagePack 是一种紧凑的序列化格式, 支持流式序列化，非常适合RPC传输，
编解码速度、占用空间和 Protobuf 一个量级，其优势是 schema-less , 可以灵活地传输各种数据。

MessagePack 实现数据紧凑的方式和 Protobuf 不同，Protobuf 使用首位标识后面是否有数据，
而 MessagePack 通过类型系统决定整数有多长，如 uint8 存放小整数，uint16 类型存放大整数。

1 byte 类型
----------------------------------

有一些type本身就是一个值，这类值没有data，只需要一个 type 字段就能表示。

如 nil (type 是 0xc0)::

    +--------+
    |  0xc0  |
    +--------+

数值类型
-----------------------------

数值类型编码范式是 `type data`。

根据数值大小自动选择 int8/int16/int32/int64 ,从而节省空间。

如 uint 16(type 是 0xcd)::

    +--------+--------+--------+
    |  0xcd  |ZZZZZZZZ|ZZZZZZZZ|
    +--------+--------+--------+

这里就有必要介绍 MessagePack 另一个创新之处: type 内部也能存储数据。 对于太小的整数（0~127）, 
其值可以直接存储在 type 里面，放在后 7 位。

uint7::

    +--------+
    |0XXXXXXX|
    +--------+

可变类型
----------------------------

字符串一般范式是 `type length data`。和数值类型一样，对于超短的字符串，length/type 可以存储在一个 byte 里面。

长度小于 31 的字符串::

    +--------+========+
    |101XXXXX|  data  |
    +--------+========+

长度在 (2^8)-1 以内的字符串::

    +--------+--------+========+
    |  0xd9  |YYYYYYYY|  data  |
    +--------+--------+========+

以此类推……
