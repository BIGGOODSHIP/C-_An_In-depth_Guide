# C++ 并发编程

> 并发编程是多线程程序正确性、性能与可维护性的核心议题。本文件覆盖 C++11 标准并发的全部机制：线程、互斥、条件变量、原子、future/promise、线程安全模式与工程实践。严格限定在 C++11 及之前标准。

---

## 目录

1. 并发 vs 并行：概念与模型
2. 数据竞争与线程安全基础
3. 线程管理与 `std::thread`
4. 共享数据与互斥锁 `<mutex>`
5. 死锁与锁策略
6. 条件变量与生产者消费者
7. 原子操作与内存序 `<atomic>`
8. 任务式并发：`std::async` / `std::future` / `std::promise` / `std::packaged_task`
9. 线程局部存储 `thread_local`
10. 并发设计模式与线程安全容器策略
11. 性能与扩展性
12. 常见错误与调试

---

## 1. 并发 vs 并行：概念与模型

- **并发（concurrency）**：多个任务在时间上交错执行，逻辑上同时推进（单核通过时间片也可并发）。
- **并行（parallelism）**：多个任务在物理上同时执行（多核/多 CPU）。

### 1.1 内存模型基础

C++11 定义了多线程内存模型：**同一内存位置被多个线程访问、至少一方为写、且无同步关系 = 数据竞争（data race）= 未定义行为**。正确并发必须通过同步机制（互斥、原子、fence）建立 happens-before 关系。

### 1.2 并发的取舍

- **收益**：利用多核、隐藏 I/O 等待、提升吞吐。
- **成本**：同步开销、上下文切换、调试复杂度、死锁/活锁/竞态风险。
- 决策原则：**没有共享可变状态的并发最容易正确**（如函数式、消息传递）。

---

## 2. 数据竞争与线程安全基础

### 2.1 什么是线程安全

“线程安全”指对象在**多个线程并发调用**时仍保持不变量，不产生数据竞争。三个层面：

1. **无共享**（immutable/线程局部）：最安全。
2. **同步共享**（锁/原子保护）：正确但需精细设计。
3. **无锁算法**：极高难度，仅专家场景。

### 2.2 竞态（race condition）与数据竞争

- **竞态**：结果依赖线程执行交错顺序，导致逻辑错误。
- **数据竞争**：C++ 内存模型中的未定义行为（UB），可能表现为读脏数据、崩溃甚至程序行为完全异常。

```cpp
int counter = 0;
// 两个线程同时 ++counter —— 数据竞争（读改写非原子）
```

### 2.3 同步原语总览

| 机制 | 头文件 | 用途 |
|---|---|---|
| `std::mutex` 及变体 | `<mutex>` | 互斥临界区 |
| `std::condition_variable` | `<condition_variable>` | 条件等待/通知 |
| `std::atomic<T>` | `<atomic>` | 无锁原子操作 |
| `std::thread` | `<thread>` | 线程管理 |
| `std::async` / `std::future` | `<future>` | 任务式并发 |
| `thread_local` | — | 线程局部存储 |

---

## 3. 线程管理与 `std::thread`

### 3.1 创建与启动

```cpp
#include <thread>
std::thread t1(f);                 // 函数
std::thread t2(f, 1, "arg");       // 带参数
std::thread t3([] { /* lambda */ });
std::thread t4(std::move(t1));     // 移动构造（线程不可拷贝）
```

### 3.2 `join` / `detach`

- **`join()`**：阻塞等待线程结束；线程对象析构前若仍 `joinable` 而未 `join`/`detach`，调用 `std::terminate`。
- **`detach()`**：脱离，让线程后台运行；线程对象不再管理它。
- **`joinable()`**：判断能否 join。

```cpp
std::thread t(f);
t.join();      // 等待完成
```

**原则**：用 RAII 封装线程生命周期，确保异常路径也会 join，避免 terminate。

### 3.3 传参与生命周期

- 线程参数**默认拷贝/移动**；传引用须显式 `std::ref`。
- 传入 `this`/指针时，必须保证线程存续期内对象存活（通常用值拷贝或在主线程 join 前不销毁对象）。

```cpp
void f(int& x) { ++x; }
int v = 0;
std::thread t(f, std::ref(v));   // 显式引用
t.join();                        // v == 1
```

### 3.4 硬件并发度

```cpp
unsigned n = std::thread::hardware_concurrency();   // 逻辑核数（可能返回0）
```

---

## 4. 共享数据与互斥锁 `<mutex>`

### 4.1 锁类型

| 类型 | 语义 |
|---|---|
| `std::mutex` | 基础互斥（不可重入） |
| `std::recursive_mutex` | 可重入（同一线程可多次 lock） |
| `std::timed_mutex` | 支持 `try_lock_for`/`try_lock_until` |
| `std::lock_guard<M>` | RAII 锁（作用域解锁） |
| `std::unique_lock<M>` | 可手动控制、可移动、可配合条件变量 |
| `std::scoped_lock` | C++17 才有；C++11 用 `std::lock` |

### 4.2 RAII 加锁

```cpp
std::mutex m;
int counter = 0;

void inc() {
    std::lock_guard<std::mutex> lk(m);   // 加锁；出作用域自动解锁
    ++counter;                            // 保护临界区
}
```

- **临界区尽量小**：只保护共享数据的读改写。
- 锁持有期间不要调用可能回调/等待的代码（死锁风险）。
- 结构化锁定：**同一个对象的所有操作**都必须持同一把锁，否则锁形同虚设（对象不变量无法保持）。

### 4.3 读写分离

- C++11 标准库**没有**内置 `shared_mutex`（那是 C++17 的 `<shared_mutex>`）。C++11 如需读写锁，需自定义或借助第三方。
- 简单场景用互斥即可；读多写少的高频场景，C++11 下可考虑原子/副本策略（详见第 11 章）。

### 4.4 一次性多锁

```cpp
std::mutex a, b;
void f() {
    std::lock(a, b);               // 同时锁定，避免死锁
    std::lock_guard<std::mutex> la(a, std::adopt_lock);
    std::lock_guard<std::mutex> lb(b, std::adopt_lock);
    // 共享 a、b 的临界区
}
```

---

## 5. 死锁与锁策略

### 5.1 死锁条件

死锁发生的四个必要条件：互斥、持有并等待、不可剥夺、循环等待。消除任一即可预防。

### 5.2 预防与检测

- **统一加锁顺序**：所有线程按相同顺序获取多把锁。
- **用 `std::lock` 一次获取多把锁**（避免顺序不一致）。
- **避免锁内等待/锁内回调**。
- 递归锁用于“同一线程重复进入”的复杂递归，但滥用会掩盖设计问题。
- 用 `try_lock` 系列 + 超时在无法满足时放弃（退避重试），降低死锁概率。

### 5.3 活锁与饿死

- 活锁：线程反复尝试但始终无法推进。
- 饿死：某线程长期得不到资源。
- 通常通过合理的公平策略/重试上限缓解。

---

## 6. 条件变量与生产者消费者

### 6.1 标准用法

```cpp
#include <condition_variable>
std::mutex m;
std::condition_variable cv;
std::queue<int> q;      // 共享队列

// 生产者
void produce(int v) {
    {
        std::lock_guard<std::mutex> lk(m);
        q.push(v);
    }
    cv.notify_one();    // 唤醒一个等待者
}

// 消费者
int consume() {
    std::unique_lock<std::mutex> lk(m);
    cv.wait(lk, [&]{ return !q.empty(); });   // 谓词防伪唤醒
    int v = q.front(); q.pop();
    return v;
}
```

### 6.2 关键规则

- `wait` 必须在持有 `unique_lock`（非 lock_guard）时调用。
- 等待期间锁被**释放**，条件满足后**重新获取**。
- **必须用谓词重载** `wait(lk, pred)`，防伪唤醒（spurious wakeup）与漏唤醒。
- 通知前不要求持锁；用 `notify_one` 唤醒单个，`notify_all` 唤醒全部。

### 6.3 超时等待

```cpp
bool ready = cv.wait_for(lk, std::chrono::seconds(1),
                         []{ return condition; });
```

返回 `bool` 表示条件是否在时限内满足，用于超时分支处理。

---

## 7. 原子操作与内存序 `<atomic>`

### 7.1 原子类型

```cpp
std::atomic<int> cnt{0};
cnt++;                     // 原子自增（fetch_add）
int v = cnt.load();        // 原子读
cnt.store(10);             // 原子写
int old = cnt.exchange(5); // 交换，返回旧值
```

- 原子操作保证：单个操作不可被其他线程撕裂。
- `is_lock_free()`：检查是否真正硬件无锁。
- 原子类型不可拷贝/移动（用 load/store）。

### 7.2 内存序（复习）

| 枚举 | 语义 |
|---|---|
| `relaxed` | 仅原子性 |
| `acquire` / `release` | 发布-获取同步 |
| `acq_rel` | RMW |
| `seq_cst` | 默认，全序 |

```cpp
// 发布-获取模式
std::atomic<bool> ready{false};
int data = 0;

// 线程A
data = 42;
ready.store(true, std::memory_order_release);

// 线程B
while (!ready.load(std::memory_order_acquire)) {}
use(data);   // 保证看到 42
```

### 7.3 CAS 与无锁

```cpp
std::atomic<int> a{0};
int expected = 0;
if (a.compare_exchange_strong(expected, 1)) {
    // 成功：a 由 0 变为 1
} else {
    // 失败：expected 被更新为当前值
}
```

无锁编程（CAS 循环、无锁队列）正确性极难；工程上**优先锁**，仅在有明确性能瓶颈且严格验证时使用。

---

## 8. 任务式并发：`<future>`

任务式并发把“异步执行一段代码并取回结果”抽象化，隐藏线程管理细节。

### 8.1 `std::async` + `std::future`

```cpp
#include <future>
auto f = std::async(std::launch::async, [] { return compute(); });
// 并行做其他事
int result = f.get();    // 阻塞直到结果就绪
```

- `std::launch::async`：强制新线程执行。
- `std::launch::deferred`：惰性，`get()` 时才在当前线程执行。
- 默认策略由实现选择（可能是 deferred，注意）。

### 8.2 `std::promise` / `std::future` 配对

promise 提供“设置结果”的一端，future 提供“获取结果”的一端，跨线程传值。

```cpp
std::promise<int> p;
std::future<int> fut = p.get_future();

std::thread t([&p] { p.set_value(compute()); });
int r = fut.get();        // 阻塞直到 set_value
t.join();
```

- `set_value` 设置结果，`set_exception` 传异常（`get` 时重新抛出）。
- `future` 不可拷贝、可移动；`get()` 只能调用一次。

### 8.3 `std::packaged_task`

把可调用对象与 future 绑定，可调度执行：

```cpp
std::packaged_task<int()> task([] { return 7; });
auto fut = task.get_future();
std::thread(std::move(task)).detach();   // 在线程中执行
int r = fut.get();
```

### 8.4 `wait_for` / `wait_until`

```cpp
if (fut.wait_for(std::chrono::milliseconds(100)) == std::future_status::ready) {
    int r = fut.get();
} else {
    // 超时未就绪
}
```

### 8.5 任务式 vs 线程式

| 维度 | `std::async`/future | 裸 `std::thread` |
|---|---|---|
| 抽象 | 结果导向，隐藏线程 | 线程直接管理 |
| 同步 | `get` 自动同步 | 需手工 join |
| 粒度 | 单次任务 | 常驻工作线程 |
| 适用 | 一次性并行任务 | 后台循环、长驻线程 |

---

## 9. 线程局部存储 `thread_local`（C++11）

```cpp
thread_local int tls_counter = 0;   // 每个线程独立一份
```

- 每个线程拥有独立副本，无共享、无竞争。
- 用于：线程专属缓冲、随机引擎、缓存、日志上下文。
- 生命周期为线程存续期；析构在线程退出时执行。

```cpp
// 每线程随机引擎
thread_local std::mt19937 gen(std::random_device{}());
```

---

## 10. 并发设计模式与线程安全容器策略

### 10.1 经典模式

| 模式 | 说明 |
|---|---|
| 生产者-消费者 | 队列 + 锁 + 条件变量解耦生产与消费 |
| 读-写分离 | 副本/不可变快照，读者无锁 |
| 任务池（thread pool） | 常驻工作线程消费任务队列 |
| 单例 | 局部静态 + magic static（线程安全一次初始化） |
| Actor / 消息传递 | 线程通过队列通信，无共享可变状态 |

### 10.2 线程安全容器的构造

标准容器**本身不线程安全**。三种路线：

1. **粗粒度封装**：所有公有接口统一加锁（简单，吞吐受限）。
2. **细粒度内部锁**：分段加锁、无锁实现（复杂）。
3. **不可变快照**：读者用 `std::shared_ptr<const T>` 快照，写者构建副本后替换（COW/副本读取）。

```cpp
// 不可变快照模式
class Registry {
public:
    std::shared_ptr<const std::map<std::string,int>> snapshot() const {
        std::lock_guard<std::mutex> lk(m_);
        return data_;          // 返回不可变副本的共享指针
    }
    void update(const std::string& k, int v) {
        auto fresh = std::make_shared<std::map<std::string,int>>(*data_);  // 先拷贝
        (*fresh)[k] = v;
        std::lock_guard<std::mutex> lk(m_);
        data_ = fresh;         // 原子替换
    }
private:
    mutable std::mutex m_;
    std::shared_ptr<std::map<std::string,int>> data_;
};
```

读者 `snapshot()` 拿到不可变快照后可**无锁安全遍历**，写者只锁替换指针的短暂临界区。这是读多写少场景的高效方案。

### 10.3 单例的线程安全

```cpp
class Logger {
public:
    static Logger& instance() {
        static Logger inst;    // C++11 magic static：线程安全一次初始化
        return inst;
    }
    Logger(const Logger&) = delete;
    Logger& operator=(const Logger&) = delete;
private:
    Logger() {}
};
```

---

## 11. 性能与扩展性

### 11.1 开销层级（从低到高）

1. 无共享 / 线程局部：零同步。
2. 原子操作：无锁，但存在缓存行争用。
3. 互斥锁：进入内核（或 futex 优化），阻塞/唤醒有成本。
4. 条件变量：涉及调度与唤醒。
5. 线程创建/销毁：最贵，避免频繁创建。

### 11.2 扩展性要点

- **线程数** ≈ 计算核心数（CPU 密集）；I/O 密集可略多。
- **避免伪共享（false sharing）**：不同线程修改相邻变量，因缓存行共享而互相失效。用对齐（`alignas(64)`）把高频更新变量隔离到不同缓存行。

```cpp
struct alignas(64) PerThread {
    int value;   // 独占缓存行，避免伪共享
};
```

- **临界区最小化**、**无锁读路径**、**批量处理减少唤醒**。

### 11.3 度量与验证

- 用 `steady_clock` 测量。
- 用 AddressSanitizer/ThreadSanitizer（TSan）检测数据竞争（见工具链章节）。
- 压测确认扩展性（加速比）。

---

## 12. 常见错误与调试

| 错误 | 后果 | 对策 |
|---|---|---|
| 数据竞争 | UB（脏读/崩溃） | 互斥/原子保护 |
| 忘 join | `std::terminate` | RAII 线程包装 |
| 锁内等待 | 死锁 | 最小临界区 |
| 多锁顺序不一致 | 死锁 | `std::lock`/统一顺序 |
| wait 无谓词 | 伪唤醒 | 谓词重载 |
| 返回局部引用给线程 | 悬垂 | 值拷贝/join 前保活 |
| 原子误用 | 无同步语义 | 明确内存序 |
| 频繁建线程 | 性能差 | 线程池 |
| 假共享 | 扩展性差 | 缓存行对齐 |

### 12.1 调试工具

- **ThreadSanitizer**（`-fsanitize=thread`）：检测数据竞争。
- **AddressSanitizer**（`-fsanitize=address`）：检测内存错误。
- GDB 多线程调试：`thread apply all bt`、`info threads`、`set scheduler-locking`。

---

> **进入下一章前自检**：能否写出生产者-消费者的正确实现（含谓词 wait）？解释死锁四条件与预防？说明 release-acquire 同步与 `std::async` 的用法？下一步请阅读 `C++网络编程.md`。
