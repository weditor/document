
Apache Thrift
=================================
* 官网: http://thrift.apache.org/
* github: https://github.com/apache/thrift/

Thrift 思路和 ASN.1 更像，并不和某一种序列化方式绑死，
不过其内部仍然自带三种序列化实现: Json、简单二进制编码、紧凑二进制编码。

Thrift 将 schema 编译好后保存在程序中，网络传输只传输数据，不涉及 type。

Thrift 官方文档写的不怎么好。只找到下面两个链接介绍其内部的紧凑编码:

* wiki: https://cwiki.apache.org/confluence/display/THRIFT/New+compact+binary+protocol
* jira: https://issues.apache.org/jira/browse/THRIFT-110
* 紧凑编码格式: `compact-proto-spec-2.txt </_static/data/compact-proto-spec-2.txt>`_

通过上面的链接可以依稀看出来, 其紧凑编码格式从 Protobuf 借鉴了很多。

Protocol 实现代码位于 thrift github 仓库的 lib/cpp/src/thrift/protocol/ 目录

下面稍微介绍下二进制编码、紧凑二进制编码。

简单二进制编码
---------------------------

实现代码:  lib/cpp/src/thrift/protocol/TBinaryProtocol.tcc 

和想象中一样，这种方式和 C-style 拷贝内存的方式并没有什么不同。

以 int64 类型的整数为例。

.. image:: /_static/images/serialize/thrift-binary.jpg

可以看到, 直接取了数据内存的 8 byte 写进去了(int64 占8byte内存)。

紧凑二进制编码
----------------------------

实现代码:  lib/cpp/src/thrift/protocol/TCompactProtocol.tcc 

紧凑编码中对数值的编码借鉴了 Protobuf：

1. 如果数值类型是 double、float 等类型，先转换为 int 类, 
2. 对 int 类进行 zig-zag 编码。
3. 使用每个 byte 的首位标识内容是否结束，为 1 则后面还有内容，为 0 表示当前是最后的 byte 。

.. image:: /_static/images/serialize/thrift-compact-1.jpg

.. image:: /_static/images/serialize/thrift-compact-2.jpg


对于 string、binary ，采用 ``length data`` 的编码方式。

而 map、list、set 属于同一个类别(collection)，所以还有一个子类别来说明他们具体是那个类型。
编码方式是 ``subtype length data`` 。 collection 还有另外一个处理，如果 length <= 14，
则省略 1 字节，直接把 length 写到 subtype 里面, 这一点很像 MessagePack 。

.. image:: /_static/images/serialize/thrift-compact-collection.jpg

附 ZigZag 源码:

.. code-block:: cpp

    /**
    * Convert l into a zigzag long. This allows negative numbers to be
    * represented compactly as a varint.
    */
    template <class Transport_>
    uint64_t TCompactProtocolT<Transport_>::i64ToZigzag(const int64_t l) {
        return (static_cast<uint64_t>(l) << 1) ^ (l >> 63);
    }

    /**
    * Convert from zigzag long to long.
    */
    template <class Transport_>
    int64_t TCompactProtocolT<Transport_>::zigzagToI64(uint64_t n) {
        return (n >> 1) ^ static_cast<uint64_t>(-static_cast<int64_t>(n & 1));
    }
