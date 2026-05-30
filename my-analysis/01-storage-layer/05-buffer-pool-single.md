# 05. 单实例缓冲池（框架版本）

## 概述

框架 `db2026-x` 给出了一个简单的单实例 BufferPoolManager。它用一把大锁保护所有操作，所有请求串行排队。这个版本**功能完整但性能不足**，是理解缓冲池核心逻辑的最佳起点。

> **学习策略**：先吃透这个简单版本，搞清楚 `fetch_page`、`unpin_page`、`find_victim_page`、`update_page` 四个方法，再看下一节的多实例优化就很容易了。

## 数据结构

```cpp
// db2026-x/src/storage/buffer_pool_manager.h（框架版本）
class BufferPoolManager {
  size_t pool_size_;                                      // Frame 总数 = 65536
  Page* pages_;                                           // Page 数组，大小 = pool_size_
  std::unordered_map<PageId, frame_id_t, PageIdHash> page_table_;  // PageId → frame_id
  std::list<frame_id_t> free_list_;                       // 空闲 frame 链表
  DiskManager* disk_manager_;                             // 未命中时读磁盘
  Replacer* replacer_;                                    // LRU 替换器
  std::mutex latch_;                                      // 全局互斥锁
};
```

**实例**：假设 student 表刚刚打开，缓冲池刚初始化：

```
pool_size_ = 65536
pages_ = [Page_0, Page_1, Page_2, ..., Page_65535]   ← 65536 个空 Page
page_table_ = {}                                       ← 空的，还没缓存任何页
free_list_ = [0, 1, 2, ..., 65535]                    ← 所有 frame 都是空闲的
```

## 核心方法实现

### find_victim_page

缓冲池满时，需要找一个"受害者"frame 来替换。优先从 free_list_ 取，没有空闲才找 Replacer：

```
输入:  frame_id_t* frame_id — 输出参数，用于传出被选中的 frame 编号
      内部读取 free_list_ 和 replacer_ 的状态（隐式输入）
输出:  bool — true 找到可用 frame / false 无可用 frame
```

```cpp
// buffer_pool_manager.cpp:14-25（框架版本）
bool BufferPoolManager::find_victim_page(frame_id_t* frame_id) {
    if (free_list_.empty()) {                // 没有空闲 frame
        if (!replacer_->victim(frame_id)) {  // 让 LRU 选一个
            return false;                    // LRU 也选不出来（都 pin 着）
        }
        return true;
    }
    *frame_id = free_list_.front();          // 有空闲，直接用第一个
    free_list_.pop_front();
    return true;
}
```

> **为什么参数是 `frame_id_t*`，却说"没有输入"？**
>
> 这里的 `frame_id` 参数是 C++ 中常见的**输出参数**写法——调用方传一个变量的地址进来，函数把结果写进去。它不是真正意义上的"输入数据"，只是借指针往外传结果。真正决定选哪个 frame 的信息来自 `free_list_` 和 `replacer_` 这两个内部成员变量（隐式输入）。
>
> ```cpp
> // 调用方的视角：
> frame_id_t victim_id;
> find_victim_page(&victim_id);  // &victim_id 是传地址，让函数往里写结果
> // 调用后 victim_id 就是被选中的 frame 编号
> ```

### update_page

当 victim frame 被选中后，需要用它来装新的页面数据：

```
输入:  Page* page (受害者 Page), PageId new_page_id (新页面标识),
       frame_id_t new_frame_id
输出:  void（直接修改 page 的内容）
```

```cpp
// buffer_pool_manager.cpp:27-49（框架版本）
void BufferPoolManager::update_page(Page* page, PageId new_page_id,
                                     frame_id_t new_frame_id) {
    // 1. 如果旧页面是脏页，先写回磁盘
    if (page->is_dirty()) {
        disk_manager_->write_page(page->get_page_id().fd,
                                  page->get_page_id().page_no,
                                  page->get_data(), PAGE_SIZE);
        page->is_dirty_ = false;
    }
    // 2. 从 page_table_ 移除旧映射
    if (page->get_page_id().page_no != INVALID_PAGE_ID) {
        page_table_.erase(page->get_page_id());
    }
    // 3. 注册新映射
    if (new_page_id.page_no != INVALID_PAGE_ID) {
        page_table_.emplace(new_page_id, new_frame_id);
    }
    // 4. 重置页面数据为新页
    page->reset_memory();       // 清零 data_[PAGE_SIZE]
    page->id_ = new_page_id;    // 更新 PageId
    page->is_dirty_ = false;
}
```

**实例**：LRU 选中 frame 7（当前装的是 course 表第 0 页，且是脏页），要装入 student 表第 5 页：

```
update_page(&pages_[7], {fd:3, page_no:5}, 7)
  1. pages_[7].is_dirty() == true → write_page(4, 0, pages_[7].data_, 4096)
  2. page_table_.erase({fd:4, page_no:0})       ← 移除 course 的映射
  3. page_table_.emplace({fd:3, page_no:5}, 7)  ← 注册 student 的映射
  4. pages_[7].id_ = {fd:3, page_no:5}          ← 换身份
     pages_[7].reset_memory()                     ← 清零
```

### fetch_page

这是最核心的方法，所有上层模块都通过它来获取页面：

```
输入:  PageId page_id (fd + page_no)
输出:  Page* (内存中的页面指针，pin_count 已 +1)
```

> **`latch_` 与 `lock` 是什么？**
>
> `latch_` 是 `std::mutex` 类型的互斥锁（`buffer_pool_manager.h:34`），保证同一时刻只有一个线程能进入方法体，其他线程在外等待。
> `std::scoped_lock lock{latch_}` 是 C++17 的 RAII 锁包装——构造时自动加锁，函数结束（return 或抛异常）时自动解锁，不会忘记释放。相当于"进门自动锁门，出门自动开门"。
>
> 本节只需知道：每个方法开头那行 `scoped_lock` 就是拿锁，方法结束时自动放锁。多线程问题留到多实例版本再深入。

> **问：`fetch_page` 已经加了锁，它内部调用的 `find_victim_page` 和 `update_page` 还需要再加锁吗？**
>
> 不需要，而且绝对不能加。同一个线程里对 `std::mutex` 重复加锁会**死锁**（自己等自己释放）。这些子函数只在 `fetch_page`、`new_page` 等主方法内部调用，调用时锁已经持有了，子函数直接操作共享数据即可。看一眼源码就能验证——`find_victim_page` 和 `update_page` 的函数体里确实没有 `scoped_lock`。
>
> **问：`std::scoped_lock lock{latch_};` 这个语法怎么理解？为什么用花括号？**
>
> ```cpp
> std::scoped_lock lock{latch_};  // ← 这是什么？
> ```
>
> - `std::scoped_lock` 是 C++17 标准库里的一个**类模板**，专门用来包装互斥锁
> - `lock` 是这个类的一个**对象**（变量），名字可以随便取，叫 `guard` 也行
> - `{latch_}` 是花括号初始化（列表初始化），等价于 `std::scoped_lock lock(latch_);`，两种写法完全一样，只是代码风格偏好
>
> 完整的"翻译"：**创建一个 `scoped_lock` 类型的对象 `lock`，把 `latch_` 传进去让它管着。** 这个对象活着的时候锁就持有着，对象销毁时（出了作用域）自动释放锁。

```cpp
// buffer_pool_manager.cpp:72-104（框架版本）
Page* BufferPoolManager::fetch_page(PageId page_id) {
    std::scoped_lock lock{latch_};           // 全局锁

    auto it = page_table_.find(page_id);
    if (it != page_table_.end()) {           // 命中！
        frame_id_t frame_id = it->second;
        replacer_->pin(frame_id);            // 通知 LRU：这个 frame 被用了
        pages_[frame_id].pin_count_++;
        return &pages_[frame_id];
    }

    // 未命中，找 victim
    frame_id_t frame_id;
    if (!find_victim_page(&frame_id)) {
        return nullptr;                      // 找不到，返回空
    }

    Page* page = &pages_[frame_id];
    update_page(page, page_id, frame_id);    // 替换旧页面

    // 从磁盘读入目标页
    disk_manager_->read_page(page_id.fd, page_id.page_no,
                             page->get_data(), PAGE_SIZE);

    replacer_->pin(frame_id);
    page->pin_count_ = 1;
    return page;
}
```

**实例**：扫描 student 表，fetch_page({fd:3, page_no:0})：

```
fetch_page({fd:3, page_no:0})
  → page_table_ 中查找 {fd:3, page_no:0}
  → 未命中（首次访问）
  → find_victim_page → free_list_ 非空 → 返回 frame_id=0
  → update_page(&pages_[0], {fd:3, page_no:0}, 0)
    → 旧页面 pin_count=0, is_dirty=false → 无需写回
    → page_table_.emplace({fd:3, page_no:0}, 0)
    → pages_[0].id_ = {fd:3, page_no:0}
  → disk_manager_->read_page(3, 0, pages_[0].data_, 4096)
  → replacer_->pin(0) → LRU 标记 frame 0 被使用
  → pages_[0].pin_count_ = 1
  → 返回 &pages_[0]
```

### unpin_page

```
输入:  PageId page_id, bool is_dirty
输出:  bool (成功/失败)
```

```cpp
bool BufferPoolManager::unpin_page(PageId page_id, bool is_dirty) {
    std::scoped_lock lock{latch_};
    auto it = page_table_.find(page_id);
    if (it == page_table_.end()) return false;

    frame_id_t frame_id = it->second;
    Page* page = &pages_[frame_id];

    if (page->pin_count_ == 0) return false;  // 不能 unpin 一个没 pin 的页

    page->pin_count_--;
    if (is_dirty) page->is_dirty_ = true;

    if (page->pin_count_ == 0) {
        replacer_->unpin(frame_id);           // 通知 LRU：这个 frame 可以淘汰了
    }
    return true;
}
```

## 单实例版本的问题

框架用 `std::mutex latch_` 保护所有操作，存在严重的性能瓶颈：

```
线程 A: fetch_page({fd:3, page_no:0})  ─┐
线程 B: fetch_page({fd:4, page_no:0})  ─┤ 排队等待，即使访问不同文件
线程 C: unpin_page(...)                 ─┤
线程 D: fetch_page({fd:3, page_no:1})  ─┘
```

所有线程串行化执行，多核 CPU 完全无法发挥作用。下一节的多实例版本就是为了解决这个问题。

## 小结

- 单实例版本虽然简单，但它包含了缓冲池的完整逻辑
- 四个关键方法：`find_victim_page` → `update_page` → `fetch_page` → `unpin_page`
- `fetch_page` 的流程：查 page_table_ → 命中返回 / 未命中 → 找 victim → update_page → 读磁盘
- 全局 `std::mutex` 导致所有操作串行化，性能差

下一节：[06. 多实例缓冲池](./06-buffer-pool-multi.md)
