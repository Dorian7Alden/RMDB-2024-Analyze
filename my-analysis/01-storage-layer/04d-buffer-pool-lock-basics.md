# 04d. 锁的基本认识

## 问题起源

在 [04b. 单实例缓冲池](./04b-buffer-pool-single.md) 中，`fetch_page` 等方法开头都有这样一行：

```cpp
std::scoped_lock lock{latch_};
```

这行代码涉及两个东西——`latch_`（互斥锁本体）和 `std::scoped_lock`（RAII 锁包装器）。下面逐个说清楚。

## latch_ — 互斥锁本体

```cpp
// db2026-x/src/storage/buffer_pool_manager.h:34
std::mutex latch_;
```

`latch_` 是 `std::mutex` 类型的变量，本质就是一把**互斥锁**。它的规则很简单：

- 同一时刻最多只有一个线程持有这把锁
- 其他线程想拿锁，必须等持有者释放
- 如果同一个线程对同一把锁重复 lock，会**死锁**（自己等自己）

变量名叫 `latch_` 只是这个项目的命名风格，叫 `mtx_`、`mutex_` 也一样。

## std::scoped_lock — 自动管理锁的包装器

```cpp
std::scoped_lock lock{latch_};
```

这行代码的逐词拆解：

- `std::scoped_lock` — C++17 标准库里的**类模板**，专门用来包装互斥锁
- `lock` — 这个类的一个**对象**（变量名），叫 `guard` 也行
- `{latch_}` — 花括号初始化（列表初始化），等价于 `std::scoped_lock lock(latch_);`

完整翻译：**创建一个 `scoped_lock` 对象 `lock`，把 `latch_` 传进去让它管着。** 这个对象活着的时候锁就持有着，对象销毁时（出了作用域）自动释放锁。

### 为什么用它

不用 `scoped_lock` 也能手动操作：

```cpp
latch_.lock();     // 手动加锁
// ... 操作共享数据 ...
latch_.unlock();   // 手动解锁 — 容易忘！出异常更不会执行！
```

`scoped_lock` 的价值就是**保证解锁**——return 也好、抛异常也好，对象销毁时自动释放锁。相当于"进门自动锁门，出门自动开门"。

## 子函数不需要重复加锁

`fetch_page` 内部调用的 `find_victim_page` 和 `update_page` 不需要再加锁——而且绝对不能加。同一个线程里对 `std::mutex` 重复 lock 会**死锁**。这些子函数只在主方法内部调用，调用时锁已经持有了，直接操作共享数据即可。验证方式：`find_victim_page` 和 `update_page` 的函数体里确实没有 `scoped_lock`。

## latch_ 和 lock 不是固定搭配

`latch_` 只是变量名（本质是 `std::mutex`），`lock` 也只是 `scoped_lock` 对象的变量名。它们在这个项目里总是一起出现，是因为 `scoped_lock` 是管理互斥锁的最佳实践，但语法上并不强制绑定。

## 另一种锁：读写锁 RWLatch

在 Page 对象中还出现了第二种锁：

```cpp
// src/storage/rwlatch.h
class RWLatch {
    void WLock()   { mutex_.lock(); }         // 写锁：独占
    void WUnlock() { mutex_.unlock(); }
    void RLock()   { mutex_.lock_shared(); }  // 读锁：共享
    void RUnlock() { mutex_.unlock_shared(); }
private:
    std::shared_mutex mutex_;
};
```

对比普通互斥锁：

| | `std::mutex`（互斥锁） | `std::shared_mutex`（读写锁） |
|---|---|---|
| 读-读 | 互斥，排队 | **共享，并行** |
| 读-写 | 互斥 | 互斥 |
| 写-写 | 互斥 | 互斥 |
| 用在 | BufferPoolManager 的 `latch_` | Page 的 `rwlatch_`（`page.h:130`） |

Page 用读写锁的原因：一个页面可能被多个执行器同时读取（读-读不冲突），`RLock` 允许并行读；修改页面时必须独占（`WLock`）。这比普通互斥锁"不分读写一律互斥"更精细。`RWLatch` 会在索引层文档中重点涉及。

## 小结

- `std::mutex`：互斥锁，同一时刻只一个线程持有，是缓冲池 `latch_` 的类型
- `std::scoped_lock`：RAII 包装器，自动管理锁的生命周期，防止忘记解锁
- `std::shared_mutex`：读写锁，允许多读单写，用于 Page 级别的并发控制
- 子函数不重复加锁，`latch_` 与 `lock` 非固定搭配

上一节：[04c-buffer-pool-page-lifecycle.md](./04c-buffer-pool-page-lifecycle.md) | 下一节：[05a-disk-manager-overview.md](./05a-disk-manager-overview.md)
