# LevelDB 中文文档

本目录为 `doc/` 下官方英文文档的中文译本，便于学习阅读。术语尽量保留英文原文（如 MemTable、SSTable、Compaction），便于对照源码。

| 文档 | 说明 | 英文原文 |
|------|------|----------|
| [index.md](index.md) | 使用指南（API、并发、性能调优等） | [../index.md](../index.md) |
| [impl.md](impl.md) | 实现说明（文件组织、LSM、压缩、恢复） | [../impl.md](../impl.md) |
| [table_format.md](table_format.md) | SSTable / Table 文件格式 | [../table_format.md](../table_format.md) |
| [log_format.md](log_format.md) | WAL 日志文件格式 | [../log_format.md](../log_format.md) |
| [benchmark.html](benchmark.html) | 2011 年性能基准测试报告 | [../benchmark.html](../benchmark.html) |

建议阅读顺序：`index.md` → `impl.md` → `log_format.md` / `table_format.md`。
