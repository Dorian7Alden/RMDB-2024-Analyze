# 06. IxScan 索引扫描

`IxScan` 实现 B+ 树的范围扫描——利用叶节点的双向链表，从 lower 遍历到 upper。

## 原理

B+ 树的叶节点通过 `prev_leaf` 和 `next_leaf` 指针构成双向链表：

```
first_leaf → [叶1] ⇄ [叶2] ⇄ [叶3] ⇄ ... ← last_leaf
```

`IxScan` 不需要从根节点逐层查找每个键——定位到起始位置后，沿链表顺序遍历即可。

## 接口

`src/index/ix_scan.h:21`

```cpp
// src/index/ix_scan.h:21
class IxScan : public RecScan {
  const IxIndexHandle* ih_;        // B+ 树句柄
  Iid iid_;                        // 当前位置
  Iid end_;                        // 结束位置
  std::shared_ptr<IxNodeHandle> cur_node_handle_;  // 当前叶节点

  void next();                     // 移动到下一个 Rid
  bool is_end();                   // iid_ == end_
  Rid rid();                       // 返回当前 Rid
  RmRecord get_key();              // 获取当前记录的实际键
};
```

## 使用方式

```cpp
// 扫描年龄在 [20, 30) 之间的所有记录
auto lower = ih->lower_bound(key_20);
auto upper = ih->upper_bound(key_30);
IxScan scan(ih.get(), lower, upper, bpm);

while (!scan.is_end()) {
  Rid rid = scan.rid();
  auto record = rm_file_handle->get_record(rid);  // 从记录层取实际数据
  scan.next();
}
```

## 关键点

- **锁**：构造函数对当前叶节点加 **Page 级读锁**（`RLatch`），锁住该叶节点的整个页面（4096 字节）。因为扫描只沿链表向前走、只读不写，读锁足够。析构函数负责解锁（`RUnlatch`）并 unpin 页面
- `next()` 在当前叶节点遍历，到头了沿 `next_leaf` 跳到下一个叶节点
- 叶节点链表只在叶子层，内部节点不参与扫描

框架中 IxScan 的实现较完整，作为参考学习即可。

## 源码对应

| 内容 | 文件 | 行号 |
|------|------|------|
| IxScan 类定义 | `src/index/ix_scan.h` | 21-56 |
| IxScan 实现 | `src/index/ix_scan.cpp` | 1-80 |

上一节：[05-index-manager.md](./05-index-manager.md) | 下一节：[07-index-interaction.md](./07-index-interaction.md)
