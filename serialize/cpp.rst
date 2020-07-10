
C/C++ 的序列化
==============================

先介绍 C 程序中的序列化, C 程序的序列化方式算是比较古老的方式。
虽然这类底层语言在很多方面都不如高级语言好用，
但是在序列化方面却出乎意料地简单。

得益于 C 程序可以直接操作内存， C 程序只需要将待序列化对象的内存内容拷贝出来就行了。

只不过这里面仍然有一些坑，比如当涉及到跨平台时，大小端序、字节对齐等带来的问题……

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

