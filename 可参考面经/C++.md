# C++核心
以下是对C++核心知识点的精准补充与完善，确保内容无错误、细节完整且逻辑清晰：

## 一、多态
### 1. 虚函数表和虚函数表指针
- 虚函数表（vftable）：编译期为含虚函数的类创建，按**虚函数声明顺序**存储虚函数地址，属于类级别的静态数据，存放在`.rodata`（只读数据段），供所有对象共享。
- 虚函数表指针（vfptr）：每个对象包含一个，运行期初始化，指向虚函数表首地址，存放位置与对象一致（堆/栈）。
- 继承规则：子类**深拷贝**父类虚函数表；若子类重写虚函数，覆盖表中对应地址；若子类新增虚函数，追加到虚函数表末尾。
- 示例：基类`Base`有虚函数`func1()`，子类`Derived`重写`func1()`并新增`func2()`，则`Derived`的虚函数表中`func1()`为子类地址，`func2()`追加到表尾。

### 2. 多态传参：指针/引用而非值传递
- 切片问题（核心原因）：值传递时，派生类对象会被“切割”为基类对象，仅保留基类部分，丢失派生类的虚函数表和成员，无法触发动态绑定。
- 性能开销：值传递需拷贝整个对象（尤其是大对象），指针/引用仅拷贝地址（4/8字节），效率更高。
- 示例：
  ```cpp
  class Base { virtual void show() { cout << "Base" << endl; } };
  class Derived : public Base { void show() override { cout << "Derived" << endl; } };
  void func(Base b) { b.show(); } // 值传递：切片，输出Base
  void func(Base& b) { b.show(); } // 引用传递：输出Derived
  ```

### 3. 构造/析构函数与虚函数
- **构造函数不能是虚函数**：
  构造函数执行时，对象尚未完全实例化，vfptr未初始化，无法通过虚函数表调用；且继承体系中“先构造基类、后构造子类”的规则，也无需虚构造。
- **析构函数建议设为虚函数（基类）**：
  若基类析构非虚，用基类指针指向子类对象时，仅调用基类析构，子类资源无法释放（内存泄漏）；设为虚析构后，会按“子类→基类”顺序析构。
  ```cpp
  class Base { public: virtual ~Base() { cout << "Base析构" << endl; } };
  class Derived : public Base { ~Derived() { cout << "Derived析构" << endl; } };
  Base* p = new Derived(); delete p; // 输出：Derived析构 → Base析构
  ```

### 4. 不能作为虚函数的函数
- 构造函数：原因同上；
- 静态成员函数：属于类而非对象，无this指针，无法绑定到具体对象，编译期确定调用；
- 友元函数：不属于类的成员函数，无法被继承，无虚函数表入口；
- 内联函数：inline是编译期展开指令，虚函数是运行时动态绑定，若内联函数标记为虚函数，编译器会忽略inline属性。

## 二、静态多态
### 1. 函数重载（Overload）
- 规则：同一作用域、函数名相同、参数列表（数量/类型/顺序）不同；**返回值不影响重载**（编译器无法通过返回值区分调用）。
- 匹配优先级：精确匹配 > 类型提升（如char→int） > 隐式转换（如int→double）。
- 示例：`void f(int)`与`void f(double)`是重载，`int f()`与`void f()`不是（仅返回值不同）。

### 2. 泛型（模板）
- 类模板/函数模板：编译期“实例化”，根据传入的类型生成具体代码（无运行时开销）。
- 示例：
  ```cpp
  template <typename T> T add(T a, T b) { return a + b; }
  add(1, 2); // 实例化int版本
  add(1.1, 2.2); // 实例化double版本
  ```

## 三、拷贝/移动语义
### 1. 拷贝构造/赋值函数
- 拷贝构造函数：
  - 必须传**const引用**：值传递会触发无限递归（拷贝构造调用自身）；
  - 调用场景：值传递参数、用对象初始化新对象、函数返回值为对象（编译器可能优化）。
- 赋值运算符重载：
  - 返回`*this`（引用），支持链式赋值（`a = b = c`）；
  - 需处理自赋值（`if (this == &rhs) return *this`）。
- 示例（深拷贝实现）：
  ```cpp
  class String {
  private:
      char* buf;
  public:
      String(const char* str) : buf(new char[strlen(str)+1]) { strcpy(buf, str); }
      // 拷贝构造（深拷贝）
      String(const String& rhs) : buf(new char[strlen(rhs.buf)+1]) {
          strcpy(buf, rhs.buf);
      }
      // 赋值重载
      String& operator=(const String& rhs) {
          if (this == &rhs) return *this;
          delete[] buf; // 释放原有资源
          buf = new char[strlen(rhs.buf)+1];
          strcpy(buf, rhs.buf);
          return *this;
      }
      ~String() { delete[] buf; }
  };
  ```

### 2. 移动构造/赋值函数
- 核心：转移资源所有权（而非拷贝），避免内存分配/拷贝开销。
- `noexcept`必要性：`vector`/`deque`等容器扩容时，若移动构造无`noexcept`，为保证异常安全，会降级调用拷贝构造。
- 移动赋值需先释放当前对象资源，再转移指针（避免内存泄漏）。
- 示例：
  ```cpp
  class String {
  private:
      char* buf;
  public:
      // 移动构造
      String(String&& rhs) noexcept : buf(rhs.buf) {
          rhs.buf = nullptr; // 源对象放弃资源所有权
      }
      // 移动赋值
      String& operator=(String&& rhs) noexcept {
          if (this == &rhs) return *this;
          delete[] buf;
          buf = rhs.buf;
          rhs.buf = nullptr;
          return *this;
      }
  };
  ```

## 四、继承与重载
### 1. 菱形继承（虚继承）
- 问题：多层继承导致基类被多次实例化（二义性），如`A→B`、`A→C`、`B→D`、`C→D`，D中存在两个A的实例。
- 解决：虚继承（`virtual`），让子类共享基类的唯一实例。
  ```cpp
  class A {};
  class B : virtual public A {}; // 虚继承A
  class C : virtual public A {}; // 虚继承A
  class D : public B, public C {}; // D中仅一个A实例
  ```

### 2. Overload vs Override vs Overwrite
- Overload（重载）：同一作用域、函数名相同、参数不同（静态多态）；
- Override（重写）：派生类重写基类**虚函数**，函数签名（名/参数/返回值）完全一致（C++11加`override`关键字，编译器检查重写合法性）；
- Overwrite（隐藏）：派生类函数屏蔽基类同名函数（非虚函数），无论参数是否相同。

## 五、深拷贝与浅拷贝
- 浅拷贝：仅拷贝成员变量的值（包括指针地址），导致多个对象共享同一块内存，析构时触发`double free`（重复释放）。
- 深拷贝：拷贝指针指向的内存（重新分配内存+拷贝数据），每个对象独立拥有资源。
- 浅拷贝风险示例：
  ```cpp
  class Test {
  public:
      int* p = new int(10);
      ~Test() { delete p; } // 浅拷贝时，两个对象析构同一指针，崩溃
  };
  Test a; Test b = a; // 浅拷贝，b.p = a.p
  ```

## 六、核心关键字
### 1. this指针
- 特性：非静态成员函数的隐含参数（`Class* const this`），编译器自动传入；
- const成员函数中：`this`变为`const Class* const this`，无法修改成员变量（除非变量被`mutable`修饰）；
- 不能在静态成员函数中使用（无this指针）。

### 2. const
- 扩展：`constexpr`（C++11）：编译期常量，比`const`更严格（`const`可能是运行时常量）；
  ```cpp
  constexpr int N = 10; // 编译期确定
  int n = 10; const int M = n; // M是运行时常量
  ```
- const成员函数：函数后加`const`，示例：`void show() const;`。

### 3. sizeof
- 计算规则：
  - 数组：`sizeof(arr)` = 数组总字节数（`sizeof(arr[0]) * 长度`）；
  - 指针：固定值（32位4字节，64位8字节），与指向的类型无关；
  - 类：包含成员变量、vfptr（1个），遵循内存对齐（如`#pragma pack(4)`指定对齐数）；
  - 空类：`sizeof(Empty) = 1`（编译器分配1字节占位，标识对象存在）。

### 4. volatile
- 核心：禁止编译器优化（每次读写都从内存取/存，而非寄存器），**不保证原子性**（多线程需加锁）；
- 典型场景：硬件寄存器、多线程共享标志、信号处理函数中的变量。

### 5. #define vs inline
- #define缺点：无类型检查、运算符优先级问题、可能重复展开；
  ```cpp
  #define ADD(a,b) a+b
  cout << ADD(1,2)*3; // 展开为1+2*3=7，非预期的9
  ```
- inline限制：递归函数、大函数（编译器会拒绝内联）、虚函数（运行时绑定，inline失效）；
- 编译器对inline的态度：`inline`是建议，最终是否内联由编译器决定。

### 6. static
- 静态成员变量：类外初始化（`int Class::x = 0;`），所有对象共享；
- 静态成员函数：无this指针，只能访问静态成员（变量/函数）。

### 7. mutable
- 仅修饰类的非静态成员变量，允许在const成员函数中修改；
- 场景：缓存、计数等无需影响对象“逻辑常量性”的变量；
  ```cpp
  class Cache {
  private:
      mutable int cache_hit = 0; // 缓存命中次数
  public:
      int get_data() const {
          cache_hit++; // const函数中可修改mutable变量
          return 10;
      }
  };
  ```

### 8. noexcept
- 两种形式：`noexcept`（函数永不抛异常）、`noexcept(表达式)`（表达式为true时不抛异常）；
- 作用：编译器优化（如移动构造）、简化异常处理（调用者无需捕获异常）。

## 七、内存管理
### 1. New/delete vs malloc/free
| 特性     | new/delete                            | malloc/free               |
| -------- | ------------------------------------- | ------------------------- |
| 语法     | 运算符                                | 函数                      |
| 初始化   | 调用构造/析构函数                     | 仅分配/释放内存，不初始化 |
| 返回值   | 对应类型指针                          | void*，需强制类型转换     |
| 失败处理 | 抛`bad_alloc`（或nothrow返回nullptr） | 返回nullptr               |
| 数组操作 | `new[]`/`delete[]`（必须配对）        | 需手动计算数组大小        |

### 2. 内存模型（地址由低到高）
| 段名            | 存储内容                               | 特性                                 |
| --------------- | -------------------------------------- | ------------------------------------ |
| 代码段（.text） | 二进制机器码、只读指令                 | 只读、共享                           |
| 数据段（.data） | 初始化的全局/静态变量、常量（.rodata） | 只读/可写、静态分配                  |
| BSS段           | 未初始化的全局/静态变量                | 加载时初始化为0、不占磁盘空间        |
| 堆区            | 动态分配的内存（new/malloc）           | 手动管理、空间大、有碎片             |
| 栈区            | 局部变量、函数参数、返回值             | 自动管理、效率高、空间小（默认几MB） |

### 3. 内存泄漏
- 类型：堆泄漏（最常见）、资源泄漏（文件句柄、套接字、锁）、全局/静态变量泄漏；
- 检测工具：
  - `valgrind`：`valgrind --tool=memcheck --leak-check=full --show-leak-kinds=all ./program`；
  - `malloc_stats()`：GDB中调用，对比函数前后堆内存使用量；
- 避免方法：RAII（智能指针、自定义资源管理类）、及时释放资源。

## 八、STL容器
### 1. 容器特性与迭代器
| 容器                        | 底层实现           | 迭代器类型     | 核心特性                                             |
| --------------------------- | ------------------ | -------------- | ---------------------------------------------------- |
| array                       | 静态数组           | 随机访问迭代器 | 栈存储、大小固定、效率最高                           |
| vector                      | 动态数组           | 随机访问迭代器 | 堆存储、尾部操作高效、扩容因子1.5/2                  |
| deque                       | 中控器+缓冲区      | 随机访问迭代器 | 头尾操作高效、缓冲区分散                             |
| list                        | 双向循环链表       | 双向迭代器     | 任意位置插入/删除高效、无随机访问                    |
| set/map                     | 红黑树             | 双向迭代器     | 有序、唯一（multiset/multimap允许重复）、O(logn)操作 |
| unordered_set/unordered_map | 哈希表（链地址法） | 前向迭代器     | 无序、唯一、O(1)平均查找、高内存占用                 |

### 2. vector扩容
- 扩容因子：GCC（2倍）、MSVC（1.5倍）；
- `reserve` vs `resize`：
  - `reserve(n)`：预分配n个元素的空间（仅改capacity，不改size）；
  - `resize(n)`：调整size为n，不足则构造元素，超出则析构元素；
- `shrink_to_fit()`（C++11）：释放多余容量（将capacity降至size）。

### 3. push_back() vs emplace_back()
- `push_back`：先构造临时对象 → 拷贝/移动到容器 → 销毁临时对象；
- `emplace_back`：直接在容器内存中就地构造对象（传递构造参数），无临时对象开销；
- 示例：
  ```cpp
  struct Person { string name; int age; Person(string n, int a) : name(n), age(a) {} };
  vector<Person> v;
  v.push_back(Person("Tom", 20)); // 临时对象+移动
  v.emplace_back("Jerry", 18); // 就地构造，更高效
  ```

### 4. 哈希表（unordered系列）
- 哈希冲突解决：链地址法（默认），链表长度>8时转为红黑树，<6时转回链表；
- 负载因子：默认1.0，超过则`rehash`（新建桶数组，重新哈希并迁移元素）；
- 自定义哈希函数：
  ```cpp
  struct MyHash {
      size_t operator()(const MyType& t) const {
          return hash<int>()(t.id); // 基于id哈希
      }
  };
  unordered_set<MyType, MyHash> s;
  ```

## 九、C++11核心新特性
### 1. auto与decltype
- auto：编译期推导类型，限制：不能用于函数参数、不能定义数组；
  ```cpp
  auto a = 10; // int
  auto b = 1.1; // double
  auto& c = a; // int&
  ```
- decltype：推导表达式类型，规则：
  - 变量：`decltype(var)` = 变量类型；
  - 表达式：左值→引用，右值→值类型；
  ```cpp
  int x = 10;
  decltype(x) y = 20; // int
  decltype((x)) z = x; // int&（(x)是左值表达式）
  ```

### 2. Lambda表达式
- 完整语法：`[捕获列表] (参数列表) mutable noexcept -> 返回类型 { 函数体 }`；
- 捕获列表：
  - 值捕获：`[x]`（拷贝x，默认不可修改，加mutable可修改）；
  - 引用捕获：`[&x]`（引用x，可修改）；
  - 隐式捕获：`[=]`（所有变量值捕获）、`[&]`（所有变量引用捕获）；
  - 移动捕获（C++14）：`[x = std::move(y)]`；
- 特性：无捕获的lambda可转换为函数指针。

### 3. 右值引用、移动语义、完美转发
- 左值/右值区分：左值（有名字、可寻址，如变量）、右值（无名字、不可寻址，如临时对象、字面量）；
- 右值引用（`T&&`）：绑定右值，实现移动语义；
- `std::move`：强制将左值转为右值引用（仅类型转换，不移动资源）；
- 完美转发（`std::forward`）：保留参数的左/右值特性，示例：
  ```cpp
  template <typename T>
  void forward_func(T&& arg) {
      target_func(std::forward<T>(arg)); // 按原类型转发
  }
  ```

### 4. 智能指针（RAII）
| 智能指针   | 特性                                   | 适用场景                     |
| ---------- | -------------------------------------- | ---------------------------- |
| unique_ptr | 独占所有权、可移动不可拷贝、支持数组   | 单个对象/数组的独占管理      |
| shared_ptr | 共享所有权、引用计数、线程安全（计数） | 多个对象共享同一资源         |
| weak_ptr   | 弱引用、不参与计数、解决循环引用       | 配合shared_ptr，避免循环引用 |

- 关键细节：
  - `make_shared`：更高效（一次分配对象+控制块内存），避免内存碎片；
  - 循环引用示例：`A`含`shared_ptr<B>`，`B`含`shared_ptr<A>` → 引用计数永不归零，改用`weak_ptr`解决；
  - `weak_ptr`常用方法：`lock()`（返回shared_ptr，过期则返回空）、`expired()`（检查是否过期）。

## 十、类型转换
| 转换方式         | 用途                                | 安全性 | 注意事项                                              |
| ---------------- | ----------------------------------- | ------ | ----------------------------------------------------- |
| static_cast      | 基本类型转换、向上转型（派生→基类） | 中等   | 不能去除const、不能向下转型（不安全）                 |
| dynamic_cast     | 基类/派生类安全转换（运行时检查）   | 高     | 基类必须有虚函数；指针失败返回nullptr，引用抛bad_cast |
| const_cast       | 去除const/volatile属性              | 低     | 仅修改类型属性，不修改对象本身                        |
| reinterpret_cast | 底层二进制重解释（任意类型转换）    | 极低   | 平台相关，仅用于底层编程                              |

## 十一、编译与链接
### 1. 代码执行流程（预处理→编译→汇编→链接→加载）
- 预处理：处理`#include`（展开头文件）、`#define`（宏替换）、`#ifdef`（条件编译）、删除注释，生成`.i`文件；
- 编译：词法/语法分析、优化，生成汇编代码（`.s`文件）；
- 汇编：将汇编代码转为机器码，生成目标文件（`.o`/`.obj`）；
- 链接：
  - 静态链接：将静态库代码拷贝到可执行文件，运行时无需依赖；
  - 动态链接：记录动态库符号，运行时加载（`ld.so`/`dll`）；
- 加载：操作系统将可执行文件加载到内存，建立虚拟地址空间，执行入口函数。

### 2. 静态库vs动态库
| 特性       | 静态库（.a/.lib） | 动态库（.so/.dll） |
| ---------- | ----------------- | ------------------ |
| 链接时机   | 编译期            | 运行期             |
| 可执行文件 | 体积大            | 体积小             |
| 更新       | 需重新编译链接    | 替换库文件即可     |
| 依赖       | 无运行时依赖      | 需依赖库文件存在   |
| 内存占用   | 多进程重复加载    | 多进程共享         |

## 十二、指针与引用
### 1. 核心区别
| 特性   | 指针               | 引用                 |
| ------ | ------------------ | -------------------- |
| 本质   | 变量（存储地址）   | 别名（语法糖）       |
| 初始化 | 可空（nullptr）    | 必须绑定变量，不可空 |
| 重定向 | 可指向不同变量     | 初始化后不可更改     |
| 操作   | 需解引用（*）      | 直接使用             |
| 语法   | `int* p = nullptr` | `int& r = x`         |

### 2. 空指针访问成员函数
- 规则：若成员函数**不访问成员变量/this指针**，可调用；若访问，则崩溃；
  ```cpp
  class Test {
  public:
      void func1() { cout << "func1" << endl; } // 无this使用，可调用
      void func2() { cout << this->x << endl; } // 访问x，崩溃
      int x;
  };
  Test* p = nullptr;
  p->func1(); // 输出func1
  p->func2(); // 崩溃（this为nullptr）
  ```

## 十三、STL线程安全
- 标准规则（C++11）：
  1. 不同线程读同一容器 → 安全；
  2. 不同线程写不同元素 → 未定义行为；
  3. 同一线程写、其他线程读/写 → 未定义行为；
- 注意：`swap`/`clear`等容器整体操作不是线程安全的，需加锁（如`std::mutex`）。
