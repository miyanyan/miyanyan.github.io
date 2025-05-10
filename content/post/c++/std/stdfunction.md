---
title:      "C++ std::function 实现原理"
date:       2024-12-22
categories: [c++]
tag: [c++, std]
---

## msvc

源码见[https://github.com/microsoft/STL/blob/main/stl/inc/functional](https://github.com/microsoft/STL/blob/main/stl/inc/functional)

### 预备知识

#### 参数类型，可以分为一元(unary)和二元(binary)，这个概念很重要，gcc的实现里也用到

可以看到msvc里定义了三个_Arg_types：无参数类型；接受一个参数，一元；接受两个参数，二元。
并且_Arg_types没有成员变量，只是定义了对应的类型。

``` c++
template <class... _Types>
struct _Arg_types {}; // provide argument_type, etc. when sizeof...(_Types) is 1 or 2

template <class _Ty1>
struct _Arg_types<_Ty1> {
    _CXX17_DEPRECATE_ADAPTOR_TYPEDEFS typedef _Ty1 _ARGUMENT_TYPE_NAME;
};

template <class _Ty1, class _Ty2>
struct _Arg_types<_Ty1, _Ty2> {
    _CXX17_DEPRECATE_ADAPTOR_TYPEDEFS typedef _Ty1 _FIRST_ARGUMENT_TYPE_NAME;
    _CXX17_DEPRECATE_ADAPTOR_TYPEDEFS typedef _Ty2 _SECOND_ARGUMENT_TYPE_NAME;
};
```

#### 一个function应该有什么接口

* 能调用。必须的！
* 能复制。
* 能移动。
* 能删除。

我们看一下msvc实现的接口类，正好符合这四个特点。
这里需要注意关键字```__declspec(novtable)```，它用于告诉编译器不要生成vtable，这样就可以减少额外的开销，毕竟function作为一个很基础的功能，当然是能省则省。

``` c++
template <class _Rx, class... _Types>
class __declspec(novtable) _Func_base { // abstract base for implementation types
public:
    virtual _Func_base* _Copy(void*) const                 = 0;
    virtual _Func_base* _Move(void*) noexcept              = 0;
    virtual _Rx _Do_call(_Types&&...)                      = 0;
    virtual const type_info& _Target_type() const noexcept = 0;
    virtual void _Delete_this(bool) noexcept               = 0;

#if _HAS_STATIC_RTTI
    const void* _Target(const type_info& _Info) const noexcept {
        return _Target_type() == _Info ? _Get() : nullptr;
    }
#endif // _HAS_STATIC_RTTI

    _Func_base()                  = default;
    _Func_base(const _Func_base&) = delete;
    _Func_base& operator=(const _Func_base&) = delete;
    // dtor non-virtual due to _Delete_this()

private:
    virtual const void* _Get() const noexcept = 0;
};
```

### 走进源码

#### 内存对象

首先进入function class，它继承自_Get_function_impl::type，public函数里都是构造函数相关，我们着重观察这个_Get_function_impl实现。**没有成员变量**

``` c++
template <class _Fty>
class function : public _Get_function_impl<_Fty>::type { // wrapper for callable objects
private:
    using _Mybase = typename _Get_function_impl<_Fty>::type;

public:
    // 构造函数
};
```

可以看到_Get_function_impl的主要部分为_Get_function_impl::type，它是_Func_class<_Ret,_Types...>。**没有成员变量**

``` c++
template <class _Tx>
struct _Get_function_impl {
    static_assert(_Always_false<_Tx>, "std::function only accepts function types as template arguments.");
};

#define _GET_FUNCTION_IMPL(CALL_OPT, X1, X2, X3)                                                  \
    template <class _Ret, class... _Types>                                                        \
    struct _Get_function_impl<_Ret CALL_OPT(_Types...)> { /* determine type from argument list */ \
        using type = _Func_class<_Ret, _Types...>;                                                \
    };

_NON_MEMBER_CALL(_GET_FUNCTION_IMPL, X1, X2, X3)
#undef _GET_FUNCTION_IMPL
```

_Func_class 模板参数为 返回值 + 可变参，继承自_Arg_types<_Types...>，根据参数个数拿到了对应的类型。

* private里有**一个成员变量**_Storage_Mystorage，它是一个联合体，在64位平台上，```_Small_object_num_ptrs = 6 + 16 / sizeof(void*) = 8```，```_Space_size = (_Small_object_num_ptrs - 1) * sizeof(void*) = 56```，
  所以这个_Mystorage的实际大小为64个byte。

  ``` c++
    union _Storage { // storage for small objects (basic_string is small)
        max_align_t _Dummy1; // for maximum alignment
        char _Dummy2[_Space_size]; // to permit aliasing
        _Ptrt* _Ptrs[_Small_object_num_ptrs]; // _Ptrs[_Small_object_num_ptrs - 1] is reserved
    };
  ```

* public里为构造函数 + 析构函数 + 重载operator()，符合我们的常规认知，function就是一个根据传参执行的函数。
  这里我们主要看三个函数，_Set、_Getimpl、_Tidy，它们都是在对_Mystorage的最后一个_Ptr进行操作
  * _Set，赋值给_Mystorage的最后一个_Ptr

    ``` c++
    void _Set(_Ptrt* _Ptr) noexcept { // store pointer to object
        _Mystorage._Ptrs[_Small_object_num_ptrs - 1] = _Ptr;
    }
    ```

  * _Getimpl，返回_Mystorage的最后一个_Ptr

    ``` c++
    _Ptrt* _Getimpl() const noexcept { // get pointer to object
        return _Mystorage._Ptrs[_Small_object_num_ptrs - 1];
    }
    ```

  * _Tidy，首先删除对应指针，然后_Mystorage的最后一个_Ptr置空。

    ``` c++
    void _Tidy() noexcept {
        if (!_Empty()) { // destroy callable object and maybe delete it
            _Getimpl()->_Delete_this(!_Local());
            _Set(nullptr);
        }
    }
    ```

* protected里主要为内存分配相关函数，我们重点关注内存分配函数_Reset，可以看到根据_Is_large主要分为两种分配方式，

  ``` c++
  // 当函数对象的大小超过56字节，或者对齐字节数大于max_align_t，或者是否可以不抛异常移动构造
  template <class _Impl> // determine whether _Impl must be dynamically allocated
  _INLINE_VAR constexpr bool _Is_large = sizeof(_Impl) > _Space_size || alignof(_Impl) > alignof(max_align_t)
                                    || !_Impl::_Nothrow_move::value;
  ```

  * 小内存函数对象，调用 placement new，直接构造在_Mystorage上
  * 大内存函数对象，需要动态分配，内存分配到堆上

    ``` c++
    template <class _Fx>
    void _Reset(_Fx&& _Val) { // store copy of _Val
        if (!_Test_callable(_Val)) { // null member pointer/function pointer/std::function
            return; // already empty
        }

        using _Impl = _Func_impl_no_alloc<decay_t<_Fx>, _Ret, _Types...>;
        if constexpr (_Is_large<_Impl>) {
            // dynamically allocate _Val
            _Set(_Global_new<_Impl>(_STD forward<_Fx>(_Val)));
        } else {
            // store _Val in-situ
            _Set(::new (static_cast<void*>(&_Mystorage)) _Impl(_STD forward<_Fx>(_Val)));
        }
    }
    ```

#### MSVC主要源码

``` c++
template <class _Ret, class... _Types>
class _Func_class : public _Arg_types<_Types...> {
public:
    using result_type = _Ret;

    using _Ptrt = _Func_base<_Ret, _Types...>;

    _Func_class() noexcept {
        _Set(nullptr);
    }

    _Ret operator()(_Types... _Args) const {
        if (_Empty()) {
            _Xbad_function_call();
        }
        const auto _Impl = _Getimpl();
        return _Impl->_Do_call(_STD forward<_Types>(_Args)...);
    }

    ~_Func_class() noexcept {
        _Tidy();
    }

protected:
    template <class _Fx, class _Function>
    using _Enable_if_callable_t = enable_if_t<conjunction_v<negation<is_same<_Remove_cvref_t<_Fx>, _Function>>,
                                                  _Is_invocable_r<_Ret, decay_t<_Fx>&, _Types...>>,
        int>;

    bool _Empty() const noexcept {
        return !_Getimpl();
    }

    void _Reset_copy(const _Func_class& _Right) { // copy _Right's stored object
        if (!_Right._Empty()) {
            _Set(_Right._Getimpl()->_Copy(&_Mystorage));
        }
    }

    void _Reset_move(_Func_class&& _Right) noexcept { // move _Right's stored object
        if (!_Right._Empty()) {
            if (_Right._Local()) { // move and tidy
                _Set(_Right._Getimpl()->_Move(&_Mystorage));
                _Right._Tidy();
            } else { // steal from _Right
                _Set(_Right._Getimpl());
                _Right._Set(nullptr);
            }
        }
    }

    template <class _Fx>
    void _Reset(_Fx&& _Val) { // store copy of _Val
        if (!_Test_callable(_Val)) { // null member pointer/function pointer/std::function
            return; // already empty
        }

        using _Impl = _Func_impl_no_alloc<decay_t<_Fx>, _Ret, _Types...>;
        if constexpr (_Is_large<_Impl>) {
            // dynamically allocate _Val
            _Set(_Global_new<_Impl>(_STD forward<_Fx>(_Val)));
        } else {
            // store _Val in-situ
            _Set(::new (static_cast<void*>(&_Mystorage)) _Impl(_STD forward<_Fx>(_Val)));
        }
    }

    void _Tidy() noexcept {
        if (!_Empty()) { // destroy callable object and maybe delete it
            _Getimpl()->_Delete_this(!_Local());
            _Set(nullptr);
        }
    }

    void _Swap(_Func_class& _Right) noexcept { // swap contents with contents of _Right
        if (!_Local() && !_Right._Local()) { // just swap pointers
            _Ptrt* _Temp = _Getimpl();
            _Set(_Right._Getimpl());
            _Right._Set(_Temp);
        } else { // do three-way move
            _Func_class _Temp;
            _Temp._Reset_move(_STD move(*this));
            _Reset_move(_STD move(_Right));
            _Right._Reset_move(_STD move(_Temp));
        }
    }

private:
    bool _Local() const noexcept { // test for locally stored copy of object
        return _Getimpl() == static_cast<const void*>(&_Mystorage);
    }

    union _Storage { // storage for small objects (basic_string is small)
        max_align_t _Dummy1; // for maximum alignment
        char _Dummy2[_Space_size]; // to permit aliasing
        _Ptrt* _Ptrs[_Small_object_num_ptrs]; // _Ptrs[_Small_object_num_ptrs - 1] is reserved
    };

    _Storage _Mystorage;
    enum { _EEN_IMPL = _Small_object_num_ptrs - 1 }; // helper for expression evaluator
    _Ptrt* _Getimpl() const noexcept { // get pointer to object
        return _Mystorage._Ptrs[_Small_object_num_ptrs - 1];
    }

    void _Set(_Ptrt* _Ptr) noexcept { // store pointer to object
        _Mystorage._Ptrs[_Small_object_num_ptrs - 1] = _Ptr;
    }
};
```

#### 如何调用

通过上一节我们可以了解到std::function核心的内存就是_Storage_Mystorage，可以通过_Mystorage的最后一个_Ptr得到函数调用的指针，那有了函数指针，怎么调用呢？
在_Func_class::_Reset中我们首先分配了对象_Func_impl_no_alloc，然后把其指针赋值给_Mystorage的最后一个_Ptr，这个_Func_impl_no_alloc就是函数调用的具体实现，
它继承自_Func_base，主要实现了调用、移动、复制、删除接口：

``` c++
template <class _Callable, class _Rx, class... _Types>
class _Func_impl_no_alloc final : public _Func_base<_Rx, _Types...> {
    // derived class for specific implementation types that don't use allocators
public:
    using _Mybase       = _Func_base<_Rx, _Types...>;
    using _Nothrow_move = is_nothrow_move_constructible<_Callable>;

    template <class _Other, enable_if_t<!is_same_v<_Func_impl_no_alloc, decay_t<_Other>>, int> = 0>
    explicit _Func_impl_no_alloc(_Other&& _Val) : _Callee(_STD forward<_Other>(_Val)) {}

    // dtor non-virtual due to _Delete_this()

private:
    _Mybase* _Copy(void* _Where) const override {
        if constexpr (_Is_large<_Func_impl_no_alloc>) {
            return _Global_new<_Func_impl_no_alloc>(_Callee);
        } else {
            return ::new (_Where) _Func_impl_no_alloc(_Callee);
        }
    }

    _Mybase* _Move(void* _Where) noexcept override {
        if constexpr (_Is_large<_Func_impl_no_alloc>) {
            return nullptr;
        } else {
            return ::new (_Where) _Func_impl_no_alloc(_STD move(_Callee));
        }
    }

    _Rx _Do_call(_Types&&... _Args) override { // call wrapped function
        return _Invoker_ret<_Rx>::_Call(_Callee, _STD forward<_Types>(_Args)...);
    }

    const void* _Get() const noexcept override {
        return _STD addressof(_Callee);
    }

    void _Delete_this(bool _Dealloc) noexcept override { // destroy self
        this->~_Func_impl_no_alloc();
        if (_Dealloc) {
            _Deallocate<alignof(_Func_impl_no_alloc)>(this, sizeof(_Func_impl_no_alloc));
        }
    }

    _Callable _Callee;
};
```

我们主要看一下_Do_call函数，它内部调用了_Invoker_ret<_Rx>::_Call，最终调用std::invoke完成函数调用

``` c++
// helper to give INVOKE an explicit return type; avoids undesirable Expression SFINAE
template <class _Rx, bool = is_void_v<_Rx>>
struct _Invoker_ret { // selected for _Rx being cv void
    template <class _Fx, class... _Valtys>
    static _CONSTEXPR20 void _Call(_Fx&& _Func, _Valtys&&... _Vals) noexcept(_Select_invoke_traits<_Fx,
        _Valtys...>::_Is_nothrow_invocable::value) { // INVOKE, "implicitly" converted to void
        _STD invoke(static_cast<_Fx&&>(_Func), static_cast<_Valtys&&>(_Vals)...);
    }
};

template <class _Rx>
struct _Invoker_ret<_Rx, false> { // selected for all _Rx other than cv void and _Unforced
    template <class _Fx, class... _Valtys>
    static _CONSTEXPR20 _Rx _Call(_Fx&& _Func, _Valtys&&... _Vals) noexcept(_Select_invoke_traits<_Fx,
        _Valtys...>::template _Is_nothrow_invocable_r<_Rx>::value) { // INVOKE, implicitly converted to _Rx
        return _STD invoke(static_cast<_Fx&&>(_Func), static_cast<_Valtys&&>(_Vals)...);
    }
};

template <>
struct _Invoker_ret<_Unforced, false> { // selected for _Rx being _Unforced
    template <class _Fx, class... _Valtys>
    static _CONSTEXPR20 auto _Call(_Fx&& _Func, _Valtys&&... _Vals) noexcept(
        _Select_invoke_traits<_Fx, _Valtys...>::_Is_nothrow_invocable::value)
        -> decltype(_STD invoke(static_cast<_Fx&&>(_Func), static_cast<_Valtys&&>(_Vals)...)) { // INVOKE, unchanged
        return _STD invoke(static_cast<_Fx&&>(_Func), static_cast<_Valtys&&>(_Vals)...);
    }
};
```

## gcc
