---
title:      "C++ 编译期获取类名"
description: 给[yalantinglibs]的pr记录
date:       2025-03-31
categories: [c++]
tag: [c++, std]
---

发现[yalantinglibs](https://github.com/alibaba/yalantinglibs/pull/874)的type_string的输出漏掉了const关键字，问了作者发现确实是一个bug，在提pr的过程也发现了这种实现还是有缺陷的，在此记录下

## 原理

* 获取到各个编译器下的原有名字，msvc下为`__FUNCSIG__`，gcc和clang下为`__PRETTY_FUNCTION__`
* 进行拆分截断

**【缺陷】并不能保证各个编译器下的字符串完全一致，包括但不限于多或少一个空格，逗号的位置等等（msvc还会加上class struct union关键字！）**

## 代码

msvc、gcc、clang下均可运行

``` c++
#include <string_view>
#include <iostream>
#include <array>
#include <vector>
#include <deque>
#include <queue>

template <typename T>
constexpr std::string_view get_raw_name() {
#ifdef _MSC_VER
  return __FUNCSIG__;
#else
  return __PRETTY_FUNCTION__;
#endif
}

template <auto T>
constexpr std::string_view get_raw_name() {
#ifdef _MSC_VER
  return __FUNCSIG__;
#else
  return __PRETTY_FUNCTION__;
#endif
}

template <typename T>
inline constexpr std::string_view type_string() {
  constexpr std::string_view sample = get_raw_name<int>();
  constexpr size_t prefix_length = sample.find("int");
  constexpr size_t suffix_length = sample.size() - prefix_length - 3;

  constexpr std::string_view str = get_raw_name<T>();
  return str.substr(prefix_length, str.size() - prefix_length - suffix_length);
}
```

[compiler explorer 在线运行链接](https://www.godbolt.org/z/dncsebqKv)
