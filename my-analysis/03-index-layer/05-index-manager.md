# 05. IxManager 索引管理器

`IxManager` 管理索引文件的生命周期，与记录层的 `RmManager` 对应。

## 结构

`src/index/ix_manager.h:21`

```cpp
// src/index/ix_manager.h:21
class IxManager {
  DiskManager* disk_manager_;
  BufferPoolManager* buffer_pool_manager_;
};
```

持有存储层指针，与 `RmManager` 完全相同。

## 文件操作

| 方法 | 作用 |
|------|------|
| `create_index` | 创建索引文件，初始化文件头（第0页）和叶头（第1页），根节点（第2页） |
| `open_index` | 打开索引文件，返回 IxIndexHandle |
| `close_index` | 关闭：写回文件头、flush 脏页、清空页表 |
| `destroy_index` | 删除索引文件 |

`create_index` 比 `create_file`（记录层）多做了两步：初始化叶头页面（叶节点链表哨兵）和根节点页面。
`close_index` 流程与 `RmManager::close_file` 完全一致。

## 与 RmManager 的对比

| | RmManager | IxManager |
|------|-----------|----------|
| 文件后缀 | `.db` | `.idx` |
| 初始页面数 | 1（文件头） | 3（文件头+叶头+根） |
| 文件头 | RmFileHdr | IxFileHdr |
| 返回句柄 | RmFileHandle | IxIndexHandle |

框架中 `IxManager` 已完整实现，不需要修改。

## 源码对应

| 内容 | 文件 | 行号 |
|------|------|------|
| IxManager 完整实现 | `src/index/ix_manager.h` | 1-80 |

上一节：[04c-btree-delete.md](./04c-btree-delete.md) | 下一节：[06-index-scan.md](./06-index-scan.md)
