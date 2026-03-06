# STL  
## std::vector  
### 1. vector 底层实现原理  
连续内存：底层是一个动态分配的数组（T* 指针）  
三个指针管理（现代实现）：  
_M_start：指向数组开头  
_M_finish：指向当前最后一个元素的下一个位置（size）  
_M_end_of_storage：指向已分配内存的末尾（capacity）  
内存布局（伪代码）：  
``` text
vector 对象（栈上）：
├── start 指针
├── finish 指针
└── end_of_storage 指针

实际数据（堆上）：
[元素0][元素1]...[元素size-1]  [未初始化空间]  ← capacity
```  
2. size vs capacity 对比表  
| 维度           | size（大小）                          | capacity（容量）                        |
|----------------|---------------------------------------|-----------------------------------------|
| 含义           | 当前实际元素个数                      | 已分配内存最多能容纳的元素个数          |
| 如何获取       | `v.size()`                            | `v.capacity()`                          |
| 修改方式       | `resize()`、`push_back` 等            | `reserve()`、`shrink_to_fit()`          |
| 内存占用       | 只影响已构造元素                      | 影响已申请的堆内存                      |
| 初始值         | 0                                     | 通常 0（首次 push_back 后分配）         |
| 关系           | `size() <= capacity()`                | `capacity() >= size()`                  |  
### 3. 扩容机制  
触发时机：size() == capacity() 时再插入元素  
增长因子（Growth Factor）：  
GCC / Clang：2倍（最常见）  
MSVC：1.5倍  
苹果 libc++：2倍  
  
扩容过程（代价很高）：  
申请新内存（通常是旧 capacity 的 1.5~2 倍）  
把旧元素移动/拷贝到新内存（C++11 后优先移动）  
释放旧内存  
更新三个指针  
  
时间复杂度：单次扩容 O(N)，但均摊 O(1)（摊销分析）  
推荐做法：提前 reserve() 避免频繁扩容！  
### 4. 时间复杂度完整表  
| 操作                     | 时间复杂度       | 说明                                      |
|--------------------------|------------------|-------------------------------------------|
| `operator[]` / `at()`    | O(1)             | 随机访问（最快）                          |
| `push_back()`            | 均摊 O(1)        | 末尾插入（扩容时 O(N)）                   |
| `pop_back()`             | O(1)             | 末尾删除                                  |
| `emplace_back()`         | 均摊 O(1)        | 原地构造，比 push_back 更快               |
| `insert()` / `erase()`   | O(N)             | 中间插入/删除（要移动后面所有元素）       |
| `resize()`               | O(N)             | 改变大小（可能构造/析构元素）             |
| `reserve()`              | O(N)             | 预分配容量（只扩容，不构造元素）          |
| `shrink_to_fit()`        | O(N)             | 释放多余容量（C++11）                     |
| `clear()`                | O(N)             | 只析构元素，不释放内存                    |  
### 5. 迭代器失效规则  
vector 迭代器失效的核心原因：内存重新分配 或 元素移动。  
| 操作                          | 迭代器是否失效                          | 说明                                      |
|-------------------------------|-----------------------------------------|-------------------------------------------|
| `push_back()` / `emplace_back()` | 可能失效（扩容时**全部失效**）         | 一旦发生 reallocation，所有迭代器失效     |
| `insert()`（任意位置）        | 插入点及**之后**的所有迭代器失效       | 前面迭代器可能仍有效                      |
| `erase()`（任意位置）         | 被删除元素及**之后**的所有迭代器失效   | 前面迭代器可能仍有效                      |
| `resize()`（扩大容量）        | 可能失效（扩容时全部失效）             | 缩小容量时不失效                          |
| `reserve()`                   | **不失效**                              | 只扩容，不移动元素（安全操作）            |
| `clear()`                     | **不失效**（但不能解引用）             | 迭代器仍指向原内存，但元素已被析构        |
| `pop_back()`                  | **不失效**                              | 只影响最后一个元素                        |
| `shrink_to_fit()`             | **可能失效**                            | 可能触发 reallocation                     |  
黄金规则：  
只要发生 reallocation（扩容），所有迭代器、指针、引用全部失效！  
reserve() 是唯一“安全扩容”操作。  
### 6. push_back vs emplace_back  
``` cpp
v.push_back(MyClass(10, "hello"));     // 先构造临时对象，再移动/拷贝
v.emplace_back(10, "hello");           // 原地直接构造（推荐！）
```
emplace_back 优势：  
避免临时对象构造 + 移动  
支持完美转发  
C++11 后强烈推荐使用  
### 7. 现代 C++ 最佳实践  
``` text
1. **提前 reserve()**：已知元素数量时一定要 reserve，避免多次扩容
2. **优先 emplace_back**：代替 push_back
3. **用 shrink_to_fit()** 释放多余内存（注意：可能触发 reallocation）
4. **移动语义友好**：vector 内部自动使用移动构造（C++11 后）
5. **避免在循环中频繁 insert/erase**：考虑用 list 或先收集再一次性插入
6. **自定义分配器**（C++17+）：可使用 `std::pmr::vector` 实现内存池
```  
### 8. vector vs list vs deque 对比表  
| 维度           | vector                  | list                    | deque                     |
|----------------|-------------------------|-------------------------|---------------------------|
| 底层结构       | 连续动态数组            | 双向链表                | 分段连续数组（中控器）    |
| 随机访问       | O(1)                    | O(N)                    | O(1)                      |
| 头尾插入       | 尾部快，头部慢          | 两端 O(1)               | 两端 O(1)                 |
| 中间插入/删除  | O(N)                    | O(1)（已有迭代器）      | O(N)                      |
| 内存连续性     | 是                      | 否                      | 逻辑连续，物理分段        |
| 缓存友好       | 极好                    | 差                      | 较好                      |
| 迭代器失效     | 频繁                    | 极少                    | 较少                      |
| 推荐场景       | 随机访问、尾部操作      | 频繁中间插入删除        | 双端队列（队列、栈）      |
## map/unordered_map  
### 1. 基础概念与头文件  
std::map  
头文件：<map>  
命名空间：std  
特点：有序关联容器，key 唯一，自动按 key 排序（默认升序）  
  
std::unordered_map  
头文件：<unordered_map>  
命名空间：std  
特点：无序关联容器，key 唯一，基于哈希实现  
  
两者都是关联容器（Associative Container），存储 key-value 对（std::pair<const Key, T>）。  
### 2. 底层实现  
| 项目              | std::map                              | std::unordered_map                          |
|-------------------|---------------------------------------|---------------------------------------------|
| 底层数据结构      | **红黑树**（RB-Tree）                 | **哈希表**（Hash Table）+ 链表/红黑树（C++11起桶内可能退化） |
| 元素存储顺序      | 按 key 严格有序                       | 无序（取决于 hash 值）                      |
| 迭代器类型        | 双向迭代器（Bidirectional）           | 前向迭代器（Forward）                       |
| 空间复杂度        | O(n)（每个节点 3 个指针 + 颜色）      | O(n) + 负载因子（通常 1.0）                 |

红黑树：自平衡二叉查找树，插入/删除/查找都是 O(log n)，保证最坏情况性能。  
哈希表：桶数组 + 链表（或 C++11 后可为红黑树），平均 O(1)，最坏 O(n)（哈希冲突严重时）。  
### 3. 时间复杂度对比  
| 操作                  | map（最坏/平均）     | unordered_map（平均 / 最坏） |
|-----------------------|----------------------|------------------------------|
| insert                | O(log n)            | O(1) / O(n)                 |
| erase                 | O(log n)            | O(1) / O(n)                 |
| find / [] / at        | O(log n)            | O(1) / O(n)                 |
| lower_bound / upper_bound | O(log n)        | 不支持（无序）              |
| begin / end           | O(1)                | O(1)（但遍历是 O(n + buckets)）|
| 遍历所有元素          | O(n)                | O(n)（但常数因子更大）      |  
### 4. 构造函数与初始化  
``` cpp
std::map<int, std::string> m1;                    // 默认
std::map<int, std::string, std::greater<int>> m2; // 自定义比较器（降序）
std::unordered_map<std::string, int> um1;         // 默认 hash

// C++11 初始化列表
std::map<int, std::string> m3 = {{1,"a"}, {2,"b"}};
```  
### 5. 关键成员函数  
``` cpp
// 插入
m.insert({key, value});           // 返回 pair<iterator, bool>
m.emplace(key, value);            // C++11，更高效（原地构造）
m[key] = value;                   // [] 会默认构造（map 特有危险操作）

// 查找
auto it = m.find(key);            // 找不到返回 end()
if (it != m.end()) ...
m.count(key);                     // 0 或 1（map 里永远是 0/1）
m.contains(key);                  // C++20

// 删除
m.erase(key);                     // 返回删除个数
m.erase(it);                      // 迭代器删除（推荐）
m.erase(first, last);

// 范围查询（仅 map 有）
auto [l, r] = m.equal_range(key);
```
[] vs at() vs find()  
[]：key 不存在时插入默认构造的 value → 可能改变容器大小  
at()：key 不存在抛 std::out_of_range  
find()：最安全，不会修改容器  
### 6. 迭代器有效性  
map：红黑树节点，插入/删除仅使指向被删元素的迭代器失效，其他全部有效。  
unordered_map：哈希表 rehash 时所有迭代器都失效（负载因子超过 max_load_factor，默认 1.0）。  
reserve(n) / rehash(n) 会触发 rehash → 迭代器全失效  
### 7. 自定义 Key 类型  
map（需要严格弱序）：  
``` cpp
struct Key {
    int a, b;
};
struct Cmp {
    bool operator()(const Key& x, const Key& y) const {
        return x.a != y.a ? x.a < y.a : x.b < y.b;
    }
};
std::map<Key, int, Cmp> m;
```
unordered_map（需要 hash + equal）：  
``` cpp
struct Key {
    std::string s;
    int i;
};
namespace std {
    template<> struct hash<Key> {
        size_t operator()(const Key& k) const {
            return std::hash<std::string>()(k.s) ^ std::hash<int>()(k.i);
        }
    };
}
bool operator==(const Key& x, const Key& y) { ... }

std::unordered_map<Key, int> um;
```
C++11 推荐写法：用 std::hash 组合 + std::equal_to。  
### 8. 内存与性能细节
map：每个节点约 32~40 字节开销（指针+颜色）
unordered_map：桶数组 + 负载因子，通常比 map 多占 20%~50% 内存
um.reserve(1000) 可大幅减少 rehash

### 9. 多重映射版本（顺便记住）  
std::multimap / std::unordered_multimap：允许 key 重复  
区别：insert 永远成功，返回 iterator；没有 operator[]  
### 10. 线程安全  
两者都不是线程安全的。并发读写必须加锁  
### 11. C++17/20/23 新特性  
C++17：insert_or_assign、try_emplace  
C++20：contains()、erase_if  
C++23：flat_map / flat_multimap  
