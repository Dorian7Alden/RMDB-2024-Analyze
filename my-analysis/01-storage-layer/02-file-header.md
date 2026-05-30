# 02. 文件头（File Header）

## 概述

每个数据库文件（`.db` / `.idx`）的第 0 页存放的是该文件的**元数据说明书**——不是实际数据，而是告诉系统"这个文件的结构是什么样的"。

文件头虽然由 Disk Manager 负责读写（`write_page(fd, 0, ...)`），但它**绕过缓冲池**直接操作磁盘，因为文件头读写频率极低，只在创建、关闭、检查点等时机触发。

## 缩写速查

| 缩写 | 全称 | 中文含义 |
|------|------|----------|
| `hdr` | Header | 头（文件头） |
| `RmFileHdr` | Record Manager File Header | 记录文件的文件头 |
| `IxFileHdr` | IndeX File Header | 索引文件的文件头 |

## RmFileHdr

`RmFileHdr`（Record Manager File Header）是表数据文件的文件头，定义在 `record/rm_defs.h:24-31`，每个 `.db` 文件一份。

```
struct RmFileHdr {
  int record_size;                // 每条记录多大？比如 student 表 100 字节
  int num_pages;                  // 当前文件一共多少页？
  int num_records_per_page;       // 一页最多能放几条记录？
  int first_free_page_no;         // 第一个有空位的页号（插入时直接去那页找空位）
  int bitmap_size;                // 每页的位图多大？
};
```

## IxFileHdr

`IxFileHdr`（IndeX File Header）是索引文件的文件头，定义在 `index/ix_defs.h:27`，每个 `.idx` 文件一份。

```
class IxFileHdr {
  page_id_t root_page_;           // B+ 树根节点的页号
  int num_pages_;                 // 文件总页数
  int col_num_;                   // 索引包含几个字段
  std::vector<ColType> col_types_; // 每个字段的类型
  std::vector<int> col_lens_;     // 每个字段的长度
  int col_tot_len_;               // 字段总长度
  int btree_order_;               // B+ 树每个节点最多放多少个键
  page_id_t first_leaf_;          // 第一个叶子节点的页号
  page_id_t last_leaf_;           // 最后一个叶子节点的页号
  ...
};
```

## 在磁盘上的布局

文件头独占第 0 页，剩余空间闲置不用。实际记录从第 1 页开始（`RM_FIRST_RECORD_PAGE = 1`，`record/rm_defs.h:20`）。这样设计是为了边界清晰：页 0 = 元数据，页 1+ = 数据，缓冲池只管理数据页。

以 student 表为例：

```
student.db:

┌──────────────────────────────────────────────────┐
│ 第 0 页（RmFileHdr，~20 字节，独占）              │
│   record_size = 100                              │
│   num_pages = 25                                 │
│   num_records_per_page = 40                      │
│   first_free_page_no = 3                         │
│   bitmap_size = 5                                │
│                                                  │
│   剩余 4076 字节空白，不存数据                     │
├──────────────────────────────────────────────────┤
│ 第 1 页 ~ 第 24 页（数据页，走缓冲池）             │
│   每页 40 条记录，共 1000 条                      │
│   缓冲池整页 4096 字节管理                        │
└──────────────────────────────────────────────────┘
```

## 读写路径

文件头**绕过缓冲池**，直接通过 Disk Manager 读写。触发时机：

```
创建表/索引时:
  create_file() / create_index()
    → 在内存中初始化 file_hdr 结构体
    → disk_manager_->write_page(fd, 0, &file_hdr, sizeof(file_hdr))
    → 直接把文件头写到第 0 页

关闭表/索引时:
  close_file() / close_index()
    → 文件头在运行过程中可能被修改了（如 num_pages 增加了）
    → disk_manager_->write_page(fd, 0, &file_hdr, sizeof(file_hdr))
    → 把最新的文件头写回磁盘

检查点刷盘时:
  flush_file() / flush_index()
    → 同关闭，但不关闭 fd，只是确保持久化
```

> `num_pages` 和 `first_free_page_no` 在每次新增页面或页面写满时都会改变。但**改变的是内存中 `RmFileHandle::file_hdr_` 的副本**，磁盘上第 0 页的文件头只在 close/flush/checkpoint 时才被覆盖写入——这和脏页的逻辑完全一样：内存中改多次，磁盘上延迟批量写回。

## 小结

- 文件头独占第 0 页，第 1 页起才是真正的数据
- 记录文件用 `RmFileHdr`，索引文件用 `IxFileHdr`
- 文件头绕过缓冲池，直接由 Disk Manager 读写，触发时机只有创建、关闭、检查点

下一节：[03. 页面数据结构](./03-page.md)
