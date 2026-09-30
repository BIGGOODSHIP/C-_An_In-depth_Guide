# C++ 常用标准库详解

> 本文件以**工具书查阅属性**讲解 C++11 高频标准库。每个库统一给出：头文件、核心组件、常用类型、关键接口、代码示例、适用场景。内容严格限定在 C++11 及之前标准。

---

## 目录

1. `<utility>` — 通用工具
2. `<tuple>` — 元组
3. `<type_traits>` — 类型萃取
4. `<chrono>` — 时间库
5. `<memory>` — 内存与智能指针
6. `<string>` — 字符串
7. `<sstream>` — 字符串流
8. `<fstream>` — 文件流
9. `<thread>` — 线程
10. `<mutex>` — 互斥锁
11. `<condition_variable>` — 条件变量
12. `<atomic>` — 原子操作
13. `<functional>` — 函数对象
14. `<random>` — 随机数
15. `<limits>` — 类型极限
16. 交叉使用与工程要点

---

## 1. `<utility>` — 通用工具

**定位**：承载一对值、交换、移动与转发等最基础工具。

### 核心组件

- **`std::pair<T1,T2>`**：两个值的组合。
- **`std::make_pair(a,b)`**：构造 pair（自动推导类型）。
- **`std::move(x)`**：无条件右值转换（移动语义触发器）。
- **`std::forward<T>(x)`**：完美转发，保留值类别。
- **`std::swap(a,b)`**：交换两对象。
- **`std::rel_ops`**：由 `==`/`<` 推导其余比较（不推荐，易与用户定义冲突）。
- `std::get<0>(pair)`、`std::tie`（C++11 结构化解包的前身）。

### 关键接口

```cpp
#include <utility>
std::pair<int, std::string> p(1, "one");
p.first;  p.second;

auto q = std::make_pair(2, std::string("two"));   // 类型推导
std::swap(p, q);

// move 与 forward
std::string s = "hi";
std::vector<std::string> v;
v.push_back(std::move(s));       // 搬移，s 变为空壳
```

### 适用场景

- 返回值/`map` 的 key-value 组合。
- `move`/`forward` 贯穿整个移动语义与完美转发体系。
- 通用交换、关联容器节点操作。

---

## 2. `<tuple>` — 元组

**定位**：固定个数的**异构类型**组合（C++11 引入），比 pair 支持任意多成员。

### 核心组件与常用类型

- `std::tuple<T...>`：异构元素元组。
- `std::make_tuple(args...)`：推导构造。
- `std::get<I>(t)` / `std::get<T>(t)`：按索引或类型取元素。
- `std::tie(a,b,...)`：绑定引用以解包。
- `std::tuple_size<T>` / `std::tuple_element<I,T>`：编译期查询。
- `std::forward_as_tuple`：引用打包（转发用）。

### 关键接口

```cpp
#include <tuple>
auto t = std::make_tuple(1, 3.14, std::string("x"));
int   a  = std::get<0>(t);
double b = std::get<1>(t);
auto  s  = std::get<std::string>(t);      // 按类型取（类型唯一时）

// tie 解包
int x; std::string y;
std::tie(x, std::ignore, y) = t;          // std::ignore 跳过第 2 个

// 结构化访问
std::tuple<int,int> r = std::make_tuple(1,2);
```

### 适用场景

- 多返回值。
- 函数签名打包（回调参数组）。
- 泛型编程中的类型组合。

---

## 3. `<type_traits>` — 类型萃取

**定位**：编译期查询/变换类型属性，是模板元编程与泛型约束的基础。

### 核心组件（常用）

| 类别 | 成员 | 含义 |
|---|---|---|
| 属性查询 | `is_integral<T>`、`is_floating_point<T>`、`is_pointer<T>`、`is_class<T>`、`is_enum<T>` | 类型类别布尔 |
| 限定符 | `is_const<T>`、`remove_const<T>`、`add_const<T>`、`remove_cv<T>` | const 处理 |
| 引用/指针 | `remove_reference<T>`、`remove_pointer<T>`、`decay<T>` | 去修饰 |
| 关系 | `is_same<T,U>`、`is_base_of<B,D>`、`is_convertible<From,To>` | 类型关系 |
| 对象性质 | `is_trivially_copyable<T>`、`is_pod<T>`、`is_standard_layout<T>` | 布局性质 |
| 构造/赋值 | `is_constructible<T,Args...>`、`is_copy_constructible<T>`、`is_move_constructible<T>` | 能力 |
| 变换 | `conditional<bool,T,F>::type`、`enable_if<Cond,T>::type`、`make_unsigned<T>::type` | 类型选择 |

### 关键接口

```cpp
#include <type_traits>
static_assert(std::is_integral<int>::value, "int must be integral");
static_assert(std::is_same<typename std::remove_reference<int&>::type, int>::value, "");

template <typename T>
typename std::enable_if<std::is_integral<T>::value, T>::type
half(T v) { return v / 2; }          // 仅整数类型可用

// conditional 编译期选型
using IntType = typename std::conditional<sizeof(int)>=4, int, long>::type;
```

### 适用场景

- 编译期断言（`static_assert`）。
- SFINAE 约束重载。
- 泛型库的类型规范与分发。

---

## 4. `<chrono>` — 时间库

**定位**：提供高精度、类型安全的时间点（time_point）、时长（duration）与时钟（clock）抽象。

### 核心组件与常用类型

- **时钟 `clock`**：`system_clock`（系统时间）、`steady_clock`（单调，适合计时）、`high_resolution_clock`（高精度，常为 steady 别名）。
- **时长 `std::chrono::duration<Rep, Period>`**：数值 + 周期比例（如 `nanoseconds`/`milliseconds`/`seconds`）。
- **时间点 `std::chrono::time_point<Clock, Duration>`**：相对时钟纪元的时刻。
- 便捷别名：`nanoseconds`、`microseconds`、`milliseconds`、`seconds`、`minutes`、`hours`。

### 关键接口

```cpp
#include <chrono>
using namespace std::chrono;

// 计时
auto start = steady_clock::now();
do_work();
auto dur = steady_clock::now() - start;
std::cout << duration_cast<milliseconds>(dur).count() << " ms\n";

// 构造时长
seconds s(5);
milliseconds ms = duration_cast<milliseconds>(seconds(3));  // 3000
auto half_second = duration<double>(0.5);                   // 分数秒

// system_clock 取当前时刻
auto now = system_clock::now();
time_t tt = system_clock::to_time_t(now);
std::cout << std::ctime(&tt);
```

要点：`duration_cast` 截断；浮点时长用 `duration<double>` 保留小数；`steady_clock` 用于测时间差（不受系统时间调整影响），`system_clock` 用于日历时间。

### 适用场景

- 性能测量、超时控制、定时调度。
- 时间戳与日志。
- 线程等待的时限参数（`wait_for`）。

---

## 5. `<memory>` — 内存与智能指针

**定位**：RAII 所有权管理核心。智能指针把“手工 new/delete”封装为对象生命周期管理。

### 核心组件

| 组件 | 语义 |
|---|---|
| `std::unique_ptr<T>` | **独占所有权**，不可拷贝、可移动；析构自动释放 |
| `std::shared_ptr<T>` | **共享所有权**，引用计数；计数归零释放 |
| `std::weak_ptr<T>` | 弱引用，不增加计数，用于破除循环引用 |
| `std::make_unique<T>` | C++14 才有；C++11 手写或直接用 `unique_ptr<T>(new T)` |
| `std::make_shared<T>` | C++11：一次分配（对象+控制块），异常安全 |
| `std::enable_shared_from_this` | 在成员内安全取得 `shared_from_this()` |
| `std::allocator<T>` | 分配器 |
| `std::unique_ptr<T[]>` | 数组特化，自动 `delete[]` |

### 关键接口

```cpp
#include <memory>

// unique_ptr
std::unique_ptr<Widget> p(new Widget());
std::unique_ptr<Widget> q = std::move(p);   // 转移所有权；p 变空
// p 离开作用域/替换时自动 delete

// shared_ptr
auto sp = std::make_shared<Widget>();
auto sp2 = sp;                 // 引用计数 +1
sp2.reset();                   // 计数 -1

// weak_ptr 破除循环
std::weak_ptr<Widget> wp = sp;
if (auto s = wp.lock()) { /* 安全使用 */ }   // lock 提升为 shared

// 数组
std::unique_ptr<int[]> arr(new int[10]);
```

要点：
- 优先 `make_shared`（异常安全 + 单次分配）；`shared_ptr` 的裸 `new` 配合参数求值顺序有内存泄漏风险。
- `weak_ptr` 不拥有对象；用 `lock()` 判断对象是否存活。
- 循环引用（A↔B 相互 shared）会泄漏，用 weak 打断。

### 适用场景

- 所有需要“自动释放”的堆对象所有权。
- 工厂返回、跨函数转移所有权（`unique_ptr`）。
- 多所有者共享（缓存、图结构、`shared_ptr`）。
- 缓存/观察引用（`weak_ptr`）。

---

## 6. `<string>` — 字符串

**定位**：动态、安全、功能完备的字符串类型（`std::basic_string<char>` 特化）。

### 常用类型

- `std::string`（`basic_string<char>`）、`std::wstring`（`wchar_t`）。
- `std::string::size_type`（实为 `size_t`）、`std::string::npos`（极大值，表示“未找到”）。

### 关键接口（高频优先，加粗为必记）

- **`size()/length()/empty()`**：长度/判空。
- **`operator[] / at`**：字符访问（`at` 越界抛异常）。
- **`append(...)` / `operator+=`**：追加字符串/字符。
- **`find(str, pos=0)` / `rfind`**：子串查找，返回位置或 `npos`。
- **`substr(pos, len=npos)`**：取子串。
- **`c_str()` / `data()`**：C 风格 `const char*`（C++11 起均以 `\0` 结尾）。
- **`compare(...)`**：字典序比较。
- `replace`、`insert`、`erase`、`clear`、`resize`、`reserve`、`swap`。
- `find_first_of`、`find_last_of`、`find_first_not_of`：字符集查找。
- `stoi/stol/stoll/stoul/stof/stod/stolld/to_string`（C++11 数值转换）。
- `push_back`、`pop_back`、`front`、`back`。
- 迭代器与范围 for。

```cpp
#include <string>
std::string s = "Hello, World";
s.append("!");                        // "Hello, World!"
if (s.find("World") != std::string::npos) { }   // 判存在
auto sub = s.substr(7, 5);            // "World"
int n = std::stoi("42");              // 42
std::string num = std::to_string(3.14);
const char* c = s.c_str();            // C 接口互操作
```

### 适用场景

- 所有文本处理、日志、配置解析。
- C 接口转换（`c_str()`）。
- 格式化拼接（配合 `std::ostringstream`）。

---

## 7. `<sstream>` — 字符串流

**定位**：把字符串当 I/O 流处理，是“类型安全格式化”的经典手段。

### 常用类型

- `std::istringstream`：从字符串读。
- `std::ostringstream`：向字符串写。
- `std::stringstream`：读写。

### 关键接口

```cpp
#include <sstream>
std::ostringstream oss;
oss << "value=" << 42 << ", pi=" << 3.14159;
std::string out = oss.str();           // 取字符串

int a; double d; std::string token;
std::istringstream iss("10 3.5 hello");
iss >> a >> d >> token;                // 格式化读取

// 解析逗号分隔（简化）
std::string line = "x,17";
std::string key; int val; char comma;
std::istringstream(line) >> key >> comma >> val;
```

### 适用场景

- 类型安全的字符串拼接/格式化。
- 输入解析、协议字段拆分。
- 日志构建。

---

## 8. `<fstream>` — 文件流

**定位**：文件读写（文本与二进制）。

### 常用类型

- `std::ifstream`：读文件。
- `std::ofstream`：写文件。
- `std::fstream`：读写。

### 关键接口与模式

```cpp
#include <fstream>
// 写
std::ofstream out("data.txt", std::ios::out | std::ios::trunc);
if (out) { out << "line1\n" << 42 << '\n'; }

// 读（按行）
std::ifstream in("data.txt");
if (!in) { /* 打不开 */ }
std::string line;
while (std::getline(in, line)) { /* 处理 */ }

// 二进制读写
std::ofstream bout("bin.dat", std::ios::binary);
bout.write(reinterpret_cast<const char*>(&val), sizeof(val));
std::ifstream bin("bin.dat", std::ios::binary);
bin.read(reinterpret_cast<char*>(&val), sizeof(val));
```

打开模式：`ios::in`、`ios::out`、`ios::app`（追加）、`ios::trunc`（截断）、`ios::binary`。对象析构自动关闭文件（RAII）。

### 适用场景

- 配置文件读写、日志落盘。
- 二进制序列化、缓存持久化。
- 数据导入导出。

---

## 9. `<thread>` — 线程

**定位**：创建与管理执行线程（C++11 正式纳入标准并发）。

### 常用类型

- `std::thread`：线程对象（不可拷贝、可移动）。
- `std::this_thread::get_id()/sleep_for()/sleep_until()/yield()`。

### 关键接口

```cpp
#include <thread>
#include <chrono>

void work(int id) { /* ... */ }

std::thread t1(work, 1);                 // 启动线程
std::thread t2([] { /* lambda */ });     // 匿名函数

t1.join();      // 等待 t1 结束（阻塞）
if (t2.joinable()) t2.join();

// 不 join 又不 detach 的线程在析构时 std::terminate
std::thread t3(work, 3);
t3.detach();    // 分离，不等待（风险：生命周期需自行保障）
```

- `joinable()`：是否可 join（已启动且未 join/detach）。
- `join()` 阻塞等待；`detach()` 脱离。
- 启动时把 `this` 或引用传线程时，须保证线程存续期内对象存活（用值拷贝/`join` 前不销毁）。

### 适用场景

- 并行任务、后台工作线程。
- 生产者/消费者线程。
- 定时/异步处理。

---

## 10. `<mutex>` — 互斥锁

**定位**：互斥访问共享数据，消除数据竞争。

### 常用类型

| 类型 | 语义 |
|---|---|
| `std::mutex` | 基本互斥锁（不可递归） |
| `std::recursive_mutex` | 可被同一线程重复加锁 |
| `std::timed_mutex` | 支持带超时加锁 |
| `std::recursive_timed_mutex` | 递归+超时 |
| `std::lock_guard<M>` | RAII 自动加解锁（锁的生存期=作用域） |
| `std::unique_lock<M>` | 可手动 lock/unlock、可移动，配合条件变量 |

### 关键接口

```cpp
#include <mutex>
std::mutex m;
int counter = 0;

// lock_guard：最简、RAII
void inc() {
    std::lock_guard<std::mutex> lock(m);   // 加锁；离开作用域自动解锁
    ++counter;
}

// unique_lock：手动控制
std::unique_lock<std::mutex> ul(m);
ul.unlock();
/* 不持锁代码 */
ul.lock();
```

- 加锁后修改共享数据；避免在锁内做耗时操作。
- 死锁：多把锁须**一致顺序**加锁，或用 `std::lock(a,b,...)` 一次锁多把。

### 适用场景

- 保护共享状态（计数器、缓存、队列）。
- 临界区互斥。

---

## 11. `<condition_variable>` — 条件变量

**定位**：让线程等待某**条件成立**，避免忙等待轮询。

### 常用类型

- `std::condition_variable`：配合 `std::unique_lock<std::mutex>`。
- `std::condition_variable_any`：配合任意可锁类型。

### 关键接口与标准用法

```cpp
#include <condition_variable>
std::mutex m;
std::condition_variable cv;
bool ready = false;

// 等待方
std::unique_lock<std::mutex> lk(m);
cv.wait(lk, [&]{ return ready; });      // 用谓词避免伪唤醒(spurious wakeup)
// 条件满足，继续

// 通知方
{
    std::lock_guard<std::mutex> lk(m);
    ready = true;
}
cv.notify_one();       // 唤醒一个等待者；notify_all() 唤醒全部
```

要点：
- 必须持有同一把互斥锁才能 `wait`；等待期间锁被释放，条件满足后重新获取。
- **必须用谓词重载**防伪唤醒与漏唤醒。
- `wait_for`/`wait_until` 提供超时。

### 适用场景

- 生产者-消费者队列。
- 事件通知、任务就绪。
- 限时等待。

---

## 12. `<atomic>` — 原子操作

**定位**：无锁的原子类型，保证单个操作不可撕裂，并提供内存序。

### 常用类型

- `std::atomic<T>`（T 为内建标量/指针等）、`std::atomic<bool>`、`std::atomic<int>`、`std::atomic_flag`。
- 便捷别名：`std::atomic_int`、`std::atomic_bool` 等。
- `std::memory_order` 枚举。

### 关键接口

```cpp
#include <atomic>
std::atomic<int> cnt{0};
cnt.store(5);                 // 写
int v = cnt.load();           // 读
int old = cnt.fetch_add(1);   // 原子自增，返回旧值
bool ok = cnt.compare_exchange_weak(expected, desired);  // CAS

std::atomic<bool> flag{false};
flag.exchange(true);          // 交换

// 内存序
cnt.store(1, std::memory_order_release);
int r = cnt.load(std::memory_order_acquire);
```

- `is_lock_free()`：判断是否真正无锁（硬件支持）。
- 默认 `memory_order_seq_cst`（顺序一致）。
- 原子类型默认不可拷贝（可用 `load/store`）；`std::atomic<T>` 由 `is_trivially_copyable` 的 T 支持。

### 适用场景

- 原子计数器、标志位。
- 无锁数据结构（谨慎）。
- 发布-获取同步（见内存模型章节）。

---

## 13. `<functional>` — 函数对象

**定位**：可调用对象的统一抽象与绑定。

### 常用类型

- `std::function<Ret(Args...)>`：类型擦除的可调用对象。
- `std::bind` + `std::placeholders`：部分应用。
- 内建仿函数：`std::less`、`std::greater`、`std::plus`、`std::minus`、`std::multiplies`、`std::divides`、`std::equal_to`、`std::logical_and`。
- `std::ref`/`std::cref`：包装引用传给 `std::function`/`std::bind`。
- `std::mem_fn`：成员函数指针包装。

### 关键接口

```cpp
#include <functional>
std::function<int(int,int)> fn;
fn = [](int a,int b){ return a+b; };   // Lambda
fn = std::multiplies<int>();           // 仿函数
int r = fn(2,3);

auto add5 = std::bind( [](int a,int b){return a+b;}, 5, std::placeholders::_1);
add5(7);                               // 12

int x = 10;
std::function<int()> f = std::bind([](int& v){ return v; }, std::ref(x));
f();                                   // 10
```

### 适用场景

- 回调注册、策略注入、事件系统。
- 泛型算法的谓词/比较器。
- 延迟执行与函数表。

---

## 14. `<random>` — 随机数

**定位**：取代 C 风格 `rand()`，提供**可复现、分布可控**的高质量随机数。

### 核心组件

- **引擎（engine）**：生成均匀随机位/数的核心：`std::mt19937`（梅森旋转，默认推荐）、`std::mt19937_64`、`std::default_random_engine`。
- **分布（distribution）**：把引擎输出映射为特定分布：`std::uniform_int_distribution`、`std::uniform_real_distribution`、`std::normal_distribution`、`std::bernoulli_distribution` 等。
- `std::random_device`：真随机种子源。

### 关键接口

```cpp
#include <random>
std::random_device rd;                    // 真随机种子
std::mt19937 gen(rd());                   // 引擎（用随机种子）

std::uniform_int_distribution<int> dist(1, 100);
int r = dist(gen);                        // [1,100] 均匀整数

std::normal_distribution<double> nd(0.0, 1.0);  // 均值0标准差1
double x = nd(gen);
```

要点：引擎无状态重置即“随机数生成器对象”可复现（传相同种子得相同序列，利于调试）；一次分布 + 引擎复用优于每次新建。

### 适用场景

- 随机抽样、模拟、游戏。
- 洗牌（`std::shuffle` 需随机引擎，C++11）。
- 概率分布模拟。

---

## 15. `<limits>` — 类型极限

**定位**：查询内建数值类型的范围与性质（编译期/constexpr）。

### 核心组件

- `std::numeric_limits<T>` 类模板 + 特化。
- `std::numeric_limits<T>::is_specialized`、`is_integer`、`is_signed`、`is_exact`。
- 极值：`max()`、`min()`、`lowest()`、`epsilon()`、`digits`、`radix`。
- `denorm_min()`、`infinity()`、`quiet_NaN()`、`signaling_NaN()`。

### 关键接口

```cpp
#include <limits>
std::numeric_limits<int>::max();         // 2147483647
std::numeric_limits<double>::min();      // 最小正规格化数
std::numeric_limits<double>::lowest();   // 最小有限值（-max）
std::numeric_limits<double>::epsilon();  // 机器精度
```

- 用 `epsilon()` 做浮点比较容差：

```cpp
bool almost_eq(double a, double b) {
    return std::abs(a-b) <= std::numeric_limits<double>::epsilon() * std::max(std::abs(a), std::abs(b));
}
```

### 适用场景

- 跨平台数值上限判断。
- 浮点容差比较。
- 序列化边界检查。

---

## 16. 交叉使用与工程要点

### 16.1 高频组合

- **`<thread>` + `<mutex>` + `<condition_variable>`**：完整线程同步（见并发编程章节）。
- **`<chrono>` + `<thread>`**：`sleep_for`、`wait_for` 超时。
- **`<sstream>` + `<string>`**：格式化与解析。
- **`<memory>` + 容器**：`vector<unique_ptr<T>>` 对象池/所有权容器。
- **`<random>` + `<algorithm>`**：`std::shuffle` 洗牌。

### 16.2 实用误区提醒

| 库 | 误区 | 正确做法 |
|---|---|---|
| `<memory>` | `new`+`shared_ptr` 丢失计数 | 用 `make_shared` |
| `<string>` | 频繁拼接低效 | `reserve` + `+=` |
| `<chrono>` | 用 `system_clock` 计时 | 计时用 `steady_clock` |
| `<thread>` | 忘 join 导致 terminate | 用 RAII 或保证 join |
| `<mutex>` | 锁内做重活 | 最小临界区 |
| `<condition_variable>` | 无谓词 wait 伪唤醒 | 谓词重载 |
| `<random>` | 每次新建引擎 | 复用引擎 |
| `<atomic>` | 当成 volatile 用 | 明确内存序 |

---

> **进入下一章前自检**：能否用 `make_shared`/`unique_ptr` 管理对象生命周期？写出 `mutex`+`condition_variable` 的生产者消费者？区分 `steady_clock` 与 `system_clock`？下一步请阅读 `C++并发编程.md`。
