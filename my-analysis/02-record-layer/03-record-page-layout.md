# 03. 数据页内部布局

搞清楚每个数据页里面长什么样——空间怎么划分、每块占多大、怎么定位到具体记录。

## Page 的原始结构

从第 1 章我们知道，每个 `Page` 对象底层是一个 4096 字节的数组（`PAGE_SIZE`）。记录层在这个数组上定义了自己的格式：

```mermaid
flowchart LR
    subgraph page["Page data 4096 字节"]
        style page fill:#f3f4f6,stroke:#6b7280,color:#374151
        direction LR
        A["LSN<br/>Page通用头部"]
        B["RmPageHdr页头<br/>8 字节"]
        C["Bitmap位图<br/>bitmap_size 字节"]
        D["Slot 0"]
        E["Slot 1"]
        F["..."]
        G["Slot N-1"]
    end

    subgraph note1["便利贴 通用头"]
        style note1 fill:#fff9c4,stroke:#f9a825,stroke-dasharray:4
        N1["每个 Page 开头有一个<br/>数据库通用的页头<br/>存储 LSN 等通用信息<br/>共OFFSET_PAGE_HDR字节"]
    end

    subgraph note2["便利贴 记录层特有"]
        style note2 fill:#fff9c4,stroke:#f9a825,stroke-dasharray:4
        N2["通用头之后的部分<br/>由记录层自行定义格式"]
    end

    note1 -.-> A
    note2 -.-> B
```

**三段式布局**（通用 page header 之后的部分）：

| 段 | 内容 | 大小 |
|----|------|------|
| RmPageHdr | 页级元信息（num_records, next_free_page_no） | `sizeof(RmPageHdr)` = 8 字节 |
| Bitmap | 位图，标记每个槽是否被占用 | `bitmap_size` 字节 |
| Slots | 定长记录槽位数组 | `num_records_per_page × record_size` 字节 |

## Bitmap 位槽对应关系

bitmap 的每一位对应一个槽位：bit = 1 表示该槽已占用，bit = 0 表示空闲。bitmap 用 `char[]` 数组实现，每字节（8 位）对应 8 个槽。

举个例子：假设 `num_records_per_page = 10`，需要 `ceil(10/8) = 2` 字节的 bitmap。两个字节共 16 位，只用前 10 位标记 slot 0~9，后 6 位闲置：

```
bitmap 第 0 字节              bitmap 第 1 字节
┌─────────────────────────┐ ┌─────────────────────────┐
│ b7 b6 b5 b4 b3 b2 b1 b0 │ │ b7 b6 b5 b4 b3 b2 b1 b0 │
│ │  │  │  │  │  │  │  │  │ │ │  │  │  │  │  │  │  │  │
│ ↓  ↓  ↓  ↓  ↓  ↓  ↓  ↓  │ │ ↓  ↓  ↓  ↓  ↓  ↓  ↓  ↓  │
│s0 s1 s2 s3 s4 s5 s6 s7  │ │s8 s9  -  -  -  -  -  -  │
└─────────────────────────┘ └─────────────────────────┘
                                           ↑ 闲置位（始终为 0）
```

如果插入了 slot 0 和 slot 3，bitmap 第 0 字节变为：

```
b7  b6  b5  b4  b3  b2  b1  b0
 1   0   0   1   0   0   0   0   = 0x90
 ↑   ↑   ↑   ↑   ↑   ↑   ↑   ↑
s0  s1  s2  s3  s4  s5  s6  s7
占           占
用           用
```

`s0=1, s3=1`，其余为 0。高位 b7 对应 slot 0，低位 b0 对应 slot 7。

## 三段大小的计算过程

以 student 表为例，假设 `record_size = 32` 字节。

### 第一步：算每页能放几条记录

计算公式来自 `RmManager::create_file()`（`src/record/rm_manager.h:48`）：

```
num_records_per_page = (BITMAP_WIDTH × (PAGE_SIZE - 1 - sizeof(RmFileHdr)) + 1)
                     / (1 + record_size × BITMAP_WIDTH)
```

这个公式一步到位，不好理解。下面逐步推导。

**第 1 步：写出空间约束方程**

一页中可用于记录层的空间是"Page 总大小 − 通用头"：

```
可用空间 = PAGE_SIZE - OFFSET_PAGE_HDR
         = 4096 - 4 = 4092 字节
```

这 4092 字节要装三样东西：`RmPageHdr` + `bitmap` + `slots`。设槽位数为 `n`：

```
sizeof(RmPageHdr) + bitmap_size + n × record_size ≤ 4092
                  8 + bitmap_size + n × 32        ≤ 4092
                        bitmap_size + n × 32      ≤ 4084   ... ①
```

**第 2 步：把 bitmap_size 用 n 精确表示**

`bitmap_size = ceil(n / 8)`。向上取整在整数运算中有个标准技巧：**`ceil(n / 8) = (n + 7) / 8`**（整数除法）。

为什么成立？设 `n = 8k + r`（`0 ≤ r < 8`）：
- 若 r = 0（整除）：`ceil(8k/8) = k`，而 `(8k+7)/8 = k`（因为 7/8 在整数除法中舍去）✓
- 若 r > 0（有余数）：`ceil(n/8) = k+1`，而 `(n+7)/8 = (8k+r+7)/8`，由于 `r ≥ 1, r+7 ≥ 8`，`(8k+r+7)/8 = k+1` ✓

因此 `bitmap_size = (n + 7) / 8`，精确成立。

代入 ①：

```
(n + 7) / 8 + n × 32 ≤ 4084
```

**第 3 步：去分母，乘以 8**

```
n + 7 + n × 256 ≤ 4084 × 8
n + 7 + n × 256 ≤ 32672
n × 257 + 7 ≤ 32672
n × 257 ≤ 32665
```

**第 4 步：解出 n**

```
n ≤ 32665 / 257
n ≤ 127.10...
```

向下取整：**`n = 127`**。即每页最多存 127 条记录。

**第 5 步：验证**

```
bitmap_size = ceil(127 / 8) = ceil(15.875) = 16 字节
slots = 127 × 32 = 4064 字节
总计 = 8 + 16 + 4064 = 4088 ≤ 4092 ✓
```

如果 n = 128：
```
bitmap_size = ceil(128 / 8) = 16 字节
slots = 128 × 32 = 4096 字节
总计 = 8 + 16 + 4096 = 4120 > 4092 ✗  放不下！
```

**通用公式与推导的对应关系**

上面逐步推导的结果等价于代码中的公式：

```
推导:  n = 4084 × BITMAP_WIDTH / (1 + record_size × BITMAP_WIDTH)
代码:  n = (BITMAP_WIDTH × (PAGE_SIZE - 1 - sizeof(RmFileHdr)) + 1)
         / (1 + record_size × BITMAP_WIDTH)
```

其中：
- `PAGE_SIZE - 1 - sizeof(RmFileHdr)` ≈ 可用空间（`sizeof(RmFileHdr)` 约等于 `OFFSET_PAGE_HDR + sizeof(RmPageHdr)`，代码中用它近似表示页面总开销）
- `+ 1` 是整数除法向上取整的修正项（保证 `ceil` 效果）
- `BITMAP_WIDTH = 8`

**实际计算结果**：student 表 `record_size = 28`（4+20+4），代入公式得到 `num_records_per_page ≈ 140`。

### 第二步：算 bitmap 大小

```
bitmap_size = ceil(num_records_per_page / 8)
            = (num_records_per_page + BITMAP_WIDTH - 1) / BITMAP_WIDTH
```

`src/record/rm_manager.h:51`

## RmPageHandle：页面的读写指针

`RmPageHandle` 是一个轻量级的包装结构体，把页面上三段区域解析成对应指针，方便后续直接读写。

### 结构体定义

`src/record/rm_file_handle.h:25`

```cpp
struct RmPageHandle {
  const RmFileHdr* file_hdr;  // 指向所属文件的文件头（只读）
  Page* page;                 // 指向缓冲池中的页面对象
  RmPageHdr* page_hdr;        // 指向页内的 RmPageHdr 区域
  char* bitmap;               // 指向页内的 Bitmap 区域
  char* slots;                // 指向页内的 Slots 区域
};
```

### `page` 指向的是什么？

`page` 是**当前操作的那一页**，不是"所有页的数组"。一张表的数据文件有很多页，每次 `fetch_page_handle(page_no)` 只从缓冲池取出**一个** Page 对象：

```
RmPageHandle ph = file_handle->fetch_page_handle(3);
// ph.page  → 指向缓冲池中 {fd:3, page_no:3} 这一个 Page
// ph.slots → 指向这一页的记录数据区
```

用完 `unpin_page`，再取下一页时生成新的 RmPageHandle 指向另一个 Page。

### `page->get_data()` 返回什么？

`get_data()` 返回 `Page` 对象内部 `data_[PAGE_SIZE]` 数组的首地址（`page.h:81`），即这一页数据在内存中的起始位置。构造函数的偏移计算都是从这个首地址开始的。

### `reinterpret_cast` 是什么？

`Page` 底层是原始字节数组 `char data_[4096]`，编译器只知道它是 `char*`。但我们知道偏移 4~11 存的是 `RmPageHdr` 结构体——怎么告诉编译器？

`reinterpret_cast<RmPageHdr*>(地址)` 的意思是：**"把这块内存强行解释为 RmPageHdr 类型的指针"**。之后就能用 `page_hdr->num_records` 直接读写字段，不需要手动逐字节解析。

打比方：一块内存像一张白纸，`reinterpret_cast` 告诉编译器"请按 RmPageHdr 的格式来读"，编译器就知道哪个偏移是 `num_records`、哪个是 `next_free_page_no`。

```mermaid
flowchart TB
    subgraph raw["Page data_ 原始字节"]
        direction TB
        B0["0x00"]
        B1["0x01"]
        B2["0x02"]
        B3["0x03"]
        B4["0x04"]
        B5["0x05"]
        BN["..."]
    end

    subgraph hdr["reinterpret_cast 后"]
        direction TB
        H0["num_records int<br/>4 字节 偏移 4-7"]
        H1["next_free_page_no int<br/>4 字节 偏移 8-11"]
    end

    raw -->|"重新解释"| hdr
```

### 构造函数：三个指针的计算

现在有了前置知识，看构造函数就清楚了：

```cpp
RmPageHandle(const RmFileHdr* fhdr_, Page* page_)
    : file_hdr(fhdr_), page(page_) {
  page_hdr = reinterpret_cast<RmPageHdr*>(
      page->get_data() + page->OFFSET_PAGE_HDR);
  bitmap = page->get_data() + sizeof(RmPageHdr)
           + page->OFFSET_PAGE_HDR;
  slots = bitmap + file_hdr->bitmap_size;
}
```

以一个具体数值来追踪——假设内存地址从 `0x1000` 开始：

```
已知:
  page->get_data()        → 0x1000   (data_ 数组首地址)
  page->OFFSET_PAGE_HDR   = 4        (LSN 占 4 字节)
  sizeof(RmPageHdr)       = 8
  file_hdr->bitmap_size   从文件头获取
```

**page_hdr** — 跳过通用头（LSN），指向 RmPageHdr：

```
reinterpret_cast<RmPageHdr*>(0x1000 + 4)
= reinterpret_cast<RmPageHdr*>(0x1004)
```

**bitmap** — 跳过 LSN 再跳过 RmPageHdr：

```
0x1000 + 4 + 8 = 0x100C
```

> 源码中写的是 `sizeof(RmPageHdr) + page->OFFSET_PAGE_HDR`（先 8 后 4），与物理布局顺序（先 LSN 后 RmPageHdr）不一致。加法满足交换律所以结果正确，但写成 `OFFSET_PAGE_HDR + sizeof(RmPageHdr)` 从左到右对应物理从上到下，阅读更顺畅。

**slots** — 在 bitmap 之后再跳过 bitmap_size：

```
bitmap + file_hdr->bitmap_size
```

三个指针的偏移关系：

```mermaid
flowchart TD
    subgraph layout["Page 内存布局"]
        style layout fill:#f3f4f6,stroke:#6b7280,color:#374151
        direction LR
        GH["通用头<br/>OFFSET_PAGE_HDR"]
        PH["RmPageHdr<br/>sizeof RmPageHdr"]
        BM["Bitmap<br/>bitmap_size"]
        SL["Slots 记录区"]
    end

    subgraph ptrs["三个指针"]
        style ptrs fill:#d1fae5,stroke:#10b981,color:#065f46
        P1["page_hdr = 通用头末尾"]
        P2["bitmap = page_hdr 末尾"]
        P3["slots = bitmap 末尾"]
    end

    PH -->|"指向"| P1
    BM -->|"指向"| P2
    SL -->|"指向"| P3
```

### 为什么 bitmap 不从 page_hdr 推算？

bitmap 紧跟在 RmPageHdr 之后，写成这样更直观：

```cpp
bitmap = reinterpret_cast<char*>(page_hdr) + sizeof(RmPageHdr);
```

源码没有这样做，而是每次都从 `get_data()` 加固定偏移。两种写法结果一样，但从 `page_hdr` 推算更能反映物理上"bitmap 紧跟在 RmPageHdr 之后"的事实。

> 我觉得就是 bullshit ，明明直接计算偏移量更符合直觉，相对位置更清晰，这么写完全就是反人类。AI 别强行解释啊喂！

## get_slot：定位具体记录

`src/record/rm_file_handle.h:46`

```cpp
inline char* get_slot(int slot_no) const {
  return slots + slot_no * file_hdr->record_size;
}
```

给定槽位号 `slot_no`，计算该记录在 slots 数组中的地址偏移：

```
偏移 = slots 首地址 + slot_no × record_size
```

因为每条记录是定长的，所以可以直接用乘法算出任意槽位的地址。这是定长记录的核心优势。

举例：student 表 `record_size = 32`，要找 slot_no = 3 的记录：

```
地址 = slots + 3 × 32 = slots + 96
```

也就是从 slots 开头往后数第 96 个字节开始，读 32 字节就是 slot 3 的记录。

## 源码对应

| 内容 | 文件 | 行号 |
|------|------|------|
| RmPageHandle 定义 | `src/record/rm_file_handle.h` | 25-51 |
| RmPageHandle 构造函数 | `src/record/rm_file_handle.h` | 37-43 |
| get_slot | `src/record/rm_file_handle.h` | 46-50 |
| num_records_per_page 计算 | `src/record/rm_manager.h` | 48-50 |
| bitmap_size 计算 | `src/record/rm_manager.h` | 51-52 |

上一节：[02. 数据结构](./02-record-data-structures.md) | 下一节：[04. Bitmap 位图](./04-record-bitmap.md)
