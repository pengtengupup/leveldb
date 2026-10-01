leveldb 文件格式
===================

> 本文为 [doc/table_format.md](../table_format.md) 的中文译本。

    <beginning_of_file>
    [data block 1]
    [data block 2]
    ...
    [data block N]
    [meta block 1]
    ...
    [meta block K]
    [metaindex block]
    [index block]
    [Footer]        (固定大小；始于 file_size - sizeof(Footer))
    <end_of_file>

文件包含内部指针。每个这样的指针称为 BlockHandle，包含如下信息：

    offset:   varint64
    size:     varint64

关于 varint64 格式的说明见 [varints](https://developers.google.com/protocol-buffers/docs/encoding#varints)。

1.  文件中的键/值对序列按有序方式存储，并划分成一串数据块。这些块在文件开头依次排列。每个数据块按 `block_builder.cc` 中的代码格式化，然后可选地压缩。

2. 数据块之后我们存储若干元数据块。支持的元数据块类型见下文。将来可能增加更多元数据块类型。每个元数据块同样使用 `block_builder.cc` 格式化，然后可选地压缩。

3. 一个 "metaindex" 块。它为每个其他元数据块包含一个条目，键为元数据块的名称，值为指向该元数据块的 BlockHandle。

4. 一个 "index" 块。该块为每个数据块包含一个条目，键是一个字符串，满足：>= 该数据块中的最后一个键，且在后继数据块的第一个键之前。值为该数据块的 BlockHandle。

5. 文件最末尾是固定长度的 footer，包含 metaindex 与 index 块的 BlockHandle 以及一个魔数。

        metaindex_handle: char[p];     // metaindex 的 Block handle
        index_handle:     char[q];     // index 的 Block handle
        padding:          char[40-p-q];// 填零字节以构成固定长度
                                       // (40==2*BlockHandle::kMaxEncodedLength)
        magic:            fixed64;     // == 0xdb4775248b80fb57（小端序）

## "filter" 元数据块

若打开数据库时指定了 `FilterPolicy`，则每个 table 中会存储一个 filter 块。"metaindex" 块包含一个条目，将 `filter.<N>` 映射到 filter 块的 BlockHandle，其中 `<N>` 是过滤策略 `Name()` 方法返回的字符串。

filter 块存储一串过滤器，其中 filter i 包含对如下范围内文件偏移所对应块中全部键调用 `FilterPolicy::CreateFilter()` 的输出：

    [ i*base ... (i+1)*base-1 ]

当前 "base" 为 2KB。例如，若块 X 和 Y 起始于范围 `[ 0KB .. 2KB-1 ]`，则 X 与 Y 中的所有键会通过调用 `FilterPolicy::CreateFilter()` 转换为一个过滤器，结果作为 filter 块中的第一个过滤器存储。

filter 块格式如下：

    [filter 0]
    [filter 1]
    [filter 2]
    ...
    [filter N-1]

    [offset of filter 0]                  : 4 bytes
    [offset of filter 1]                  : 4 bytes
    [offset of filter 2]                  : 4 bytes
    ...
    [offset of filter N-1]                : 4 bytes

    [offset of beginning of offset array] : 4 bytes
    lg(base)                              : 1 byte

filter 块末尾的偏移数组允许从数据块偏移高效映射到对应的过滤器。

## "stats" 元数据块

该元数据块包含一组统计信息。键是统计项名称，值包含统计数据。

TODO(postrelease)：记录如下统计项。

    data size
    index size
    key size (uncompressed)
    value size (uncompressed)
    number of entries
    number of data blocks
