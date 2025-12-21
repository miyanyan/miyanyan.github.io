---
title:      "SPSC的性能优化"
description: SPSC的代码里哪些地方可以优化？
date:       2025-12-08
categories: [c++]
tag: [c++, std]
---

## 背景

SPSC是Single Producer Single Consumer的简称，即单生产者单消费者模型。在多线程编程中，SPSC模型是一种常见的模型，它要求一个线程只负责生产数据，另一个线程只负责消费数据。

在引入优化点之前，我们要先达成几点共识：
* SPSC一般情况下限制最大的buffer大小，一开始就分配足够的空间，我们假定非特殊情况buffer是不会被填满的，如果buffer经常被填满那你的系统就有问题了。当然像[readerwriterqueue](https://github.com/cameron314/readerwriterqueue)就是支持扩容的，引入内存分配就变得复杂起来，先不考虑。
* 生产者只push，消费者只pop
* 以非阻塞版本的push和pop来进行比较优化

## std::queue + std::mutex

首先引入基础版本的SPSC模型，即使用std::queue和std::mutex。
```c++
#include <queue>
#include <atomic>
#include <mutex>

template<typename T>
class SPSC_Mutex
{
public:
    SPSC_Mutex() = default;

    bool push(T&& value)
    {
        std::lock_guard<std::mutex> lk(m_mutex);
        m_queue.push(std::move(value));
        return true;
    }
    bool pop(T& value)
    {
        std::lock_guard<std::mutex> lk(m_mutex);
        if (m_queue.empty()) {
            return false;
        }
        value = std::move(m_queue.front());
        m_queue.pop();
        return true;
    }

private:
    std::mutex m_mutex;
    std::queue<T> m_queue;
};
```

## 优化点1: 使用环形缓冲区替代std::queue，使用atomic操作替代锁

基于上文buffer可以设置最大大小的假设，我们可以考虑使用环形缓冲区来替代std::queue。环形缓冲区的介绍参考[wiki:Circular_buffer](https://en.wikipedia.org/wiki/Circular_buffer)，基于此：  
* 判空，empty = (readIndex == writeIndex)
* 判满，这里可以牺牲一个槽位(slot)防止判空和判满的歧义，full = ((writeIndex + 1) % capacity == readIndex)
* 剩余空间，size = (writeIndex - readIndex + capacity) % capacity
* push, writeIndex = (writeIndex + pushSize) % capacity，注意，由于牺牲了一个槽位，所以这里的pushSize不能超过**剩余空间-1**
* pop, readIndex = (readIndex + popSize) % capacity

基于环形缓冲区的特点，我们可以考虑使用atomic操作来替代锁。
* 生产者只修改writeIndex，消费者只修改readIndex，因此可以使用atomic操作来保证线程安全。
* 对于不同的操作，可以采用不同的内存序:
  * push, 由于writeIndex只有生产者修改，因此可以使用std::memory_order_relaxed来进行读取，使用std::memory_order_release来进行写入。
  * pop, 由于readIndex只有消费者修改，因此可以使用std::memory_order_relaxed来进行读取，使用std::memory_order_release来进行写入。
  * push/pop里读取对方的index时，使用std::memory_order_acquire来进行读取。
  * 判空/判满/剩余空间，生产者消费者线程都可以调用，全部采用std::memory_order_acquire来进行读取。

C++ 的六种内存序（从弱到强）：C++11 定义了 6 种内存序，按照约束强度从弱到强排序，性能开销通常也随之增加。
1. std::memory_order_relaxed (松散序)
   * 约束：最弱。只保证原子性（操作不可分割），不保证任何顺序。
   * 行为：不同线程看到的变量更新顺序可能完全不同。
   * 场景：统计计数器（如 std::shared_ptr 的引用计数增加）。只需保证“不撕裂写入”，不在乎谁先谁后。
2. std::memory_order_consume (消费序)
   * 约束：比 Acquire 弱。只保证有数据依赖关系的指令不乱序。
   * 现状：极难正确使用，编译器支持糟糕（通常直接提升为 Acquire）。C++ 标准委员会正在修订它。
   * 建议：不要使用，直接用 Acquire 代替。
3. std::memory_order_acquire (获取序)
   * 操作：Load（读取）。
   * 约束：后面的读写不能排到前面。
   * 场景：互斥锁的加锁（Lock）、读取信号量、消费者读取 tail 指针。
4. std::memory_order_release (释放序)
   * 操作：Store（写入）。
   * 约束：前面的读写不能排到后面。
   * 场景：互斥锁的解锁（Unlock）、释放信号量、生产者更新 head 指针。
5. std::memory_order_acq_rel (获取释放序)
   * 操作：Read-Modify-Write (如 fetch_add, exchange, compare_exchange)。
   * 约束：兼具 Acquire 和 Release 的双重效果。既是输入的屏障，也是输出的屏障。
   * 场景：原子操作（如 fetch_add, exchange, compare_exchange）。CAS 操作。多个线程同时读写同一个原子变量进行同步时（例如实现一个自旋锁）。
6. std::memory_order_seq_cst (顺序一致性)
   * 约束：最强，也是 C++ 原子操作的默认选项。
   * 行为：包含 Acquire/Release 的所有语义。全局全序：所有线程看到的“所有 Seq_Cst 操作的顺序”是一模一样的。
   * 代价：在 x86 上通常也是廉价的（类似 Acquire/Release），但在 ARM/PowerPC 等弱内存模型架构上，需要插入很重的内存屏障指令（Memory Fence），严重影响性能。
   * 场景：新手默认使用。逻辑极其复杂，Acquire/Release 搞不定时。需要保证全局单一顺序时。

最终实现如下(folly的实现)：
```c++
template<typename T>
class SPSC1
{
public:
    explicit SPSC1(size_t minCapacity)
        : m_capacity(minCapacity)
        , m_buffer(static_cast<T*>(std::malloc(m_capacity * sizeof(T))))
    {
        if (!m_buffer) {
            throw std::bad_alloc();
        }
    }

    ~SPSC1()
    {
        // We need to destruct anything that may still exist in our queue.
        // (No real synchronization needed at destructor time: only one
        // thread can be doing this.)
        if (!std::is_trivially_destructible<T>::value) {
            size_t readIndex = m_consumerData.readIndex;
            size_t endIndex = m_producerData.writeIndex;
            while (readIndex != endIndex) {
                m_buffer[readIndex].~T();
                readIndex++;
                if (readIndex >= m_capacity) {
                    readIndex = 0;
                }
            }
        }

        std::free(m_buffer);
    }

    SPSC1(const SPSC1&) = delete;
    SPSC1& operator=(const SPSC1&) = delete;
    SPSC1(SPSC1&&) = delete;
    SPSC1& operator=(SPSC1&&) = delete;

    template<class... Args>
    bool push(Args&&... args)
    {
        size_t currentWriteIndex = m_producerData.writeIndex.load(std::memory_order_relaxed);
        size_t currentReadIndex = m_consumerData.readIndex.load(std::memory_order_acquire);
        size_t availableSpace = getAvailableSpace(currentWriteIndex, currentReadIndex);

        if (availableSpace < 1) {
            return false;
        }

        new (&m_buffer[currentWriteIndex]) T(std::forward<Args>(args)...);

        size_t nextWriteIndex = currentWriteIndex + 1;
        if (nextWriteIndex >= m_capacity) {
            nextWriteIndex = 0;
        }
        m_producerData.writeIndex.store(nextWriteIndex, std::memory_order_release);
        return true;
    }

    bool pop(T& output)
    {
        size_t currentReadIndex = m_consumerData.readIndex.load(std::memory_order_relaxed);
        size_t currentWriteIndex = m_producerData.writeIndex.load(std::memory_order_acquire);
        size_t availableSamples = calculateSize(currentWriteIndex, currentReadIndex);

        if (availableSamples < 1) {
            return false;
        }

        output = std::move(m_buffer[currentReadIndex]);
        m_buffer[currentReadIndex].~T();

        size_t nextReadIndex = currentReadIndex + 1;
        if (nextReadIndex >= m_capacity) {
            nextReadIndex = 0;
        }
        m_consumerData.readIndex.store(nextReadIndex, std::memory_order_release);
        return true;
    }

    size_t capacity() const { return m_capacity - 1; }

    bool isEmpty() const
    {
        return m_producerData.writeIndex.load(std::memory_order_acquire) ==
               m_consumerData.readIndex.load(std::memory_order_acquire);
    }
    bool isFull() const
    {
        auto nextWriteIndex = m_producerData.writeIndex.load(std::memory_order_acquire) + 1;
        if (nextWriteIndex >= m_capacity) {
            nextWriteIndex = 0;
        }
        return m_consumerData.readIndex.load(std::memory_order_acquire) == nextWriteIndex;
    }

private:
    size_t calculateSize(size_t writeIndex, size_t readIndex) const
    {
        if (writeIndex >= readIndex) return writeIndex - readIndex;
        return m_capacity - readIndex + writeIndex;
    }
    size_t getAvailableSpace(size_t writeIndex, size_t readIndex) const
    {
        return m_capacity - 1 - calculateSize(writeIndex, readIndex);
    }

    using AtomicIndex = std::atomic<size_t>;

    const size_t m_capacity;
    T* const m_buffer;

    struct
    {
        AtomicIndex readIndex = {0};
    } m_consumerData;

    struct
    {
        AtomicIndex writeIndex = {0};
    } m_producerData;
};
```

## 优化点2: 对齐cacheline防止false sharing
[wiki:False_sharing](https://en.wikipedia.org/wiki/False_sharing)

如果你看了[ProducerConsumerQueue](https://github.com/facebook/folly/blob/main/folly/ProducerConsumerQueue.h#L176)的实现，你可能已经注意到了hardware_destructive_interference_size，它在folly里的定义如下：
```c++
#if defined(__cpp_lib_hardware_interference_size)

//  GCC unconditionally warns about uses of the std's interference-size
//  constants, on the basis that their uses in public ABIs is likely broken:
//
//    its value can vary between compiler versions or with different ‘-mtune’
//    or ‘-mcpu’ flags; if this use is part of a public ABI, change it to
//    instead use a constant variable you define
//
//  For now, these remain theoretical concerns in the expected scenario, where
//  all of the application is built together with the same compiler options.
FOLLY_PUSH_WARNING
FOLLY_GCC_DISABLE_WARNING("-Winterference-size")

constexpr std::size_t hardware_constructive_interference_size =
    std::hardware_constructive_interference_size;

constexpr std::size_t hardware_destructive_interference_size =
    std::hardware_destructive_interference_size;

FOLLY_POP_WARNING

#else

//  Memory locations within the same cache line are subject to destructive
//  interference, also known as false sharing, which is when concurrent
//  accesses to these different memory locations from different cores, where at
//  least one of the concurrent accesses is or involves a store operation,
//  induce contention and harm performance.
//
//  Microbenchmarks indicate that pairs of cache lines also see destructive
//  interference under heavy use of atomic operations, as observed for atomic
//  increment on Sandy Bridge.
//
//  We assume a cache line size of 64, so we use a cache line pair size of 128
//  to avoid destructive interference.
//
//  mimic: std::hardware_destructive_interference_size, C++17
constexpr std::size_t hardware_destructive_interference_size =
    (kIsArchArm || kIsArchS390X) ? 64 : 128;
```

这里关于cacheline的大小，不同的设备架构可能有不同的大小，大部分情况下为64，folly在c++17以上支持std::hardware_destructive_interference_size时以此为默认值，否则以128为默认值。std::hardware_destructive_interference_size的值又得视情况而定，比如MSVC的源码里就直接写成64：
```c++
#if defined(_M_IX86) || defined(_M_X64) || defined(_M_ARM) || defined(_M_ARM64)
_EXPORT_STD inline constexpr size_t hardware_constructive_interference_size = 64;
_EXPORT_STD inline constexpr size_t hardware_destructive_interference_size  = 64;
#else // ^^^ supported hardware / unsupported hardware vvv
#error Unsupported architecture
#endif // ^^^ unsupported hardware ^^^
```
gcc/clang的情况则各式各样：https://github.com/llvm/llvm-project/pull/89446#issuecomment-2070649367

关于cacheline，folly的注释里还提及相邻的cacheline也可能对性能产生影像，Intel CPU可能会加载相邻的cacheline
> Intel's optimization manual does describe the L2 spatial prefetcher in SnB-family CPUs. Yes, it tries to complete 128B-aligned pairs of 64B lines, when there's spare memory bandwidth (off-core request tracking slots) when the first line is getting pulled in.

这里提到的是SnB-family CPUs，stackoverflow上有人提到确实有这种优化，但我在自己的intel和amd电脑上未观察到cacheline在128时比64有明显提升的情况。

参考：
* https://stackoverflow.com/questions/72126606/should-the-cache-padding-size-of-x86-64-be-128-bytes
* https://stackoverflow.com/questions/39680206/understanding-stdhardware-destructive-interference-size-and-stdhardware-cons

综上，cacheline应该怎么选择？我建议直接使用std::hardware_destructive_interference_size的大小，除非你的benchmark能明显看到性能提升，目前的绝大多数开源库也都是这么做的:  
* folly，c++17以上使用标准库，否则默认128
* [SPSCQueue](https://github.com/rigtorp/SPSCQueue/blob/1053918dbd251fbff69b24ef27fa5d51c29ec2af/include/rigtorp/SPSCQueue.h#L210-L215)，c++17以上使用标准库，否则默认64
* [readerwriterqueue](https://github.com/cameron314/readerwriterqueue/blob/master/readerwriterqueue.h)，使用宏来控制，默认64
* [atomic_queue](https://github.com/max0x7ba/atomic_queue/blob/master/include/atomic_queue/defs.h)，分不同的架构设置不同的大小，默认64

最终基于cacheline的优化代码，只需要设置读写索引的对齐大小为cacheline大小：
```c++
#pragma once

#include <new>

// This block handles a GCC-specific warning about ABI stability.
#if defined(__GNUC__) && !defined(__clang__)
#    pragma GCC diagnostic push
#    pragma GCC diagnostic ignored "-Winterference-size"
#endif
#if defined(__cpp_lib_hardware_interference_size)
inline constexpr size_t kCacheLineSize = std::hardware_destructive_interference_size;
#else
inline constexpr size_t kCacheLineSize = 64;
#endif
#if defined(__GNUC__) && !defined(__clang__)
#    pragma GCC diagnostic pop
#endif
```
以上为kCacheLineSize的定义，后面出现的所有`kCacheLineSize`都是指这个值。

```c++
template<typename T>
class SPSC2
{
public:
    explicit SPSC2(size_t minCapacity)
        : m_capacity(minCapacity)
        , m_buffer(static_cast<T*>(std::malloc(m_capacity * sizeof(T))))
    {
        if (!m_buffer) {
            throw std::bad_alloc();
        }
    }

    ~SPSC2()
    {
        // We need to destruct anything that may still exist in our queue.
        // (No real synchronization needed at destructor time: only one
        // thread can be doing this.)
        if (!std::is_trivially_destructible<T>::value) {
            size_t readIndex = m_consumerData.readIndex;
            size_t endIndex = m_producerData.writeIndex;
            while (readIndex != endIndex) {
                m_buffer[readIndex].~T();
                readIndex++;
                if (readIndex >= m_capacity) {
                    readIndex = 0;
                }
            }
        }

        std::free(m_buffer);
    }

    SPSC2(const SPSC2&) = delete;
    SPSC2& operator=(const SPSC2&) = delete;
    SPSC2(SPSC2&&) = delete;
    SPSC2& operator=(SPSC2&&) = delete;

    template<class... Args>
    bool push(Args&&... args)
    {
        size_t currentWriteIndex = m_producerData.writeIndex.load(std::memory_order_relaxed);
        size_t currentReadIndex = m_consumerData.readIndex.load(std::memory_order_acquire);
        size_t availableSpace = getAvailableSpace(currentWriteIndex, currentReadIndex);

        if (availableSpace < 1) {
            return false;
        }

        new (&m_buffer[currentWriteIndex]) T(std::forward<Args>(args)...);

        size_t nextWriteIndex = currentWriteIndex + 1;
        if (nextWriteIndex >= m_capacity) {
            nextWriteIndex = 0;
        }
        m_producerData.writeIndex.store(nextWriteIndex, std::memory_order_release);
        return true;
    }

    bool pop(T& output)
    {
        size_t currentReadIndex = m_consumerData.readIndex.load(std::memory_order_relaxed);
        size_t currentWriteIndex = m_producerData.writeIndex.load(std::memory_order_acquire);
        size_t availableSamples = calculateSize(currentWriteIndex, currentReadIndex);

        if (availableSamples < 1) {
            return false;
        }

        output = std::move(m_buffer[currentReadIndex]);
        m_buffer[currentReadIndex].~T();

        size_t nextReadIndex = currentReadIndex + 1;
        if (nextReadIndex >= m_capacity) {
            nextReadIndex = 0;
        }
        m_consumerData.readIndex.store(nextReadIndex, std::memory_order_release);
        return true;
    }

    size_t capacity() const { return m_capacity - 1; }

    bool isEmpty() const
    {
        return m_producerData.writeIndex.load(std::memory_order_acquire) ==
               m_consumerData.readIndex.load(std::memory_order_acquire);
    }
    bool isFull() const
    {
        auto nextWriteIndex = m_producerData.writeIndex.load(std::memory_order_acquire) + 1;
        if (nextWriteIndex >= m_capacity) {
            nextWriteIndex = 0;
        }
        return m_consumerData.readIndex.load(std::memory_order_acquire) == nextWriteIndex;
    }

private:
    size_t calculateSize(size_t writeIndex, size_t readIndex) const
    {
        if (writeIndex >= readIndex) return writeIndex - readIndex;
        return m_capacity - readIndex + writeIndex;
    }
    size_t getAvailableSpace(size_t writeIndex, size_t readIndex) const
    {
        return m_capacity - 1 - calculateSize(writeIndex, readIndex);
    }

    using AtomicIndex = std::atomic<size_t>;

    const size_t m_capacity;
    T* const m_buffer;

    struct alignas(kCacheLineSize)
    {
        AtomicIndex readIndex = {0};
    } m_consumerData;

    struct alignas(kCacheLineSize)
    {
        AtomicIndex writeIndex = {0};
    } m_producerData;
};
```

## 优化点3: 缓存读写位置
参考：https://rigtorp.se/ringbuffer/

缓存读写位置的优化，可以减少生产者和消费者线程的交互，提高缓存命中率。
* 生产者维护自己的写位置，缓存的读位置，每次push计算可用空间，只有空间不足的情况下才会访问消费者更改的读位置，否则空间足够完全可以直接push
* 消费者维护自己的读位置，缓存的写位置，每次pop计算已用空间，只有空间不足的情况下才会访问生产者更改的写位置，否则空间足够完全可以直接pop

最终的基于索引缓存的成员变量
```c++
template<typename T>
class SPSC3
{
public:
    explicit SPSC3(size_t minCapacity)
        : m_capacity(minCapacity)
        , m_buffer(static_cast<T*>(std::malloc(m_capacity * sizeof(T))))
    {
        if (!m_buffer) {
            throw std::bad_alloc();
        }
    }

    ~SPSC3()
    {
        // We need to destruct anything that may still exist in our queue.
        // (No real synchronization needed at destructor time: only one
        // thread can be doing this.)
        if (!std::is_trivially_destructible<T>::value) {
            size_t readIndex = m_consumerData.readIndex;
            size_t endIndex = m_producerData.writeIndex;
            while (readIndex != endIndex) {
                m_buffer[readIndex].~T();
                readIndex++;
                if (readIndex >= m_capacity) {
                    readIndex = 0;
                }
            }
        }

        std::free(m_buffer);
    }

    SPSC3(const SPSC3&) = delete;
    SPSC3& operator=(const SPSC3&) = delete;
    SPSC3(SPSC3&&) = delete;
    SPSC3& operator=(SPSC3&&) = delete;

    template<class... Args>
    bool push(Args&&... args)
    {
        size_t currentWriteIndex = m_producerData.writeIndex.load(std::memory_order_relaxed);
        size_t availableSpace = getAvailableSpace(currentWriteIndex, m_producerData.readIndexCache);

        if (availableSpace < 1) {
            // if not enough space, refresh the cached read index and check again
            m_producerData.readIndexCache = m_consumerData.readIndex.load(std::memory_order_acquire);
            availableSpace = getAvailableSpace(currentWriteIndex, m_producerData.readIndexCache);
            if (availableSpace < 1) {
                return false; // not enough space
            }
        }

        new (&m_buffer[currentWriteIndex]) T(std::forward<Args>(args)...);

        size_t nextWriteIndex = currentWriteIndex + 1;
        if (nextWriteIndex >= m_capacity) {
            nextWriteIndex = 0;
        }
        m_producerData.writeIndex.store(nextWriteIndex, std::memory_order_release);
        return true;
    }

    bool pop(T& output)
    {
        size_t currentReadIndex = m_consumerData.readIndex.load(std::memory_order_relaxed);
        size_t availableSamples = calculateSize(m_consumerData.writeIndexCache, currentReadIndex);

        if (availableSamples < 1) {
            m_consumerData.writeIndexCache = m_producerData.writeIndex.load(std::memory_order_acquire);
            availableSamples = calculateSize(m_consumerData.writeIndexCache, currentReadIndex);
            if (availableSamples < 1) {
                return false;  // not enough samples
            }
        }

        output = std::move(m_buffer[currentReadIndex]);
        m_buffer[currentReadIndex].~T();

        size_t nextReadIndex = currentReadIndex + 1;
        if (nextReadIndex >= m_capacity) {
            nextReadIndex = 0;
        }
        m_consumerData.readIndex.store(nextReadIndex, std::memory_order_release);
        return true;
    }

    size_t capacity() const { return m_capacity - 1; }

    bool isEmpty() const
    {
        return m_producerData.writeIndex.load(std::memory_order_acquire) ==
               m_consumerData.readIndex.load(std::memory_order_acquire);
    }
    bool isFull() const
    {
        auto nextWriteIndex = m_producerData.writeIndex.load(std::memory_order_acquire) + 1;
        if (nextWriteIndex >= m_capacity) {
            nextWriteIndex = 0;
        }
        return m_consumerData.readIndex.load(std::memory_order_acquire) == nextWriteIndex;
    }

private:
    size_t calculateSize(size_t writeIndex, size_t readIndex) const
    {
        if (writeIndex >= readIndex) return writeIndex - readIndex;
        return m_capacity - readIndex + writeIndex;
    }
    size_t getAvailableSpace(size_t writeIndex, size_t readIndex) const
    {
        return m_capacity - 1 - calculateSize(writeIndex, readIndex);
    }

    using AtomicIndex = std::atomic<size_t>;

    const size_t m_capacity;
    T* const m_buffer;

    struct alignas(kCacheLineSize)
    {
        AtomicIndex readIndex = {0};
        size_t writeIndexCache = {0};
    } m_consumerData;

    struct alignas(kCacheLineSize)
    {
        AtomicIndex writeIndex = {0};
        size_t readIndexCache = {0};
    } m_producerData;
};
```
实测下来，相比于没有索引缓存的版本，吞吐量有明显提升。

## 优化点4: 利用无符号数的回绕特性和2的幂次mask减少分支判断和取模操作

可以发现，folly版本使用了一个空槽来防止判空和判满的歧义，并且每次更新读写索引时，需要添加一个if来判断是否将索引变回0。同理，如果每次push或pop是批量操作，也需要添加if判断或者取模操作。

我们可以利用size_t的回绕特性，以及将buffer大小设置为2的幂次来减少分支判断和取模操作：
* 读写索引一直累加，直到超过size_t最大值，然后回绕到0
* 设置capacity为2的幂次，这样可以用位运算来代替取模操作
* 当前大小：writeIndex - readIndex，利用size_t的回绕特性，可以直接计算当前大小
* 判空：readIndex == writeIndex
* 判满：writeIndex - readIndex == capacity，可以发现不需要空一个槽位了！这里基于的假设是，申请的buffer的最大值是永远不可能超过size_t的最大值的，申请不到这么大的内存。
* 获取真实的物理索引即对应到buffer里的索引，pyhsicalIndex = index & (capacity - 1)
* push/pop，直接累加读写索引，更新的读写数据位置则使用上一条的物理索引计算方法定位

但是，实际测试下来只看到了一丢丢的提升，毕竟按照之前的写法，分支判断大部分情况下都不会进去，cpu的分支预测起作用了？或者是内存瓶颈这一点点计算优化影响不大。

最终写出c++20版本的spsc队列：
```c++
template<typename T>
class SPSC4
{
public:
    explicit SPSC4(size_t minCapacity)
        : m_capacity(std::bit_ceil(minCapacity <= 1 ? 2 : minCapacity))
        , m_mask(m_capacity - 1)
        , m_buffer(static_cast<T*>(std::malloc(m_capacity * sizeof(T))))
    {
        if (!m_buffer) {
            throw std::bad_alloc();
        }
    }

    ~SPSC4()
    {
        // We need to destruct anything that may still exist in our queue.
        // (No real synchronization needed at destructor time: only one
        // thread can be doing this.)
        if (!std::is_trivially_destructible<T>::value) {
            size_t readIndex = m_consumerData.readIndex;
            size_t endIndex = m_producerData.writeIndex;
            while (readIndex != endIndex) {
                m_buffer[readIndex & m_mask].~T();
                readIndex++;
            }
        }

        std::free(m_buffer);
    }

    SPSC4(const SPSC4&) = delete;
    SPSC4& operator=(const SPSC4&) = delete;
    SPSC4(SPSC4&&) = delete;
    SPSC4& operator=(SPSC4&&) = delete;

    template<class... Args>
    bool push(Args&&... args)
    {
        size_t currentWriteIndex = m_producerData.writeIndex.load(std::memory_order_relaxed);
        size_t availableSpace = getAvailableSpace(currentWriteIndex, m_producerData.readIndexCache);

        if (availableSpace < 1) {
            // if not enough space, refresh the cached read index and check again
            m_producerData.readIndexCache = m_consumerData.readIndex.load(std::memory_order_acquire);
            availableSpace = getAvailableSpace(currentWriteIndex, m_producerData.readIndexCache);
            if (availableSpace < 1) {
                return false; // not enough space
            }
        }

        size_t physicalWriteIndex = currentWriteIndex & m_mask;

        new (&m_buffer[physicalWriteIndex]) T(std::forward<Args>(args)...);

        size_t nextWriteIndex = currentWriteIndex + 1;
        m_producerData.writeIndex.store(nextWriteIndex, std::memory_order_release);
        return true;
    }

    bool pop(T& output)
    {
        size_t currentReadIndex = m_consumerData.readIndex.load(std::memory_order_relaxed);
        size_t availableSamples = calculateSize(m_consumerData.writeIndexCache, currentReadIndex);

        if (availableSamples < 1) {
            m_consumerData.writeIndexCache = m_producerData.writeIndex.load(std::memory_order_acquire);
            availableSamples = calculateSize(m_consumerData.writeIndexCache, currentReadIndex);
            if (availableSamples < 1) {
                return false;  // not enough samples
            }
        }

        size_t physicalReadIndex = currentReadIndex & m_mask;

        output = std::move(m_buffer[physicalReadIndex]);
        m_buffer[physicalReadIndex].~T();

        size_t nextReadIndex = currentReadIndex + 1;
        m_consumerData.readIndex.store(nextReadIndex, std::memory_order_release);
        return true;
    }

    size_t capacity() const { return m_capacity; }

    bool isEmpty() const
    {
        return m_producerData.writeIndex.load(std::memory_order_acquire) ==
               m_consumerData.readIndex.load(std::memory_order_acquire);
    }
    bool isFull() const
    {
        // use the warp-around logic to check if the buffer is full
        return (m_producerData.writeIndex.load(std::memory_order_acquire) -
                m_consumerData.readIndex.load(std::memory_order_acquire)) == m_capacity;
    }

private:
    size_t calculateSize(size_t writeIndex, size_t readIndex) const { return writeIndex - readIndex; }
    size_t getAvailableSpace(size_t writeIndex, size_t readIndex) const
    {
        return m_capacity - calculateSize(writeIndex, readIndex);
    }

    using AtomicIndex = std::atomic<size_t>;

    const size_t m_capacity;
    const size_t m_mask;
    T* const m_buffer;

    struct alignas(kCacheLineSize)
    {
        AtomicIndex readIndex = {0};
        size_t writeIndexCache = {0};
    } m_consumerData;

    struct alignas(kCacheLineSize)
    {
        AtomicIndex writeIndex = {0};
        size_t readIndexCache = {0};
    } m_producerData;
};

```

## 写在最后

可以看到目前的开源库基本都没有提供批量操作，[concurrentqueue](https://github.com/cameron314/concurrentqueue)倒是提供了bulk操作。

那怎么才能批量操作呢？
* 把上述的每次操作的**索引+1**改成**索引+n**，这样就可以批量操作了
* 对于push和pop，都要处理回绕问题，需要分成两块数据操作

除此之外，还可以使用Claim的方式
* Claim: 生产者问队列：“给我预留 N 个位置，把指针给我”。
* Write: 生产者直接往这些指针里写数据（Zero-Copy，直接构造）。
* Commit: 生产者告诉队列：“这 N 个写完了，更新索引”。

## 更新
发现[cppcon2023 spsc](https://www.youtube.com/watch?v=K3P_Lmq6pw0)的视频已经讲过了，对应的[github代码](https://github.com/CharlesFrasch/cppcon2023)。  
而且总结的条款顺序和我一样，不过作者最后放的测试结果，我的机器上没发现他写的Fifi4a比其他的开源库有明显的性能提升，毕竟这些spsc的代码都大差不差的。

我的测试代码见[miyanyan/spsc](https://github.com/miyanyan/spsc)