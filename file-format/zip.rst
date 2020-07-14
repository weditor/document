###################################
Zip
###################################

.. image:: /_static/images/file/zip-format.svg

* zip 格式: https://pkware.cachefly.net/webdocs/casestudies/APPNOTE.TXT
* zip 格式(PDF版): http://www.idea2ic.com/File_Formats/ZIP%20File%20Format%20Specification.pdf

两个版本文档稍微有些区别, 下面的讲解以 PDF 版为准。

.. note:: 我也不知道目前流通的是哪个版本, 要翻源码确认一下。但是整体思路都是一样的。



zip 文档结构
==================================

基础结构::

    [local file header 1]
    [file data 1]
    [data descriptor 1]
    . 
    .
    .
    [local file header n]
    [file data n]
    [data descriptor n]

    [archive decryption header] 
    [archive extra data record] 
    [central directory]
    [zip64 end of central directory record]
    [zip64 end of central directory locator] 
    [end of central directory record]

从上面的结构可以明显看到， zip 压缩策略与 tar.* 明显不同，tar 将所有文件打包到一起进行压缩，
zip 保留了文件元信息, 只对内容进行压缩(即 [file data] 段), 
这个特性使zip在不进行解压的情况下就能得到文件信息。

文件数据
============================

每个文件包含三个数据段: local file header/file data/file descriptor 。

一个 zip 中可以包含多个文件，即以上三段重复多次。

其中 **local file header** 包含文件元信息，如文件名、压缩前大小、压缩后大小、校验码。
**file data** 包含压缩后的数据。 **file descriptor** 是 **local file header** 的补充，
因为 **local file header** 首先写入zip包，而有些信息无法在压缩前就得知，
如压缩后大小，就可以在 **file descriptor** 中补充, 多用于无法进行 seek 操作的存储介质。


local file header 结构::

    local file header signature     4 bytes  (0x04034b50)
    version needed to extract       2 bytes
    general purpose bit flag        2 bytes
    compression method              2 bytes
    last mod file time              2 bytes
    last mod file date              2 bytes
    crc-32                          4 bytes
    compressed size                 4 bytes  // 压缩后大小
    uncompressed size               4 bytes
    file name length                2 bytes
    extra field length              2 bytes
    file name (variable size)   
    extra field (variable size)

中央目录结构
=============================

另一个很重要的结构是 **central directory** (中央目录结构)。

central directory::

    [file header 1]
    .
    .
    .
    [file header n]
    [digital signature]

File header::

    central file header signature   4 bytes  (0x02014b50)
    version made by                 2 bytes
    version needed to extract       2 bytes
    general purpose bit flag        2 bytes
    compression method              2 bytes
    last mod file time              2 bytes
    last mod file date              2 bytes
    crc-32                          4 bytes
    compressed size                 4 bytes // 压缩后大小
    uncompressed size               4 bytes
    file name length                2 bytes
    extra field length              2 bytes
    file comment length             2 bytes
    disk number start               2 bytes
    internal file attributes        2 bytes
    external file attributes        4 bytes
    relative offset of local header 4 bytes // 文件数据偏移量
    file name (variable size)
    extra field (variable size)
    file comment (variable size)

Digital signature::

    header signature                4 bytes  (0x05054b50)
    size of data                    2 bytes
    signature data (variable size)

中央目录结构在zip中的设计有几个特点:

    * 此结构放在zip文件末尾。这使得zip在追加文件的时候比较方便，对整个 zip 包影响很小。
    * 存放了所有文件的信息。这种设计让 zip 具备了一定程度的随机读取能力，

所以, 对于大量小文件，如果使用zip压缩，使用时只需要将 zip 包尾部的少量信息加载到内存，
就能根据根据其中的信息找到需要的文件并解压出来。

.. warning:: zip 的随机读取仅限于理论，实际情况要看库的实现方式，
    比如 Python 读取zip，使用过程中发现仍然会把整个 zip 文件缓存到内存。

