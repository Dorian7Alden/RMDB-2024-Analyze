# 01. 磁盘管理器（Disk Manager）

## 概述

Disk Manager 是存储层的最底层，负责直接与磁盘文件打交道。它做的事情非常直观：把数据写到磁盘文件、从磁盘文件读数据、管理文件和目录。

**框架已完整实现此模块**（`db2026-x/src/storage/disk_manager.cpp` 与参考实现仅有代码格式差异），本章简要讲解其数据结构和接口。

## 前置知识

- **`fd`（File Descriptor，文件描述符）**：Linux 中打开文件后得到的一个整数编号，后续读写操作都用这个编号指代文件，而不是每次传文件路径。
- **`lseek` / `write` / `read`**：这三个是 **POSIX 系统调用**（声明于 `<unistd.h>`），不是 C++ 标准库函数。它们是 Unix/Linux 操作系统提供的 C 语言接口，Windows 上不可用（这也是 RMDB 要求 Ubuntu 环境的原因）。
  - `lseek(fd, offset, whence)`：移动文件读写的"光标"位置。`whence` 指定参考点：
    - `SEEK_SET`（值为 0）：从**文件开头**向后偏移 offset 字节
    - `SEEK_CUR`（值为 1）：从**当前位置**向后偏移
    - `SEEK_END`（值为 2）：从**文件末尾**向后偏移
    - Disk Manager 只用 `SEEK_SET`，因为页面位置是绝对偏移量（`page_no × PAGE_SIZE`），不需要相对定位。
  - `read(fd, buf, n)`：从光标位置读 n 字节到 buf。
  - `write(fd, buf, n)`：从光标位置取 buf 的前 n 字节写入文件。
- **`PAGE_SIZE`**：RMDB 中页面大小固定为 4096 字节（`common/config.h:38`），偏移量 = 页号 × 4096。

## 数据结构

Disk Manager 的核心成员变量（`disk_manager.h:93-105`）：

```
class DiskManager {
  std::unordered_map<std::string, int> path2fd_;   // 路径 → fd
  std::unordered_map<int, std::string> fd2path_;   // fd → 路径
  int log_fd_ = -1;                                // WAL 日志文件 fd
  std::atomic<page_id_t> fd2pageno_[MAX_FD]{};     // 每个文件已分配的页数
};
```

**举例说明**：假设数据库 `mydb` 有两张表 `student` 和 `course`：

```
path2fd_:  { "mydb/student.db" → 3, "mydb/course.db" → 4 }
fd2path_:  { 3 → "mydb/student.db", 4 → "mydb/course.db" }
fd2pageno_: [0, 0, 0, 5, 3, ...]  ← fd=3 的文件已分配 5 页, fd=4 已分配 3 页
```

## 核心接口

### write_page

`write_page` 将内存中的数据写入磁盘文件的指定页面。

```cpp
// disk_manager.cpp:32
void DiskManager::write_page(int fd, page_id_t page_no,
                             const char* data, int num_bytes) {
    off_t offset = page_no * PAGE_SIZE;
    lseek(fd, offset, SEEK_SET);
    write(fd, data, num_bytes);
}
```

- `fd`：文件描述符
- `page_no`：页号，磁盘偏移 = `page_no × 4096`
- `data`：要写入磁盘的数据缓冲区
- `num_bytes`：写入字节数，淘汰脏页时为 `PAGE_SIZE`，写文件头时为 `sizeof(RmFileHdr)` 等
- 写入是**覆盖式**的——`lseek` 定位后 `write` 覆盖原有字节，不影响其他页面

**触发 `write_page` 的所有场景**：

| 触发场景 | 源文件 | 写入内容 |
|----------|--------|----------|
| 缓冲池淘汰脏页 | `buffer_pool_instance.cpp:46` | 脏页整页数据 `PAGE_SIZE` |
| 主动刷盘（flush） | `buffer_pool_instance.cpp:187` | 指定页面 |
| 全部刷盘 | `buffer_pool_instance.cpp:250` | 某文件的所有脏页 |
| 检查点刷盘 | `buffer_pool_instance.cpp:279,298` | 检查点时的脏页 |
| 更新记录文件头 | `rm_manager.h:56,86,101` | `sizeof(RmFileHdr)` 字节 |
| 更新索引文件头 | `ix_manager.h:109,197,210` | `fhdr->tot_len_` 字节 |

### read_page

`read_page` 将磁盘文件中指定页面的数据读入内存缓冲区。

```cpp
// disk_manager.cpp:58
void DiskManager::read_page(int fd, page_id_t page_no,
                            char* data, int num_bytes) {
    off_t offset = page_no * PAGE_SIZE;
    lseek(fd, offset, SEEK_SET);
    read(fd, data, num_bytes);
}
```

- 参数含义与 `write_page` 相同，`write` 换成 `read`
- `data`：从磁盘读取的数据写入此缓冲区
- 调用场景：缓冲池未命中时（`fetch_page` 需要从磁盘加载页面）、读取文件头时（`RmFileHandle` 构造）

> **Q: read_page 读完之后为什么没有返回值？**
>
> 调用方在调用 `read_page` 之前，就已经准备好了目标缓冲区，函数只管往里填数据。以 `fetch_page` 为例：
>
> ```cpp
> // buffer_pool_instance.cpp
> disk_manager_->read_page(page_id.fd, page_id.page_no,
>                           page->get_data(), PAGE_SIZE);
> //                        ^^^^^^^^^^^^^^^^
> //                        数据直接写入 Page 对象内的 data_ 数组
> ```
>
> 调用方自己提供的缓冲区（`page->get_data()` 返回 `data_` 数组地址），所以不需要返回值——数据已经在它指定的位置了。
>
> **Q: 读入内存缓冲区的数据怎么区分？Page 的 `data_[4096]` 只是原始字节，换入换出不会乱吗？**
>
> 不会乱，区分靠三层信息：
>
> | 层级 | 机制 | 说明 |
> |------|------|------|
> | **页归属** | `Page::id_` | 每个 Page 对象身上有 `PageId {fd, page_no}`，标明"这是哪个文件的第几页" |
> | **页内结构** | 页面头 | `data_` 内部有 `RmPageHdr` 或 `IxPageHdr`，记录页内记录数、空闲位置等 |
> | **帧定位** | `page_table_` | Buffer Pool 维护的 `unordered_map<PageId, frame_id>`，给定 PageId 能找到对应 Page 对象 |
>
> 流程示意：
>
> ```
> fetch_page({fd:3, page_no:1}):
>   1. 在 frames_ 中分配一个空闲 Page 对象（如 frame_id=2）
>   2. 设置 page->id_ = {fd:3, page_no:1}          ← 标记页归属
>   3. disk_manager_->read_page(3, 1, page->data_)  ← 填入原始字节
>   4. page_table_[{fd:3, page_no:1}] = 2           ← 注册索引
>
> 之后访问:
>   page_table_[{fd:3, page_no:1}] → frame_id=2 → frames_[2] → Page 对象
> ```
>
> 原始字节不知道自己是谁——但包裹它的 Page 对象和 Buffer Pool 的索引表一起完成了定位。

### 脏页淘汰实例

缓冲池满时，`fetch_page` 调用 `update_page` 淘汰旧页装入新页。这是实际源码（`buffer_pool_instance.cpp:31-58`）：

```cpp
void BufferPoolInstance::update_page(Page* page, PageId new_page_id,
                                     frame_id_t new_frame_id) {
    // 1. 如果旧页面是脏页，先写回磁盘
    if (page->is_dirty()) {
        disk_manager_->write_page(page->get_page_id().fd,
                                  page->get_page_id().page_no,
                                  page->get_data(), PAGE_SIZE);
        page->is_dirty_ = false;
    }

    // 2. 更新 page_table_ 映射
    page_table_.erase(page->get_page_id());
    page_table_[new_page_id] = new_frame_id;

    // 3. 更新 page 身份
    page->id_ = new_page_id;
    page->pin_count_ = 0;
}
```

- `page`：被淘汰的旧页面（Victim），可能是脏页
- `new_page_id`：要装入的新页面的 PageId
- 如果旧页是脏页（`is_dirty_ == true`），先调用 `disk_manager_->write_page` 写回磁盘，再更新映射、更换身份
- 之后再通过 `disk_manager_->read_page` 从磁盘读入新页面数据（在 `fetch_page` 中完成，不在 `update_page` 里）

### allocate_page

`allocate_page` 为指定文件分配一个新页面，返回新页号。

```cpp
// disk_manager.cpp:83
page_id_t DiskManager::allocate_page(int fd) {
    return fd2pageno_[fd]++;
}
```

- `fd`：文件描述符，每个文件独立计数
- 返回：新分配的页号，分配策略为自增计数器——返回当前值，然后 +1
- 对应的 `fd2pageno_[MAX_FD]` 数组记录每个文件已分配的页数

### 文件与目录管理

| 方法 | 作用 | 系统调用 |
|------|------|----------|
| `create_file(path)` | 创建新数据库文件 | `open(O_CREAT)` |
| `destroy_file(path)` | 删除文件 | `unlink()` |
| `open_file(path)` | 打开文件，记录到 path2fd_ | `open(O_RDWR)` |
| `close_file(fd)` | 关闭文件，从映射表移除 | `close()` |
| `create_dir(path)` | 创建数据库目录 | `mkdir` 命令 |
| `destroy_dir(path)` | 删除目录 | `rm -r` 命令 |

## 常见问题解答

**Q1: 每次 write_page 都要写一整页（4096 字节）吗？**

不一定。`num_bytes` 是调用方传入的参数。实际使用中分两种情况：

| 场景 | num_bytes | 说明 |
|------|-----------|------|
| 缓冲池淘汰脏页 | `PAGE_SIZE`（4096） | 整页写回，保证磁盘页数据完整 |
| 写文件头 | `sizeof(RmFileHdr)` | 只写文件头的几十到几百字节 |

绝大多数情况下是写整页——Buffer Pool 的最小管理单位就是页，淘汰脏页时必须整页写回。

**Q2: 写入是覆盖式的吗？**

是的。`lseek` 定位到文件中的某个字节偏移后，`write` 从那个位置开始覆盖原有的字节。例如写第 1 页只覆盖字节 4096~8191，不影响其他页。

**Q3: 为什么写入总是从页首开始，而不能从页面内的指定偏移开始？**

因为 DBMS 把**页当作不可分割的原子单元**——要么整页读，要么整页写：

| 原因 | 说明 |
|------|------|
| **数据一致性** | 只写页面中间 50 字节，磁盘上就是"一半新一半旧"，无法分辨 |
| **脏页标记粒度** | Page 只有一个 `is_dirty_` 标志，若支持局部写需要逐字节追踪 |
| **硬件层面** | 操作系统和磁盘控制器在物理层按扇区（512B/4KB）读写，局部写不省 I/O |

`num_bytes` 能控制写**多少**数据，但不能控制从页内**哪个位置**开始写。`num_bytes` 的灵活性是为文件头这类特殊场景预留的。数据页面永远整页操作。

**Q4: 文件头可以只写部分，和"页是最小 I/O 单元"不矛盾吗？**

不矛盾。"整页是原子单元"是**缓冲池的设计约束**，不是 Disk Manager 的硬限制。两条路径互不干扰：

```
普通数据页: 上层 → Buffer Pool → 整页 read/write → Disk Manager → 磁盘
文件头:      RmManager/IxManager → 局部 read/write → Disk Manager → 磁盘（绕过 Buffer Pool）
```

**Q5: 脏页还没写回磁盘，服务器突然崩溃了怎么办？**

这正是 DBMS 需要 **恢复层（WAL）** 的原因。修改前先把"要改什么"记录到日志文件并强制刷盘，再修改内存中的页面。崩溃后重放日志即可恢复所有已提交的修改。

```
UPDATE student SET age=21 WHERE id=1;

  1. LogManager::add_log_to_buffer()     ← 把 UPDATE 记录写入 WAL 日志缓冲区
  2. flush_log_to_disk()                ← 强制把日志刷到磁盘（db.log）
  3. Page::GetDataMut() 修改内存中的页面  ← 改内存，标记 is_dirty_ = true
  4. unpin_page(page_id, true)           ← 释放页面引用，但不写磁盘
  ... 可能很多操作之后 ...
  5. LRU 淘汰该页 → write_page()         ← 最终持久化到数据文件

如果步骤 4 之后、步骤 5 之前崩溃了：
  → 重放步骤 1-2 的日志，恢复步骤 3 的修改 ✅
```

> WAL 的完整机制将在**第 7 章：故障恢复**中深入讲解。

## 在整体架构中的位置

Disk Manager 被 BufferPoolInstance 持有（`DiskManager*` 指针，`buffer_pool_instance.h:25`），只在缓冲池未命中时才被调用：

```
上层 (RM/IX) → Buffer Pool → [未命中] → Disk Manager → 磁盘文件
                              [命中]   → 直接返回内存页面
```

Disk Manager 对更上层（RM、IX、Executor）完全透明——那些模块根本不知道 Disk Manager 的存在，它们只跟 Buffer Pool 打交道。

## 小结

- Disk Manager 封装了所有磁盘 I/O 操作，对外提供按页读写的接口
- 核心公式：**偏移量 = page_no × PAGE_SIZE**（4096 字节）
- 页面分配使用最简单的自增计数器
- 框架已完整实现，理解其接口即可

下一节：[02. 文件头](./02-file-header.md)
