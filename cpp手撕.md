# C++ 高频手撕题速记版



---

# 1. LRU

核心结构：
- `unordered_map<int, Node*> key_to_node`
- 单 `dummy` 节点
- 循环双向链表
- 头部是最近使用
- 尾部是最久未使用
- 访问成功就移动到头部

---

## 1.1 标准版代码

```cpp
#include <unordered_map>
using namespace std;

struct Node {
    int key;
    int val;
    Node* prev;
    Node* next;
    Node(int k=0, int v=0) : key(k), val(v), prev(nullptr), next(nullptr) {}
};

class LRUCache {
    unordered_map<int, Node*> key_to_node;
    int capacity;
    Node* dummy;

    void remove(Node* x) {
        x->prev->next = x->next;
        x->next->prev = x->prev;
    }

    void push_front(Node* x) {
        x->next = dummy->next;
        x->prev = dummy;
        x->prev->next = x;
        x->next->prev = x;
    }

    Node* get_node(int key) {
        auto it = key_to_node.find(key);
        if (it == key_to_node.end())
            return nullptr;
        Node* node = it->second;
        remove(node);
        push_front(node);
        return node;
    }

public:
    LRUCache(int capacity) : capacity(capacity), dummy(new Node()) {
        dummy->next = dummy;
        dummy->prev = dummy;
    }

    int get(int key) {
        Node* node = get_node(key);
        if (node)
            return node->val;
        return -1;
    }

    void put(int key, int value) {
        Node* node = get_node(key);
        if (node)
            node->val = value;
        else {
            key_to_node[key] = node = new Node(key, value);
            push_front(node);
        }

        if (key_to_node.size() > capacity) {
            Node* backnode = dummy->prev;
            key_to_node.erase(backnode->key);
            remove(backnode);
            delete backnode;
        }
    }

    ~LRUCache() {
        Node* cur = dummy->next;
        while (cur != dummy) {
            Node* nxt = cur->next;
            delete cur;
            cur = nxt;
        }
        delete dummy;
    }
};
```

---

## 1.2 这题到底在考什么

LRU 不只是考你会不会背模板，主要考四件事：

1. **能不能快速选对数据结构**
   - 查找要快：哈希表
   - 移动、删除要快：双向链表

2. **能不能把“最近使用”变成具体操作**
   - 被访问的节点移动到头部
   - 淘汰尾部节点

3. **边界处理稳不稳**
   - 空链表
   - 只有一个节点
   - 更新已有 key
   - 容量满了后淘汰

4. **代码是否够干净**
   - 核心操作能不能拆出来复用
   - 有没有明显内存问题

---

## 1.3 这版为什么好背

你实际上只要记住三件事：

```cpp
remove(x)
push_front(x)
get_node(key)
```

再加一个淘汰尾节点：

```cpp
Node* backnode = dummy->prev;
```

然后：
- `get`：调用 `get_node`
- `put`：
  - 有旧节点：更新值
  - 没有：新建并插到头部
  - 超容量：删尾

---

## 1.4 记忆口诀

**“查到了就提头，没查到就插头，超容量就删尾。”**

或者记成：

**哈希找，链表挪；新放头，旧提头；满了删尾。**

---

## 1.5 面试解释模板

可以直接这么说：

> 我习惯用哈希表加循环双向链表实现 LRU。  
> 哈希表负责 O(1) 找节点，链表负责 O(1) 删除和插入。  
> 我这里用了一个 dummy 节点，把链表做成循环结构，这样头尾统一，不需要判断空链表、头节点、尾节点，代码更紧凑。  
> 最近访问的节点放头部，最久未使用的节点在尾部，也就是 `dummy->prev`。

---

## 1.6 这版常见追问

### Q1：为什么 `get_node` 要顺便把节点移到头部？
因为 LRU 的“最近使用”语义要求访问成功后更新新旧顺序。

### Q2：为什么用双向链表？
因为删除当前节点需要 O(1) 拿到前驱和后继。

### Q3：为什么用单 dummy？
因为循环链表下：
- 头节点：`dummy->next`
- 尾节点：`dummy->prev`
- 空链表：`dummy->next == dummy`
统一好写。

### Q4：`capacity == 0` 怎么办？
你这版也能正常工作，只是插入后会立刻淘汰。

---

## 1.7 易错点

### 1）更新已有 key 时，只改值，不挪位置
这是错的。访问或更新已有 key，都应该变成最近使用。

### 2）淘汰尾节点后忘记从 map 删
```cpp
key_to_node.erase(backnode->key);
```

### 3）删除节点时顺序乱了
建议始终先：
- 改前驱的 `next`
- 再改后继的 `prev`

### 4）写完不释放节点
LeetCode 不一定卡，但面试官会看内存意识。

---

# 2. LRU 变种

---

## 2.1 带过期时间 TTL-LRU

这个是最常见的 LRU 变种之一。

思路：
- 节点多一个 `expire`
- `get_node` 里顺便检查是否过期
- 过期就删掉并返回空
- 没过期再移到头部

---

## 2.2 TTL-LRU 代码（沿用你的风格）

```cpp
#include <unordered_map>
#include <chrono>
using namespace std;

struct Node {
    int key;
    int val;
    long long expire;
    Node* prev;
    Node* next;
    Node(int k=0, int v=0, long long e=0)
        : key(k), val(v), expire(e), prev(nullptr), next(nullptr) {}
};

class TTL_LRUCache {
    unordered_map<int, Node*> key_to_node;
    int capacity;
    Node* dummy;

    long long now_ms() {
        return chrono::duration_cast<chrono::milliseconds>(
            chrono::steady_clock::now().time_since_epoch()
        ).count();
    }

    bool is_expired(Node* x) {
        return x->expire <= now_ms();
    }

    void remove(Node* x) {
        x->prev->next = x->next;
        x->next->prev = x->prev;
    }

    void push_front(Node* x) {
        x->next = dummy->next;
        x->prev = dummy;
        x->prev->next = x;
        x->next->prev = x;
    }

    void erase_node(Node* x) {
        key_to_node.erase(x->key);
        remove(x);
        delete x;
    }

    Node* get_node(int key) {
        auto it = key_to_node.find(key);
        if (it == key_to_node.end())
            return nullptr;

        Node* node = it->second;
        if (is_expired(node)) {
            erase_node(node);
            return nullptr;
        }

        remove(node);
        push_front(node);
        return node;
    }

public:
    TTL_LRUCache(int capacity) : capacity(capacity), dummy(new Node()) {
        dummy->next = dummy;
        dummy->prev = dummy;
    }

    int get(int key) {
        Node* node = get_node(key);
        if (node)
            return node->val;
        return -1;
    }

    void put(int key, int value, long long ttl_ms) {
        long long expire = now_ms() + ttl_ms;
        Node* node = get_node(key);

        if (node) {
            node->val = value;
            node->expire = expire;
        } else {
            key_to_node[key] = node = new Node(key, value, expire);
            push_front(node);
        }

        if (key_to_node.size() > capacity) {
            Node* backnode = dummy->prev;
            key_to_node.erase(backnode->key);
            remove(backnode);
            delete backnode;
        }
    }

    ~TTL_LRUCache() {
        Node* cur = dummy->next;
        while (cur != dummy) {
            Node* nxt = cur->next;
            delete cur;
            cur = nxt;
        }
        delete dummy;
    }
};
```

---

## 2.3 TTL-LRU 到底比普通 LRU 多了什么

你可以直接记成：

**普通 LRU = key + val**  
**TTL-LRU = key + val + expire**

普通 LRU 只看“新旧”  
TTL-LRU 还要再看“有没有过期”

所以 `get_node` 的逻辑变成：

1. 先看 key 在不在
2. 在的话先看过期没
3. 过期就删掉
4. 没过期再移到头部

---

## 2.4 TTL-LRU 的面试回答模板

> 如果 LRU 带过期时间，我会在节点里加一个 `expire` 字段。  
> 访问时在 `get_node` 中判断是否过期，过期就直接从哈希表和链表中删除。  
> 这是惰性删除的做法，简单而且适合面试。进一步可以用小顶堆或后台线程做更高效的主动清理。

---

## 2.5 惰性删除是什么意思

意思是：
- 不是一到过期时间就立刻删除
- 而是在访问到这个 key 时再顺手删掉

优点：
- 实现简单
- 面试很适合

缺点：
- 如果很多 key 过期了但一直没人访问，它们会暂时留在缓存里

---

## 2.6 主动清理怎么答

你可以口头回答：

### 朴素版
定期遍历链表，把过期节点删掉。

### 更优版
用一个按过期时间排序的小顶堆，快速找到最早过期的节点。

---

## 2.7 线程安全 LRU 怎么答

最基础写法就是在 `get/put` 外加一把锁。

```cpp
#include <mutex>
using namespace std;

mutex mtx;

int get(int key) {
    lock_guard<mutex> lk(mtx);
    ...
}

void put(int key, int value) {
    lock_guard<mutex> lk(mtx);
    ...
}
```

进一步可以说：
- 一把大锁简单，但并发度低
- 可以做分片 LRU
- 每个 shard 各有自己的 map、链表和锁

---

## 2.8 这部分记忆口诀

**普通 LRU 管容量，TTL-LRU 多管时间。**  
**访问先判死活，活着再提到头。**

---

# 3. 手撕 String

---

## 3.1 这题在考什么

这题本质是在考你对 **资源管理** 的理解：

- 堆内存是谁申请的
- 谁负责释放
- 拷贝时是浅拷贝还是深拷贝
- 赋值时会不会内存泄漏
- 移动语义知不知道

这题不是在考“字符串算法”，而是在考 **C++ 类设计基本功**。

---

## 3.2 标准版

```cpp
#include <cstring>
using namespace std;

class MyString {
    char* data_;
    size_t size_;

public:
    MyString() : data_(new char[1]), size_(0) {
        data_[0] = '\0';
    }

    MyString(const char* s) {
        if (!s) {
            data_ = new char[1];
            data_[0] = '\0';
            size_ = 0;
        } else {
            size_ = strlen(s);
            data_ = new char[size_ + 1];
            memcpy(data_, s, size_ + 1);
        }
    }

    MyString(const MyString& other) {
        size_ = other.size_;
        data_ = new char[size_ + 1];
        memcpy(data_, other.data_, size_ + 1);
    }

    MyString& operator=(const MyString& other) {
        if (this == &other) return *this;
        char* new_data = new char[other.size_ + 1];
        memcpy(new_data, other.data_, other.size_ + 1);
        delete[] data_;
        data_ = new_data;
        size_ = other.size_;
        return *this;
    }

    MyString(MyString&& other) noexcept {
        data_ = other.data_;
        size_ = other.size_;
        other.data_ = new char[1];
        other.data_[0] = '\0';
        other.size_ = 0;
    }

    MyString& operator=(MyString&& other) noexcept {
        if (this == &other) return *this;
        delete[] data_;
        data_ = other.data_;
        size_ = other.size_;
        other.data_ = new char[1];
        other.data_[0] = '\0';
        other.size_ = 0;
        return *this;
    }

    ~MyString() {
        delete[] data_;
    }

    char& operator[](size_t i) { return data_[i]; }
    const char* c_str() const { return data_; }
    size_t size() const { return size_; }
};
```

---

## 3.3 你要怎么理解它

### 默认构造
创建一个空串，但必须保证：
```cpp
data_[0] = '\0';
```
因为字符串最后要有结束符。

### 构造函数
如果传入的是 `const char*`，就分配新空间，把内容拷贝进来。

### 拷贝构造
新对象从旧对象初始化，所以要：
- 新开空间
- 拷贝内容

这就是深拷贝。

### 拷贝赋值
已有对象被重新赋值，所以要：
- 先防自赋值
- 再开新空间
- 再释放旧空间
- 再替换指针

### 移动构造 / 移动赋值
不再拷贝数据，而是“偷走”对方资源。

---

## 3.4 记忆点

### 这一题的核心关键词
- **深拷贝**
- **Rule of Five**
- **自赋值**
- **异常安全**
- **移动语义**

### 一句话记忆
**谁申请，谁释放；拷贝要新开；移动直接偷。**

---

## 3.5 易错点

### 1）浅拷贝
如果直接：
```cpp
data_ = other.data_;
```
那两个对象会指向同一块内存，析构时会重复释放。

### 2）赋值时先删后申请
如果你先 `delete[] data_`，再 `new` 失败，会出问题。  
更稳妥是：先申请新空间，再删旧空间。

### 3）忘记处理 `nullptr`
构造时传入空指针要小心。

### 4）移动后不把源对象置为空
这样源对象析构时可能出问题。

---

## 3.6 面试追问

### Q1：为什么要写拷贝构造？
因为类里有堆资源，默认浅拷贝不安全。

### Q2：为什么还要写拷贝赋值？
因为“初始化”和“赋值”是两回事。

### Q3：为什么要有移动语义？
为了减少深拷贝，提高性能。

### Q4：为什么析构函数要 `delete[]` 而不是 `delete`？
因为分配的是数组。

---

# 4. 手撕 Vector

---

## 4.1 这题在考什么

这题主要考：
- 动态数组的底层思想
- 扩容
- 元素拷贝
- 容量和大小的区别
- 深拷贝

---

## 4.2 标准版

```cpp
using namespace std;

class MyVector {
    int* data_;
    size_t size_;
    size_t capacity_;

    void grow() {
        size_t new_capacity = (capacity_ == 0 ? 1 : capacity_ * 2);
        int* new_data = new int[new_capacity];
        for (size_t i = 0; i < size_; ++i)
            new_data[i] = data_[i];
        delete[] data_;
        data_ = new_data;
        capacity_ = new_capacity;
    }

public:
    MyVector() : data_(nullptr), size_(0), capacity_(0) {}

    ~MyVector() {
        delete[] data_;
    }

    MyVector(const MyVector& other) {
        size_ = other.size_;
        capacity_ = other.capacity_;
        data_ = new int[capacity_];
        for (size_t i = 0; i < size_; ++i)
            data_[i] = other.data_[i];
    }

    MyVector& operator=(const MyVector& other) {
        if (this == &other) return *this;
        delete[] data_;
        size_ = other.size_;
        capacity_ = other.capacity_;
        data_ = new int[capacity_];
        for (size_t i = 0; i < size_; ++i)
            data_[i] = other.data_[i];
        return *this;
    }

    void push_back(int x) {
        if (size_ == capacity_)
            grow();
        data_[size_++] = x;
    }

    int& operator[](size_t i) { return data_[i]; }
    size_t size() const { return size_; }
    size_t capacity() const { return capacity_; }
};
```

---

## 4.3 你要理解的关键点

### size 和 capacity 的区别
- `size_`：已经存了多少元素
- `capacity_`：当前底层数组最多能装多少

### 为什么要扩容
因为底层数组满了之后，不能无限继续塞，只能重新申请更大的空间。

### 为什么通常扩成 2 倍
因为这样能保证 `push_back` 的均摊复杂度比较好。

---

## 4.4 记忆口诀

**Vector 三件套：`data_ / size_ / capacity_`**  
**满了就扩，扩了就拷，拷完换新。**

---

## 4.5 易错点

### 1）只改 `capacity_`，不重新分配空间
这是错的。

### 2）扩容后忘记拷贝旧数据

### 3）`size_` 和 `capacity_` 混了
很多人面试会讲反。

### 4）赋值时不处理自赋值

---

## 4.6 面试追问

### Q1：为什么 vector 扩容会导致迭代器失效？
因为底层数组地址变了。

### Q2：为什么不每次只扩 1 个？
那样扩容次数太多，效率差。

### Q3：vector 和 list 的区别？
- vector 连续内存，随机访问快
- list 插删方便，但空间额外开销大

---

# 5. shared_ptr 与变种

---

## 5.1 这题在考什么

这题主要考：
- 智能指针的本质
- 引用计数
- 共享所有权
- 析构时机
- 循环引用
- `unique_ptr / weak_ptr` 的作用

---

## 5.2 shared_ptr 简化版

```cpp
template<class T>
class MySharedPtr {
    T* ptr_;
    int* count_;

    void release() {
        if (!count_) return;
        --(*count_);
        if (*count_ == 0) {
            delete ptr_;
            delete count_;
        }
    }

public:
    MySharedPtr(T* ptr=nullptr) {
        ptr_ = ptr;
        count_ = ptr ? new int(1) : nullptr;
    }

    MySharedPtr(const MySharedPtr& other) {
        ptr_ = other.ptr_;
        count_ = other.count_;
        if (count_) ++(*count_);
    }

    MySharedPtr& operator=(const MySharedPtr& other) {
        if (this == &other) return *this;
        release();
        ptr_ = other.ptr_;
        count_ = other.count_;
        if (count_) ++(*count_);
        return *this;
    }

    ~MySharedPtr() {
        release();
    }

    T& operator*() { return *ptr_; }
    T* operator->() { return ptr_; }
    T* get() const { return ptr_; }
    int use_count() const { return count_ ? *count_ : 0; }
};
```

---

## 5.3 你要怎么理解

### shared_ptr 的核心
它不是“自动 delete”这么简单。  
它真正的核心是：

**多个人可以一起拥有同一个对象，谁都不能提前把它删掉。**

所以要有一个公共计数器：
- 新建时计数是 1
- 拷贝时加 1
- 析构时减 1
- 减到 0 才真正释放资源

---

## 5.4 记忆口诀

**对象一个，计数一个；拷贝加一，析构减一；减到零了再释放。**

---

## 5.5 易错点

### 1）忘记处理空指针
`count_` 可能是空。

### 2）赋值时不先 release
会导致原来那份资源泄漏。

### 3）把 shared_ptr 理解成对象本身线程安全
通常只是引用计数操作要线程安全，对象内部不一定安全。

---

## 5.6 面试追问

### Q1：为什么需要引用计数？
因为一个对象可能被多个智能指针共同拥有。

### Q2：为什么会有循环引用问题？
A 持有 B，B 持有 A，双方计数都不归零，就都释放不了。

### Q3：怎么解决循环引用？
用 `weak_ptr`。

---

## 5.7 unique_ptr 简化版

```cpp
template<class T>
class MyUniquePtr {
    T* ptr_;

public:
    MyUniquePtr(T* ptr=nullptr) : ptr_(ptr) {}
    ~MyUniquePtr() { delete ptr_; }

    MyUniquePtr(const MyUniquePtr&) = delete;
    MyUniquePtr& operator=(const MyUniquePtr&) = delete;

    MyUniquePtr(MyUniquePtr&& other) noexcept {
        ptr_ = other.ptr_;
        other.ptr_ = nullptr;
    }

    MyUniquePtr& operator=(MyUniquePtr&& other) noexcept {
        if (this == &other) return *this;
        delete ptr_;
        ptr_ = other.ptr_;
        other.ptr_ = nullptr;
        return *this;
    }

    T& operator*() { return *ptr_; }
    T* operator->() { return ptr_; }
};
```

---

## 5.8 unique_ptr 怎么记

它和 shared_ptr 的最大区别就是：

**shared_ptr 是共享所有权**  
**unique_ptr 是独占所有权**

所以：
- 不能拷贝
- 只能移动

---

## 5.9 weak_ptr 怎么答

- 不增加强引用计数
- 只是观察对象
- 用来解决循环引用问题

一句话：
**weak_ptr 看得见，但不拥有。**

---

# 6. LFU

---

## 6.1 这题为什么比 LRU 难

LRU 只维护“最近是否被用过”  
LFU 要维护“被用了多少次”

所以 LFU 要多一层结构。

核心是：
- `key -> 节点位置`
- `freq -> 同频次链表`
- `min_freq -> 当前最小频次`

---

## 6.2 标准版

```cpp
#include <unordered_map>
#include <list>
using namespace std;

class LFUCache {
    struct Node {
        int key;
        int val;
        int freq;
        Node(int k, int v, int f) : key(k), val(v), freq(f) {}
    };

    int capacity;
    int min_freq;
    unordered_map<int, list<Node>::iterator> key_to_node;
    unordered_map<int, list<Node>> freq_to_list;

    void touch(list<Node>::iterator it) {
        int key = it->key;
        int val = it->val;
        int freq = it->freq;

        freq_to_list[freq].erase(it);
        if (freq_to_list[freq].empty()) {
            freq_to_list.erase(freq);
            if (min_freq == freq)
                ++min_freq;
        }

        freq_to_list[freq + 1].push_front(Node(key, val, freq + 1));
        key_to_node[key] = freq_to_list[freq + 1].begin();
    }

public:
    LFUCache(int capacity) : capacity(capacity), min_freq(0) {}

    int get(int key) {
        if (!key_to_node.count(key))
            return -1;
        int val = key_to_node[key]->val;
        touch(key_to_node[key]);
        return val;
    }

    void put(int key, int value) {
        if (capacity == 0) return;

        if (key_to_node.count(key)) {
            key_to_node[key]->val = value;
            touch(key_to_node[key]);
            return;
        }

        if (key_to_node.size() == capacity) {
            auto node = freq_to_list[min_freq].back();
            key_to_node.erase(node.key);
            freq_to_list[min_freq].pop_back();
            if (freq_to_list[min_freq].empty())
                freq_to_list.erase(min_freq);
        }

        freq_to_list[1].push_front(Node(key, value, 1));
        key_to_node[key] = freq_to_list[1].begin();
        min_freq = 1;
    }
};
```

---

## 6.3 你要怎么理解 LFU

可以拆成两层理解：

### 第一层：按使用次数分类
比如：
- 访问 1 次的在 `freq=1`
- 访问 2 次的在 `freq=2`
- 访问 3 次的在 `freq=3`

### 第二层：同频次下还要分新旧
因为如果多个 key 频次一样，还要淘汰最旧的那个。

所以 `freq_to_list[freq]` 里也要保留顺序。

---

## 6.4 `touch` 函数是关键

每访问一次：
1. 把节点从旧频次链表删掉
2. 频次加 1
3. 插入到新频次链表头部
4. 更新 `key_to_node`
5. 必要时更新 `min_freq`

所以可以记成：

**删旧桶，频次加一，放进新桶，更新指针。**

---

## 6.5 记忆口诀

**LFU = key 表 + freq 表 + min_freq**  
**访问一次，频次加一，节点换桶。**

---

## 6.6 易错点

### 1）更新节点频次后，忘记更新 map 里的迭代器

### 2）旧频次桶空了，忘记更新 `min_freq`

### 3）同频次淘汰规则没想清楚
一般是同频次下淘汰最久未使用的，所以删链表尾。

---

## 6.7 面试追问

### Q1：LFU 和 LRU 的区别？
- LRU 看最近访问时间
- LFU 看访问频次

### Q2：为什么 LFU 更复杂？
因为要额外维护频次分组。

### Q3：同频次下怎么淘汰？
淘汰该频次中最久未使用的节点。

---

# 7. 生产者消费者

---

## 7.1 这题在考什么

主要考：
- 线程同步
- 互斥锁
- 条件变量
- 缓冲区满和空时如何阻塞

---

## 7.2 标准版

```cpp
#include <queue>
#include <mutex>
#include <condition_variable>
using namespace std;

class ProducerConsumer {
    queue<int> q;
    int capacity;
    mutex mtx;
    condition_variable cv_producer;
    condition_variable cv_consumer;

public:
    ProducerConsumer(int capacity) : capacity(capacity) {}

    void produce(int x) {
        unique_lock<mutex> lock(mtx);
        cv_producer.wait(lock, [&]() { return q.size() < capacity; });
        q.push(x);
        cv_consumer.notify_one();
    }

    int consume() {
        unique_lock<mutex> lock(mtx);
        cv_consumer.wait(lock, [&]() { return !q.empty(); });
        int x = q.front();
        q.pop();
        cv_producer.notify_one();
        return x;
    }
};
```

---

## 7.3 你要怎么理解

这里有一个共享缓冲区 `q`：

- 生产者往里面放东西
- 消费者从里面取东西

问题在于：
- 队列满了，生产者不能继续放
- 队列空了，消费者不能继续取

所以需要两个条件变量：
- 一个通知“可以生产了”
- 一个通知“可以消费了”

---

## 7.4 记忆口诀

**满了生产者睡，空了消费者睡；状态变了就唤醒。**

---

## 7.5 易错点

### 1）`wait` 不写谓词
可能会被虚假唤醒坑到。

### 2）用 `lock_guard` 调 `wait`
不行，`wait` 需要 `unique_lock`。

### 3）放完数据不通知消费者
消费者会一直睡。

---

## 7.6 面试追问

### Q1：为什么 `wait` 要传 lambda？
为了防止虚假唤醒。

### Q2：为什么要用 `unique_lock`？
因为 `wait` 会在内部解锁和重新加锁。

### Q3：为什么不是一个条件变量也行吗？
可以，但两个更清晰。

---

# 8. 线程安全单例

---

## 8.1 这题在考什么

考你知不知道：
- 单例是什么
- 为什么要线程安全
- C++11 局部静态变量初始化线程安全

---

## 8.2 标准版

```cpp
class Singleton {
    Singleton() {}
    Singleton(const Singleton&) = delete;
    Singleton& operator=(const Singleton&) = delete;

public:
    static Singleton& get_instance() {
        static Singleton instance;
        return instance;
    }
};
```

---

## 8.3 为什么这版最好

因为：
- 代码最短
- 不用自己写锁
- 不用自己回收
- C++11 起局部静态对象初始化线程安全

---

## 8.4 记忆口诀

**局部静态一招鲜，线程安全还省心。**

---

## 8.5 面试追问

### Q1：为什么不用双检锁？
可以用，但更复杂，容易出错。

### Q2：为什么删除拷贝构造和赋值？
防止单例被复制出多个对象。

---

# 9. 并发安全队列

---

## 9.1 这题在考什么

这题和生产者消费者很像，但更偏工程：
- 线程安全队列怎么封装
- 阻塞 pop 怎么写
- 队列关闭后线程怎么退出

---

## 9.2 标准版

```cpp
#include <queue>
#include <mutex>
#include <condition_variable>
using namespace std;

template<class T>
class SafeQueue {
    queue<T> q;
    mutex mtx;
    condition_variable cv;
    bool stopped = false;

public:
    void push(const T& value) {
        {
            lock_guard<mutex> lock(mtx);
            q.push(value);
        }
        cv.notify_one();
    }

    bool pop(T& value) {
        unique_lock<mutex> lock(mtx);
        cv.wait(lock, [&]() { return stopped || !q.empty(); });

        if (stopped && q.empty())
            return false;

        value = q.front();
        q.pop();
        return true;
    }

    void close() {
        {
            lock_guard<mutex> lock(mtx);
            stopped = true;
        }
        cv.notify_all();
    }
};
```

---

## 9.3 你要怎么理解

这个类一般是给线程池、任务系统当底层组件的。

### push
放任务进去，然后唤醒一个等待线程。

### pop
如果队列空，就阻塞等。  
如果队列关闭且为空，就返回 `false`，告诉外部该退出了。

### close
通知所有等待线程：别等了，可以结束了。

---

## 9.4 记忆口诀

**有任务就取，没任务就等；队列一关，线程别撑。**

---

## 9.5 易错点

### 1）没有 `stopped`
那线程池析构时，工作线程可能永远卡住。

### 2）只 `notify_one`
关闭时应该 `notify_all`，让所有等待线程都醒来。

---

## 9.6 面试追问

### Q1：为什么 `pop` 返回 `bool`？
为了告诉调用方：是拿到了数据，还是队列已经关闭。

### Q2：为什么线程池需要这个关闭语义？
不然工作线程无法优雅退出。

---

# 10. 简易线程池

---

## 10.1 这题在考什么

线程池本质上就是：
- 一堆工作线程
- 一个任务队列
- 线程不断从队列里取任务执行

考点主要是：
- 线程复用
- 任务分发
- 优雅关闭

---

## 10.2 标准版

```cpp
#include <vector>
#include <thread>
#include <functional>
using namespace std;

class ThreadPool {
    SafeQueue<function<void()>> task_queue;
    vector<thread> workers;

public:
    ThreadPool(int n) {
        for (int i = 0; i < n; ++i) {
            workers.emplace_back([this]() {
                function<void()> task;
                while (task_queue.pop(task)) {
                    task();
                }
            });
        }
    }

    void add_task(function<void()> task) {
        task_queue.push(task);
    }

    ~ThreadPool() {
        task_queue.close();
        for (auto& worker : workers) {
            if (worker.joinable())
                worker.join();
        }
    }
};
```

---

## 10.3 你要怎么理解

### 初始化
创建 n 个线程，每个线程都做同一件事：
- 不停从任务队列取任务
- 取到就执行
- 取不到且队列关闭就退出

### add_task
外部提交一个任务进队列。

### 析构
关闭队列，然后等所有工作线程结束。

---

## 10.4 记忆口诀

**线程先建好，任务往里抛；队列关掉后，工人全收摊。**

---

## 10.5 易错点

### 1）析构时只 close，不 join
线程还在跑，程序可能出问题。

### 2）任务队列没有关闭机制
线程会一直阻塞。

### 3）把线程池理解成“每来一个任务开一个线程”
那就不是线程池了。

---

## 10.6 面试追问

### Q1：线程池为什么比每次新建线程更高效？
减少线程频繁创建和销毁的开销。

### Q2：任务队列为什么要线程安全？
因为可能多个线程同时投任务、多个线程同时取任务。

### Q3：线程池还能扩展什么？
- 返回值
- Future
- 动态扩缩容
- 定时任务

---

# 11. 用队列实现栈

---

## 11.1 这题考什么

考你会不会把一个数据结构的行为，用另一个数据结构模拟出来。

栈是：
- 后进先出

队列是：
- 先进先出

关键是怎么让队列“表现得像栈”。

---

## 11.2 标准版

```cpp
#include <queue>
using namespace std;

class MyStack {
    queue<int> q;

public:
    void push(int x) {
        q.push(x);
        int n = q.size();
        while (n > 1) {
            q.push(q.front());
            q.pop();
            --n;
        }
    }

    int pop() {
        int x = q.front();
        q.pop();
        return x;
    }

    int top() {
        return q.front();
    }

    bool empty() {
        return q.empty();
    }
};
```

---

## 11.3 你要怎么理解

关键就在 `push`。

每次新元素入队后，把前面的老元素全部重新转到后面去。  
这样新元素就会跑到队首。

于是：
- 队首看起来就像“栈顶”
- `pop/top` 都直接对队首操作

---

## 11.4 记忆口诀

**新来的先不急，老的都往后排；新元素转到前面，就像栈顶。**

---

## 11.5 面试追问

### Q1：为什么不是在 `pop` 的时候处理？
也可以，但这版把复杂度放在 `push`，`pop/top` 更简单。

---

# 12. 用栈实现队列

---

## 12.1 这题考什么

也是在考数据结构模拟。

队列要求：
- 先进先出

栈是：
- 后进先出

所以要用两个栈来“倒一下顺序”。

---

## 12.2 标准版

```cpp
#include <stack>
using namespace std;

class MyQueue {
    stack<int> in_st;
    stack<int> out_st;

    void move() {
        if (!out_st.empty()) return;
        while (!in_st.empty()) {
            out_st.push(in_st.top());
            in_st.pop();
        }
    }

public:
    void push(int x) {
        in_st.push(x);
    }

    int pop() {
        move();
        int x = out_st.top();
        out_st.pop();
        return x;
    }

    int peek() {
        move();
        return out_st.top();
    }

    bool empty() {
        return in_st.empty() && out_st.empty();
    }
};
```

---

## 12.3 你要怎么理解

### `in_st`
负责接收新来的元素。

### `out_st`
负责真正出队。

当 `out_st` 空时，把 `in_st` 全部倒进去。  
倒完以后，最早进来的元素就到了 `out_st` 的栈顶。

所以：
- `push` 只进 `in_st`
- `pop/peek` 优先看 `out_st`

---

## 12.4 记忆口诀

**进的先进 in，出的看 out；out 没货了，再把 in 全倒过去。**

---

## 12.5 面试追问

### Q1：为什么不每次 `push` 都倒？
因为这样太浪费。  
只在 `out_st` 为空时再倒，可以做到均摊更优。



---

# 总记忆区

## 一句话总表

- **LRU**：哈希表 + 循环双向链表 + 单 dummy
- **TTL-LRU**：LRU + expire + 惰性删除
- **String**：深拷贝 + Rule of Five
- **Vector**：动态数组 + 扩容
- **shared_ptr**：引用计数
- **LFU**：key 表 + freq 表 + min_freq
- **生产者消费者**：锁 + 条件变量
- **单例**：局部静态对象
- **并发安全队列**：阻塞 pop + close
- **线程池**：任务队列 + 工作线程
- **队列实现栈**：push 时轮转
- **栈实现队列**：两个栈倒顺序

## 最适合面试前速背的口诀

- **LRU**：查到提头，没查插头，满了删尾
- **TTL-LRU**：先判过期，活着提头
- **String**：谁申请谁释放，拷贝新开，移动直接偷
- **Vector**：满了扩容，扩了拷贝
- **shared_ptr**：拷贝加一，析构减一，归零释放
- **LFU**：访问一次，频次加一，节点换桶
- **生产者消费者**：满了等，空了等，状态变了就唤醒
- **线程池**：线程先建好，任务往里抛，关队列再回收
