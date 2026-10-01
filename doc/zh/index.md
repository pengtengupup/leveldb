leveldb
=======

_Jeff Dean, Sanjay Ghemawat_

> 本文为 [doc/index.md](../index.md) 的中文译本。

leveldb 库提供持久化的键值存储。键和值均为任意字节数组。键值存储中的键按照用户指定的比较函数排序。

## 打开数据库

一个 leveldb 数据库的名称对应文件系统中的一个目录。数据库的全部内容都存放在该目录中。下面的示例展示如何打开数据库（若不存在则创建）：

```c++
#include <cassert>
#include "leveldb/db.h"

leveldb::DB* db;
leveldb::Options options;
options.create_if_missing = true;
leveldb::Status status = leveldb::DB::Open(options, "/tmp/testdb", &db);
assert(status.ok());
...
```

若希望在数据库已存在时报错，可在调用 `leveldb::DB::Open` 之前加入：

```c++
options.error_if_exists = true;
```

## Status

上文中出现了 `leveldb::Status` 类型。leveldb 中大多数可能出错的函数都会返回该类型。你可以检查结果是否成功，并打印相关错误信息：

```c++
leveldb::Status s = ...;
if (!s.ok()) cerr << s.ToString() << endl;
```

## 关闭数据库

使用完毕后，直接删除数据库对象即可。例如：

```c++
... 按上文所述打开 db ...
... 对 db 做一些操作 ...
delete db;
```

## 读与写

数据库提供 Put、Delete 和 Get 方法来修改/查询数据。例如，下面的代码将 key1 下的值移动到 key2：

```c++
std::string value;
leveldb::Status s = db->Get(leveldb::ReadOptions(), key1, &value);
if (s.ok()) s = db->Put(leveldb::WriteOptions(), key2, value);
if (s.ok()) s = db->Delete(leveldb::WriteOptions(), key1);
```

## 原子更新

注意：若进程在 Put key2 之后、Delete key1 之前崩溃，同一个值可能同时残留在多个键下。使用 `WriteBatch` 类原子地应用一组更新可以避免此类问题：

```c++
#include "leveldb/write_batch.h"
...
std::string value;
leveldb::Status s = db->Get(leveldb::ReadOptions(), key1, &value);
if (s.ok()) {
  leveldb::WriteBatch batch;
  batch.Delete(key1);
  batch.Put(key2, value);
  s = db->Write(leveldb::WriteOptions(), &batch);
}
```

`WriteBatch` 保存一系列待应用到数据库的编辑，批内编辑按顺序执行。注意我们先调用 Delete 再调用 Put，这样当 key1 与 key2 相同时，不会错误地把值整个丢掉。

除了原子性，`WriteBatch` 还可通过把大量单次修改放入同一批来加速批量更新。

## 同步写

默认情况下，每次写入 leveldb 都是异步的：把写操作从进程推送到操作系统后即返回。从操作系统内存到持久化存储的落盘是异步完成的。可为某次写打开 sync 标志，使写操作在数据真正推送到持久化存储后才返回。（在 Posix 系统上，这通过在写返回前调用 `fsync(...)`、`fdatasync(...)` 或 `msync(..., MS_SYNC)` 实现。）

```c++
leveldb::WriteOptions write_options;
write_options.sync = true;
db->Put(write_options, ...);
```

异步写通常比同步写快上千倍。缺点是机器崩溃可能导致最近若干次更新丢失。注意：仅写入进程崩溃（而非整机重启）不会造成丢失，因为即使 sync 为 false，更新也会在被视为完成前从进程内存推入操作系统。

异步写在很多场景下可以安全使用。例如批量加载数据时，崩溃后可通过重新开始批量加载来处理丢失的更新。也可以采用混合方案：每 N 次写做一次同步写，崩溃后从上一次完成的同步写位置继续加载。（同步写可更新一个标记，描述崩溃后应从何处重启。）

`WriteBatch` 也为异步写提供了替代方案。可将多次更新放入同一 WriteBatch，再以同步写（即 `write_options.sync = true`）一次性应用。同步写的额外开销会分摊到批内所有写上。

## 并发

同一数据库一次只能由一个进程打开。leveldb 实现会向操作系统获取锁以防止误用。在单个进程内，同一个 `leveldb::DB` 对象可被多个并发线程安全共享。也就是说，不同线程可向同一数据库写入、获取迭代器或调用 Get，无需外部同步（leveldb 实现会自动做必要同步）。但其他对象（如 Iterator 和 `WriteBatch`）可能需要外部同步。若两个线程共享此类对象，必须用自己的加锁协议保护访问。更多细节见公开头文件。

## 迭代

下面的示例演示如何打印数据库中所有键值对：

```c++
leveldb::Iterator* it = db->NewIterator(leveldb::ReadOptions());
for (it->SeekToFirst(); it->Valid(); it->Next()) {
  cout << it->key().ToString() << ": "  << it->value().ToString() << endl;
}
assert(it->status().ok());  // 检查扫描过程中是否有错误
delete it;
```

下面的变体展示如何只处理区间 [start, limit) 内的键：

```c++
for (it->Seek(start);
   it->Valid() && it->key().ToString() < limit;
   it->Next()) {
  ...
}
```

也可以按逆序处理条目。（注意：反向迭代可能比正向迭代稍慢。）

```c++
for (it->SeekToLast(); it->Valid(); it->Prev()) {
  ...
}
```

## 快照

快照提供对整个键值存储状态的一致只读视图。可将 `ReadOptions::snapshot` 设为非 NULL，表示读应基于 DB 状态的某个特定版本。若 `ReadOptions::snapshot` 为 NULL，则读操作基于当前状态的隐式快照。

快照由 `DB::GetSnapshot()` 创建：

```c++
leveldb::ReadOptions options;
options.snapshot = db->GetSnapshot();
... 对 db 做一些更新 ...
leveldb::Iterator* iter = db->NewIterator(options);
... 用 iter 读取快照创建时的状态 ...
delete iter;
db->ReleaseSnapshot(options.snapshot);
```

注意：快照不再需要时应通过 `DB::ReleaseSnapshot` 接口释放。这样实现可以丢弃仅为支持该快照读取而维护的状态。

## Slice

上文 `it->key()` 和 `it->value()` 的返回值是 `leveldb::Slice` 类型的实例。Slice 是一个简单结构，包含长度以及指向外部字节数组的指针。返回 Slice 比返回 `std::string` 更便宜，因为无需拷贝可能很大的键和值。另外，leveldb 方法不返回以空字符结尾的 C 风格字符串，因为 leveldb 的键和值允许包含 `'\0'` 字节。

C++ 字符串和以空字符结尾的 C 风格字符串都可以方便地转换为 Slice：

```c++
leveldb::Slice s1 = "hello";

std::string str("world");
leveldb::Slice s2 = str;
```

Slice 也可以方便地转回 C++ 字符串：

```c++
std::string str = s1.ToString();
assert(str == std::string("hello"));
```

使用 Slice 时要小心：调用方必须保证 Slice 指向的外部字节数组在 Slice 使用期间仍然存活。例如下面的代码是有 bug 的：

```c++
leveldb::Slice slice;
if (...) {
  std::string str = ...;
  slice = str;
}
Use(slice);
```

当 if 语句离开作用域时，str 会被销毁，slice 的底层存储也随之消失。

## 比较器

前面的例子使用了默认的键排序函数，即按字节字典序排序。打开数据库时也可以提供自定义比较器。例如，假设每个数据库键由两个数字组成，应按第一个数字排序，相同时再按第二个数字排序。首先定义 `leveldb::Comparator` 的合适子类来表达这些规则：

```c++
class TwoPartComparator : public leveldb::Comparator {
 public:
  // 三路比较函数：
  //   若 a < b：返回负值
  //   若 a > b：返回正值
  //   否则：返回零
  int Compare(const leveldb::Slice& a, const leveldb::Slice& b) const {
    int a1, a2, b1, b2;
    ParseKey(a, &a1, &a2);
    ParseKey(b, &b1, &b2);
    if (a1 < b1) return -1;
    if (a1 > b1) return +1;
    if (a2 < b2) return -1;
    if (a2 > b2) return +1;
    return 0;
  }

  // 下面几个方法暂时可忽略：
  const char* Name() const { return "TwoPartComparator"; }
  void FindShortestSeparator(std::string*, const leveldb::Slice&) const {}
  void FindShortSuccessor(std::string*) const {}
};
```

然后用该自定义比较器创建数据库：

```c++
TwoPartComparator cmp;
leveldb::DB* db;
leveldb::Options options;
options.create_if_missing = true;
options.comparator = &cmp;
leveldb::Status status = leveldb::DB::Open(options, "/tmp/testdb", &db);
...
```

### 向后兼容

比较器 Name 方法的返回值在数据库创建时会附着到数据库上，并在之后每次打开时检查。若名称变化，`leveldb::DB::Open` 调用会失败。因此，仅当新的键格式与比较函数与现有数据库不兼容、并且可以丢弃所有现有数据库内容时，才应更改名称。

不过，稍作预先规划仍可随时间逐步演进键格式。例如，可在每个键末尾存储版本号（多数用途一个字节即可）。当你希望切换到新键格式时（例如为 `TwoPartComparator` 处理的键增加可选的第三部分），可：(a) 保持相同的比较器名称；(b) 为新键递增版本号；(c) 修改比较函数，根据键中的版本号决定如何解释它们。

## 性能

可通过修改 `include/options.h` 中定义的类型默认值来调优性能。

### 块大小

leveldb 将相邻键归入同一块，块是与持久化存储之间传输的单位。默认块大小约为 4096 未压缩字节。主要做数据库内容批量扫描的应用可能希望增大该值。大量做小值点查的应用，若性能测量表明有改善，可改用更小的块。使用小于一千字节或大于几兆字节的块通常没有太大收益。另请注意，较大的块更有利于压缩。

### 压缩

每个块在写入持久化存储前会单独压缩。默认开启压缩，因为默认压缩方法非常快，并且对不可压缩数据会自动禁用。极少数情况下应用可能希望完全禁用压缩，但应仅在基准测试显示性能有提升时这样做：

```c++
leveldb::Options options;
options.compression = leveldb::kNoCompression;
... leveldb::DB::Open(options, name, ...) ....
```

### 缓存

数据库内容存放在文件系统中的一组文件里，每个文件存储一串压缩块。若 `options.block_cache` 非 NULL，则用于缓存常用的未压缩块内容。

```c++
#include "leveldb/cache.h"

leveldb::Options options;
options.block_cache = leveldb::NewLRUCache(100 * 1048576);  // 100MB 缓存
leveldb::DB* db;
leveldb::DB::Open(options, name, &db);
... 使用 db ...
delete db
delete options.block_cache;
```

注意缓存保存的是未压缩数据，因此应按应用层数据大小来设置容量，不要按压缩后体积缩减。（压缩块的缓存留给操作系统缓冲缓存，或客户端提供的自定义 Env 实现。）

执行批量读取时，应用可能希望禁用缓存，以免批量读取处理的数据把大部分缓存内容挤掉。可用按迭代器的选项实现：

```c++
leveldb::ReadOptions options;
options.fill_cache = false;
leveldb::Iterator* it = db->NewIterator(options);
for (it->SeekToFirst(); it->Valid(); it->Next()) {
  ...
}
delete it;
```

### 键布局

注意磁盘传输与缓存的单位是块。相邻键（按数据库排序顺序）通常会放在同一块中。因此，应用可通过把经常一起访问的键放在彼此附近、把不常用的键放在键空间的独立区域来提升性能。

例如，假设我们在 leveldb 之上实现一个简单文件系统。可能希望存储的条目类型有：

    filename -> permission-bits, length, list of file_block_ids
    file_block_id -> data

我们可能希望给 filename 键加一个字母前缀（比如 '/'），给 `file_block_id` 键加另一个字母前缀（比如 '0'），这样仅扫描元数据时不会被迫拉取并缓存体积庞大的文件内容。

### 过滤器

由于 leveldb 数据在磁盘上的组织方式，单次 `Get()` 可能涉及多次磁盘读。可选的 FilterPolicy 机制可大幅减少磁盘读次数。

```c++
leveldb::Options options;
options.filter_policy = NewBloomFilterPolicy(10);
leveldb::DB* db;
leveldb::DB::Open(options, "/tmp/testdb", &db);
... 使用数据库 ...
delete db;
delete options.filter_policy;
```

上面的代码为数据库关联了基于 Bloom 过滤器的过滤策略。基于 Bloom 过滤器的过滤依赖为每个键在内存中保留若干比特（本例为每键 10 比特，即传给 `NewBloomFilterPolicy` 的参数）。该过滤器可将 Get() 调用中不必要的磁盘读大约减少 100 倍。增大每键比特数会带来更大的减少幅度，但代价是更多内存。我们建议工作集无法放入内存且大量做随机读的应用设置过滤策略。

若使用自定义比较器，应确保所用过滤策略与比较器兼容。例如，考虑一个比较时忽略尾部空格的比较器。此时不得使用 `NewBloomFilterPolicy`。应用应提供同样忽略尾部空格的自定义过滤策略。例如：

```c++
class CustomFilterPolicy : public leveldb::FilterPolicy {
 private:
  leveldb::FilterPolicy* builtin_policy_;

 public:
  CustomFilterPolicy() : builtin_policy_(leveldb::NewBloomFilterPolicy(10)) {}
  ~CustomFilterPolicy() { delete builtin_policy_; }

  const char* Name() const { return "IgnoreTrailingSpacesFilter"; }

  void CreateFilter(const leveldb::Slice* keys, int n, std::string* dst) const {
    // 去掉尾部空格后使用内置 bloom 过滤器代码
    std::vector<leveldb::Slice> trimmed(n);
    for (int i = 0; i < n; i++) {
      trimmed[i] = RemoveTrailingSpaces(keys[i]);
    }
    builtin_policy_->CreateFilter(trimmed.data(), n, dst);
  }
};
```

高级应用可提供不使用 bloom 过滤器、而用其他机制汇总键集合的过滤策略。详见 `leveldb/filter_policy.h`。

## 校验和

leveldb 为其在文件系统中存储的所有数据关联校验和。对校验和的校验激进程度有两个独立控制项：

可将 `ReadOptions::verify_checksums` 设为 true，强制对某次读从文件系统代表读取的所有数据做校验和验证。默认不做此类验证。

可在打开数据库前将 `Options::paranoid_checks` 设为 true，使数据库实现一检测到内部损坏就报错。根据损坏发生在数据库的哪一部分，错误可能在打开时抛出，也可能在之后的另一次数据库操作中抛出。默认关闭偏执检查，以便即使持久化存储的部分内容已损坏，数据库仍可使用。

若数据库已损坏（例如开启偏执检查时无法打开），可使用 `leveldb::RepairDB` 函数尽量恢复数据。

## 近似大小

可用 `GetApproximateSizes` 方法获取一个或多个键区间所占用的大致文件系统字节数。

```c++
leveldb::Range ranges[2];
ranges[0] = leveldb::Range("a", "c");
ranges[1] = leveldb::Range("x", "z");
uint64_t sizes[2];
db->GetApproximateSizes(ranges, 2, sizes);
```

上述调用会将 `sizes[0]` 设为键区间 `[a..c)` 所用的大致文件系统字节数，将 `sizes[1]` 设为键区间 `[x..z)` 所用的大致字节数。

## 环境（Environment）

leveldb 实现发出的所有文件操作（以及其他操作系统调用）都经由 `leveldb::Env` 对象转发。高级客户端可能希望提供自己的 Env 实现以获得更好的控制。例如，应用可在文件 IO 路径中引入人为延迟，以限制 leveldb 对系统中其他活动的影响。

```c++
class SlowEnv : public leveldb::Env {
  ... Env 接口的实现 ...
};

SlowEnv env;
leveldb::Options options;
options.env = &env;
Status s = leveldb::DB::Open(options, ...);
```

## 移植

可通过为 `leveldb/port/port.h` 导出的类型/方法/函数提供平台相关实现，将 leveldb 移植到新平台。详见 `leveldb/port/port_example.h`。

此外，新平台可能需要新的默认 `leveldb::Env` 实现。示例见 `leveldb/util/env_posix.h`。

## 其他信息

leveldb 实现细节见以下文档：

1. [实现说明](impl.md)
2. [不可变 Table 文件格式](table_format.md)
3. [日志文件格式](log_format.md)
