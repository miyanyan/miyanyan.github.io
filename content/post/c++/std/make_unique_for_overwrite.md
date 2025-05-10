---
title: 使用std::make_unique_for_overwrite创建一个char数组当缓冲区
description: 偶然从别人博客评论区发现的，原来std还有这个！
date: 2025-02-24
categories:
    - c++
tags:
    - c++
    - std
---


## 起因

看到这个[博客](https://microcai.org/2024/11/16/do-not-new-a-buffer.html)里写道这样写进行了初始化操作

```c++
auto buffer_size = 5*1024*1024;
auto buffer = std::make_unique<char[]>(buffer_size);
```

评论区jajuju提出可以使用创建buffer
> 其实这里的情况是 make_unique 里用的 new 表达式是 `new T[n]()` ，对于 char 数组会进行零初始化。
如果换用 make_unique_for_overwrite，里面的 new 表达式就是 `new T[n]`，此时进行默认初始化，就不会给 char 数组内容清零。

瞬间感觉学到了一个新知识，这个`make_unique_for_overwrite`真是第一次见了(没看过cppreference上的make_unique一节...)

## 源码分析

我们以[cppreference](https://en.cppreference.com/w/cpp/memory/unique_ptr/make_unique)上的实现分析下源码：

1. 非数组类型

    ```c++
    template<class T>
    requires (!std::is_array_v<T>)
    std::unique_ptr<T> make_unique_for_overwrite() {
        return std::unique_ptr<T>(new T);
    }
    ```

* 效果类似于 new T，但没有 ()，因此不会执行默认初始化（特别是对于 POD 类型）。
* 构造函数仍然会被调用（如果 T 不是 POD）。

2. 动态数组

    ```c++
    template<class T>
    requires std::is_unbounded_array_v<T>
    std::unique_ptr<T> make_unique_for_overwrite(std::size_t n) {
        return std::unique_ptr<T>(new std::remove_extent_t<T>[n]);
    }
    ```

* 分配 n 个 T 元素的未初始化数组，不调用 ()，因此不会默认初始化（类似 new T[n]）。
* 不会调用构造函数（如果 T 是 POD）。

3. 禁止定长数组

    ```c++
    template<class T, class... Args>
    requires std::is_bounded_array_v<T>
    void make_unique_for_overwrite(Args&&...) = delete;
    ```

* 因为定长数组不能通过 new 进行动态分配，所以编译时禁止。

## 参考

[MSVC STL实现](https://github.com/microsoft/STL/blob/main/stl/inc/memory#L3618)
