---
title:      "C++ 模板遇上动态库：为什么导出宏需要区分声明与定义"
description: C++ 模板遇上动态库：为什么导出宏需要区分声明与定义
date:       2026-10-21
categories: [c++]
tag: [c++]
---

# C++ 模板遇上动态库：为什么导出宏需要区分声明与定义

为普通函数设计跨平台导出宏并不复杂：Windows 构建动态库时使用 `__declspec(dllexport)`，使用动态库时使用 `__declspec(dllimport)`；Linux 则使用 `__attribute__((visibility("default")))`。

但把同一个宏放到类模板的显式实例化上，事情就没有这么简单了。某种写法可能在 MSVC 下触发警告，另一种写法又可能被 GCC 忽略，最后表现为调用方链接失败。

Chromium 和 LLVM libc++ 都为此区分了两类宏：一类用于**显式实例化声明**，另一类用于**显式实例化定义**。理解这种设计，需要先分清“生成模板代码”和“导出符号”这两件事。

---

假设一个库提供了下面的模板：

```cpp
template<class T>
class Box
{
public:
    int value();
};
```

如果方法定义在头文件中，调用方通常可以按需实例化它。但有些库希望把实现留在 `.cpp` 中，只提供若干预先编译好的实例，例如 `Box<int>`。

这时可以在头文件中写：

```cpp
extern template class Box<int>;
```

并在实现文件中，在相关方法定义之后写：

```cpp
template class Box<int>;
```

前者是显式实例化声明，用于抑制相应的隐式实例化；后者是显式实例化定义，负责生成实例。调用方使用的方法，必须最终有可链接的定义。[Microsoft：显式实例化](https://learn.microsoft.com/en-us/cpp/cpp/explicit-instantiation?view=msvc-170)

这里的 `extern` 并不表示导出。

**显式实例化解决的是代码在哪里生成；符号导出解决的是生成的代码能否被动态库外部使用。**

因此，当 `Box<int>` 由动态库提供时，还要考虑导出属性放在哪里。

---

在 Windows 上，一个很自然的尝试是：

```cpp
extern template class __declspec(dllexport) Box<int>;
```

但 MSVC 会对这种组合产生 C4910 警告。官方文档明确指出，`extern` 和 `dllexport` 在这里不兼容：前者抑制实例化，后者要求导出并实例化类成员。[Microsoft：C4910](https://learn.microsoft.com/en-us/cpp/error-messages/compiler-warnings/compiler-warning-level-1-c4910?view=msvc-170)

可以直接打开这个示例：

**[Godbolt：MSVC 的 `extern template + dllexport` 冲突](https://godbolt.org/z/ePhr79Gac)**

示例使用 MSVC 19.44，并添加：

```text
/std:c++20 /W1 /we4910
```

`/we4910` 将该警告提升为错误，方便观察。去掉这个选项后，重点仍然是诊断本身：不能因为编译器继续生成代码，就把这种组合视为合适的跨平台接口约定。

对于库的实现方，更合适的组合是：

```cpp
// 头文件：保留声明，但不附加 dllexport
extern template class Box<int>;

// 实现文件：在这里生成并导出
template class __declspec(dllexport) Box<int>;
```

调用方看到的则是：

```cpp
extern template class __declspec(dllimport) Box<int>;
```

也就是说，**Windows 构建 DLL 时不必删除 `extern template`，只需避免在它上面添加 `dllexport`。**

---

那么，能否把所有平台的导出属性都移到 `.cpp` 的实例化定义上？

在 GCC 下，这也不能一概而论。尤其是模板已经因其他声明而被实例化后，再添加类型可见性属性，可能已经太晚。

下面用一个嵌套实现类型说明这个问题：

```cpp
#define API __attribute__((visibility("default")))

template<class T>
class Box
{
public:
    int value();

private:
    struct Impl;
};

template<>
struct Box<int>::Impl
{
    int value = 42;
};

template<class T>
int Box<T>::value()
{
    return Impl{}.value;
}

// 在这个例子中，属性出现得太晚
template class API Box<int>;
```

GCC 13.3 会诊断：

```text
type attributes ignored after type is already defined
```

嵌套类型特化的定义使外层 `Box<int>` 的实例化先于最后那条声明发生。此时再试图通过类型属性改变可见性，GCC 会忽略它。如果库使用默认隐藏符号的构建方式，这就可能让本应公开的方法仍然处于隐藏状态。

**[Godbolt：比较 GCC 中“提前声明”和“声明过晚”的结果](https://godbolt.org/z/b65vWrxG3)**

这个链接提供两个 GCC 13.3 编译窗口，使用相同源码：

- `EARLY=0`：仅在最后的实例化定义上标注属性，产生诊断。
- `EARLY=1`：提前声明可见性，编译通过。

两个窗口均启用：

```text
-fvisibility=hidden
-fvisibility-inlines-hidden
-Werror=attributes
```

修复后的关键顺序是：

```cpp
// 在定义 Impl 之前，先声明实例的可见性
extern template class API Box<int>;

// 随后定义 Impl 和方法……

// 最后生成实例，无需重复添加属性
template class Box<int>;
```

这个例子说明的是**可见性属性的声明时机**。它不意味着 GCC 在所有代码结构中都禁止给显式实例化定义添加属性。

---

跨平台库需要表达的规则，其实可以整理成一张表：

| 场景 | 头文件中的实例化声明 | 实现文件中的实例化定义 |
|---|---|---|
| Windows 构建 DLL | 不带属性 | `dllexport` |
| Windows 使用 DLL | `dllimport` | 调用方不定义实例 |
| Linux 共享库 | 提前声明默认可见性 | 不重复添加属性 |

Chromium 将这组规则封装为：

```cpp
// 头文件
extern template class EXPORT_TEMPLATE_DECLARE(FOO_EXPORT) Foo<int>;

// 实现文件
template class EXPORT_TEMPLATE_DEFINE(FOO_EXPORT) Foo<int>;
```

它的 [`base/export_template.h`](https://github.com/chromium/chromium/blob/main/base/export_template.h) 不仅实现了这些宏，还在文件开头解释了不同平台的展开方式。它会识别传入的导出宏，将属性放到适当的位置。

LLVM libc++ 使用不同的名称，但职责划分一致：

- `_LIBCPP_EXTERN_TEMPLATE_TYPE_VIS`：用于类模板实例化声明。
- `_LIBCPP_CLASS_TEMPLATE_INSTANTIATION_VIS`：用于类模板实例化定义。

libc++ 文档明确说明：Windows 构建库时，声明宏为空，定义宏负责导出；其他平台则不需要在实例化定义上重复标注。[libc++：符号可见性宏](https://libcxx.llvm.org/DesignDocs/VisibilityMacros.html)

对于只维护少量动态库的项目，不一定需要复制 Chromium 的宏识别逻辑。直接定义一对宏，也能表达同样的规则：

```cpp
#if defined(_WIN32)

#  if defined(DEMO_BUILD)
#    define DEMO_API __declspec(dllexport)
#    define DEMO_TEMPLATE_DECLARE
#    define DEMO_TEMPLATE_DEFINE DEMO_API
#  else
#    define DEMO_API __declspec(dllimport)
#    define DEMO_TEMPLATE_DECLARE DEMO_API
#    define DEMO_TEMPLATE_DEFINE
#  endif

#else

#  define DEMO_API __attribute__((visibility("default")))
#  define DEMO_TEMPLATE_DECLARE DEMO_API
#  define DEMO_TEMPLATE_DEFINE

#endif
```

各个类只需要使用统一语法：

```cpp
// 头文件
extern template class DEMO_TEMPLATE_DECLARE Box<int>;

// 实现文件
template class DEMO_TEMPLATE_DEFINE Box<int>;
```

**[Godbolt：声明与定义分离后的跨平台实现](https://godbolt.org/z/Ydsjjhvrb)**

该示例已在 Godbolt 的 MSVC 19.44 和 GCC 13.3 上编译通过。它演示的是动态库构建侧；实际项目中，`DEMO_BUILD` 应由构建系统为库目标设置，调用方不定义它。示例未包含静态库配置。

这样设计以后，平台条件集中在导出头文件中。增加新的模板类时，只需添加声明和定义，不必再复制一组 `#if`。

---

阅读其他开源库时，还需要注意：它们实例化的是整个类，还是某个函数。

例如，fmt 11.2 的头文件中存在这样的写法：

```cpp
extern template FMT_API auto decimal_point_impl(locale_ref) -> char;
```

对应的实现文件也在显式实例化时使用 `FMT_API`。[fmt 头文件](https://github.com/fmtlib/fmt/blob/11.2.0/include/fmt/format.h#L1151)、[fmt 实现文件](https://github.com/fmtlib/fmt/blob/11.2.0/src/format.cc#L28)

这并不与前面的设计矛盾：**它是函数模板实例化，不是整类实例化。** libc++ 文档也明确区分了这两种情况，不能把类模板的限制直接套到函数模板上。

逐个实例化成员函数也是一种可选方案。它能更精确地控制哪些函数进入动态库，但需要维护每个成员的实例化签名。如果公开成员较多，整类实例化配合声明、定义宏，通常更容易保持一致。

Godbolt 很适合验证这些编译器诊断和声明顺序。不过，发布动态库时，还应增加一个独立的调用方程序：库和调用方分别编译，再进行链接和运行。Linux 可以通过 `nm -D -C` 检查动态符号，Windows 可以通过 `dumpbin /EXPORTS` 检查 DLL 导出表。

**库自身编译成功，并不等于库外调用方能够使用它。** 对模板动态库而言，独立消费者的链接测试，才能覆盖代码生成、符号可见性和导入声明是否真正配合正确。
