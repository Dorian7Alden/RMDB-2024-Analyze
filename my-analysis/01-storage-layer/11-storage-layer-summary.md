# 11. 存储层总结

存储层共 14 个小节，覆盖了数据如何从磁盘到内存的完整链路：

```
Disk Manager → Page → Buffer Pool → Replacer → Page Guard → RWLatch
   (磁盘读写)   (数据载体)  (缓存管理)  (页面淘汰)   (RAII安全)  (并发控制)
```

| 模块 | 框架状态 | 核心学习点 |
|------|---------|-----------|
| Disk Manager | 已完整实现 | 偏移量 = page_no × PAGE_SIZE |
| Page | 基础结构已给 | PageId、pin_count、脏页标记 |
| Buffer Pool（单实例） | 骨架已给 | fetch/unpin/victim/update 四方法 |
| Buffer Pool（多实例） | **需从零实现** | 哈希分区 + 16 路并发 |
| Clock Replacer | **需从零实现** | 循环指针近似 LRU |
| Page Guard | **需从零实现** | RAII 自动 unpin + 三级锁守卫 |
| RWLatch | **需从零实现** | shared_mutex 封装，页级读写锁 |

下一章：[第 2 章：记录层](../02-record-layer/README.md)（待编写）
