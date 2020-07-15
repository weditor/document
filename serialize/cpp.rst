
C/C++ 的序列化
==============================

C-Style 的列化是一种比较古老的序列化形式。
虽然这类底层语言在很多方面都不如高级语言好用，
却在在序列化方面却出乎意料地简单。

得益于 C 内存操作能力， 只需要将待序列化对象的内存内容拷贝出来就行了。

.. code-block:: cpp

    struct UserInfo 
    {
        char name[32];
        int age;
    };

    void writer(struct UserInfo *pInfo) 
    {
        FILE *fp = fopen("test.info", "wb");
        // 将 info 所处的内存内容直接写入文件
        fwrite(pInfo, 1, sizeof(struct UserInfo), fp);
        fclose(fp);
    }

    void reader(struct UserInfo *pInfo) 
    {
        FILE *fp = fopen("test.info", "rb");
        // 将文件内容恢复回来
        fread(pInfo, sizeof(struct UserInfo), 1, fp);
        fclose(fp);
    }


只不过涉及到跨平台时，会有一些坑……

而这些问题也是几乎所有序列化库需要解决的。

大小端
----------------------

.. image:: /_static/images/serialize/c-endian.svg

不同大小端的系统会将同样的数据序列化为不同的格式。

一般方法是转换为统一的网络序。

字节对齐
-----------------------

.. image:: /_static/images/serialize/c-align.svg

内存的字节对齐也是阻止 C-Style 序列化推广的原因之一 。

这个策略有助于加速数据在内存的寻址速度，但是对序列化毫无用处。

一般序列化库会直接去除对齐, 使数据紧凑化。
采用的方式有 type+定长、flag实现的变长(Varint)、length+data 等。
