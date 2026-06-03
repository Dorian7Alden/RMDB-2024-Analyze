# 10. 框架与参考实现对比

记录层的框架（`db2026-x/`）和参考实现（`src/`）在整体结构上是一致的——数据结构定义、`RmPageHandle`、`RmManager`、`Bitmap` 都完全相同。差异集中在 `RmFileHandle` 的 CRUD 实现和 `RmScan` 的扫描策略上。

## 差异总览

| 方面 | 框架 | 参考实现 |
|------|------|----------|
| 并发保护（文件级） | 无 `latch_` 互斥锁 | 有 `std::mutex latch_` |
| 并发保护（页级） | 无 WLatch/WUnlatch | insert 和 delete 加写锁 |
| 边界检查 | fetch_page_handle 不检查 page_no 合法性 | 检查 page_no 范围并抛异常 |
| get_record 位图检查 | 不检查 bitmap，直接读 | 先检查 bitmap，无记录抛异常 |
| 空闲链表移除 | 独立方法 `remove_page_from_free_list` | 内联在 insert_record 中处理 |
| RmScan 页面缓存 | 不缓存，每次 next 都 fetch | 缓存 cur_page_handle_ |
| RmScan 结束标记 | `page_no = num_pages` | `page_no = RM_NO_PAGE` |
| RmScan.get_record | 无 | 有 |

## 差异 1：并发保护

### 文件级互斥锁

**框架**：`RmFileHandle` 没有 `latch_` 成员。

**参考实现**：`src/record/rm_file_handle.h:65`

```cpp
std::mutex latch_;
```

`create_page_handle()` 中加锁：

```cpp
RmPageHandle RmFileHandle::create_page_handle() {
  std::lock_guard lock(latch_);  // 框架没有这一行
  // ...
}
```

**为什么要加锁？**

`create_page_handle` 读取 `first_free_page_no` 并返回对应的页面。如果两个线程同时插入：
1. 线程 A 读到 `first_free_page_no = 3`
2. 线程 B 也读到 `first_free_page_no = 3`
3. 两者都往页面 3 插入，可能插到同一个 slot

| | 优点 | 缺点 |
|------|------|------|
| 无锁（框架） | 实现简单，无锁开销 | 并发插入可能冲突，不适用于多线程 |
| 有锁（参考） | 并发安全 | 锁粒度是文件级，高并发插入串行化 |

### 页级写锁

**框架**：`insert_record` 和 `delete_record` 不加 `WLatch()`。

**参考实现**：修改页面数据前加写锁。

```
page_handle.page->WLatch();   // 加锁
// ... 修改 bitmap 和 page_hdr ...
page_handle.page->WUnlatch();  // 解锁
```

| | 优点 | 缺点 |
|------|------|------|
| 无 WLatch（框架） | 简单，不需要理解读写锁 | 并发修改同一页时可能损坏数据 |
| 有 WLatch（参考） | 保证页级数据一致性 | 多一层锁开销，需要配合 RWLatch 机制 |

## 差异 2：错误处理

### fetch_page_handle 边界检查

**框架**（`db2026-x/src/record/rm_file_handle.cpp:151`）：

```cpp
RmPageHandle RmFileHandle::fetch_page_handle(int page_no) const {
  auto page = buffer_pool_manager_->fetch_page(
      PageId{.fd = fd_, .page_no = page_no});
  return RmPageHandle(&file_hdr_, page);
}
```

**参考实现**：

```cpp
RmPageHandle RmFileHandle::fetch_page_handle(int page_no) const {
  if (page_no >= file_hdr_.num_pages || page_no < 0) {
    throw PageNotExistError(disk_manager_->get_file_name(fd_), page_no);
  }
  auto page = buffer_pool_manager_->fetch_page({fd_, page_no});
  if (page == nullptr) {
    throw PageNotExistError(disk_manager_->get_file_name(fd_), page_no);
  }
  return {&file_hdr_, page};
}
```

| | 优点 | 缺点 |
|------|------|------|
| 无检查（框架） | 代码简洁 | 传入非法 page_no 时行为未定义 |
| 有检查（参考） | 尽早发现错误，抛有意义的异常 | 多两行检查代码 |

### get_record 位图检查

**框架**：不检查 bitmap，直接从槽位读数据。

**参考实现**：

```cpp
if (!Bitmap::is_set(page_handle.bitmap, rid.slot_no)) {
  throw RecordNotFoundError(rid.page_no, rid.slot_no);
}
```

| | 优点 | 缺点 |
|------|------|------|
| 不检查（框架） | 少一次位图访问 | 读已删除的记录可能得到脏数据 |
| 检查（参考） | 明确区分"记录存在与否" | 多一次 bitmap 判断 |

## 差异 3：空闲链表移除策略

**框架** 有独立的 `remove_page_from_free_list` 方法（`db2026-x/src/record/rm_file_handle.cpp:220`），遍历链表找到前驱节点再删除。

**参考实现** 没有这个方法，而是直接修改 `first_free_page_no` 来实现移除：

```cpp
if (++page_handle.page_hdr->num_records == file_hdr_.num_records_per_page) {
  // 当前页是链表头，直接移除头部
  file_hdr_.first_free_page_no = page_handle.page_hdr->next_free_page_no;
}
```

这种做法成立的前提是：**被移除的页面一定是链表头**。因为 `create_page_handle()` 返回的总是链表头，在插入过程中这个页面没被其他线程抢走（有 `latch_` 保护），所以它一定还是链表头。

| | 优点 | 缺点 |
|------|------|------|
| 框架方式 | 支持从链表任意位置移除，通用性强 | 代码复杂，需要遍历链表，性能差 |
| 参考方式 | 代码简单，O(1) 操作 | 依赖"一定是链表头"的前置条件 |

**框架为什么写了更复杂的版本？** 一种可能是框架没有 `latch_`，所以无法保证拿到的页面是链表头；也可能是在开发过程中先实现了通用版本作为防御性编程。有了 `latch_` 后，参考实现简化为直接移除头部。

## 差异 4：RmScan 扫描策略

已在 [07 RmScan](./07-record-scan.md) 中详细分析，这里简要总结：

| 方面 | 框架 | 参考实现 |
|------|------|----------|
| 页面缓存 | 每次 next 都 fetch_page_handle | 成员变量 cur_page_handle_ 缓存 |
| 缓冲池访问次数 | N 次（每条记录都 fetch+unpin） | M 次（每页只 fetch 一次，M = 页数） |
| get_record | 无，需用 RmFileHandle::get_record | 有，直接从缓存页读取 |
| 结束标记 | page_no = num_pages | page_no = RM_NO_PAGE (-1) |

**性能差距**：假设 student 表有 1000 条记录、每页 100 条（10 页），框架全表扫描会触发 1000 次缓冲池 fetch/unpin，而参考实现只需 10 次。

## 从学习角度看

框架的做法体现了"先跑通基础功能，再考虑并发和安全"的思路：

1. **先保证正确性**：CRUD 的基本逻辑在框架中已实现
2. **再补充并发保护**：`latch_` 和 `WLatch` 是参考实现新增的
3. **再补充错误处理**：边界检查和 bitmap 校验
4. **最后做性能优化**：RmScan 的页面缓存

这些差异指出了学习者在框架基础上需要提升的四个方向。在实现自己的记录层时，应该同时考虑这四个方面，而不是重复框架的简化版本。

## 源码对应

| 差异点 | 框架文件 | 参考文件 |
|--------|----------|----------|
| latch_ 互斥锁 | 无 | `src/record/rm_file_handle.h:65` |
| WLatch | `db2026-x/src/record/rm_file_handle.cpp` 无 | `src/record/rm_file_handle.cpp:56,74` |
| fetch_page_handle 检查 | `db2026-x/src/record/rm_file_handle.cpp:151` | `src/record/rm_file_handle.cpp:189` |
| get_record 位图检查 | `db2026-x/src/record/rm_file_handle.cpp:19` | `src/record/rm_file_handle.cpp:19` |
| remove_page_from_free_list | `db2026-x/src/record/rm_file_handle.cpp:220` | 无 |
| RmScan.next | `db2026-x/src/record/rm_scan.cpp:30` | `src/record/rm_scan.cpp:35` |

上一节：[09-record-structure-example.md](./09-record-structure-example.md) | 下一节：[12-record-layer-summary.md](./12-record-layer-summary.md)
