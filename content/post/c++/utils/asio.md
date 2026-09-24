---
title:      "ASIO资料记录"
description: ASIO相关知识点记录
date:       2025-12-08
categories: [c++]
tag: [c++, asio]
---

* `asio::any_io_executor` 带来的性能损失，在我的workflow里多线程消费小数据量场景下，any_io_executor 每次 post 分配类型擦除的 op state 的耗时占比明显增加，改为模板后性能提升40%左右，当然单位是 ns/帧，这个量级很小，不过改成模板是一个很简单的动作，影响范围不大，值得纳入
  ```c++
    using DispatchTask = std::function<void()>;
    using DispatchFn = std::function<void(DispatchTask)>;

    // BEFORE：executor 在形参处被类型擦除成 any_io_executor
    DispatchFn dispatchAsioErased(asio::any_io_executor executor)
    {
        return [ex = std::move(executor)](DispatchTask task) mutable {
            if (task)
            {
                asio::post(ex, std::move(task));
            }
        };
    }

    // AFTER：executor 保持具体类型
    template <typename Executor>
    DispatchFn dispatchAsioConcrete(Executor executor)
    {
        return [ex = std::move(executor)](DispatchTask task) mutable {
            if (task)
            {
                asio::post(ex, std::move(task));
            }
        };
    }
    ```