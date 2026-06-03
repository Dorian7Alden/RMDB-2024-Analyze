# 11. 记录层 API 速查

快速回顾记录层每个类的核心接口。

## RmManager

文件生命周期管理。`src/record/rm_manager.h:20`

```cpp
void create_file(const std::string& filename, int record_size)  // 创建表数据文件并初始化文件头
void destroy_file(const std::string& filename)                   // 删除表数据文件
std::unique_ptr<RmFileHandle> open_file(const std::string&)     // 打开文件，返回 RmFileHandle
void close_file(const RmFileHandle* file_handle)                 // 关闭文件：写文件头→刷脏页→清页表→关文件
void flush_file(const RmFileHandle* file_handle)                 // checkpoint 时刷盘
```

## RmFileHandle

单表记录操作。`src/record/rm_file_handle.h:55`

```cpp
// 记录 CRUD
std::unique_ptr<RmRecord> get_record(const Rid& rid, Context*)  // 读取指定 Rid 的记录，返回独立副本
Rid insert_record(char* buf, Context*)                           // 自动找空位插入，返回 Rid
void insert_record(const Rid& rid, char* buf)                    // 插入到指定 Rid 位置
void delete_record(const Rid& rid, Context*)                     // 删除指定记录
void update_record(const Rid& rid, char* buf, Context*)         // 覆盖更新指定记录（定长直接覆盖）

// 页面访问
RmPageHandle fetch_page_handle(int page_no)                      // 从缓冲池获取指定页面
RmPageHandle create_page_handle()                                // 获取有空位的页面（无则新建）
RmPageHandle create_new_page_handle()                            // 强制分配全新页面

// 空闲链表
void release_page_handle(RmPageHandle&)                          // 页面从满变不满时加回链表

// 查询
bool is_record(const Rid& rid)                                   // 通过 Bitmap 判断指定位置是否有记录
```

## RmPageHandle

页面封装，解析原始字节为结构化指针。`src/record/rm_file_handle.h:25`

```cpp
char* get_slot(int slot_no)                                      // 返回指定槽位的首地址

// 成员
const RmFileHdr* file_hdr   // 所属文件的文件头（只读）
Page* page                   // 缓冲池中的 Page 对象指针
RmPageHdr* page_hdr          // 页内 RmPageHdr 区域指针
char* bitmap                 // 页内 Bitmap 区域指针
char* slots                  // 页内记录数据区指针
```

## Bitmap

位图工具类，全静态方法。`src/record/bitmap.h`

```cpp
static void init(char* bm, int size)                             // 全部清零
static void set(char* bm, int pos)                               // 第 pos 位置为 1（标记占用）
static void reset(char* bm, int pos)                             // 第 pos 位置为 0（标记空闲）
static bool is_set(const char* bm, int pos)                      // 判断第 pos 位是否为 1
static int first_bit(bool bit, const char* bm, int max_n)        // 找第一个 0 或 1 的位
static int next_bit(bool bit, const char* bm, int max_n, int)   // 从指定位置后找下一个 0 或 1 的位
```

## RmScan

全表顺序扫描器，继承 RecScan。`src/record/rm_scan.h:18`

```cpp
void next()                                                       // 移动到下一条已占用的记录
bool is_end()                                                     // 是否扫描完毕
Rid rid()                                                         // 获取当前记录位置
std::unique_ptr<RmRecord> get_record()                            // 从缓存页面直接读取当前记录
```

## 数据结构

```cpp
// Rid — 记录的唯一地址  src/defs.h:33
struct Rid { int page_no; int slot_no; };

// RmRecord — 记录内容  src/record/rm_defs.h:40
struct RmRecord { char* data; int size; bool allocated_; };

// RmFileHdr — 文件头（第 0 页） src/record/rm_defs.h:24
struct RmFileHdr { int record_size; int num_pages; int num_records_per_page;
                   std::atomic<int> first_free_page_no; int bitmap_size; };

// RmPageHdr — 页头（每个数据页开头） src/record/rm_defs.h:34
struct RmPageHdr { int next_free_page_no; int num_records; };
```

上一节：[10-record-frame-vs-reference.md](./10-record-frame-vs-reference.md) | 下一节：[12-record-layer-summary.md](./12-record-layer-summary.md)
