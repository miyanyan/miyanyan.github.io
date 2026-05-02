---
title:      "C++ sort，递归深度，快排pivot的选择"
date:       2026-05-02
categories: [c++]
tag: [c++, std]
---

# std::sort是introsort
对区间[first, last)
* 大区间：quicksort partition，主力，平均最快
* 递归过深：heapsort 兜底，防止 O(n^2)
* 小区间(gcc是16 msvc是32)：insertion sort，循环简单、访问连续、缓存友好...

## 快排的pivot选择

### MSVC

MSVC 使用的函数之一是：

```cpp
template <class _RanIt, class _Pr>
_CONSTEXPR20 void _Guess_median_unchecked(
    _RanIt _First,
    _RanIt _Mid,
    _RanIt _Last,
    _Pr _Pred
) {
    // sort median element to middle
    using _Diff        = _Iter_diff_t<_RanIt>;
    const _Diff _Count = _Last - _First;

    if (40 < _Count) { // Tukey's ninther
        const _Diff _Step     = (_Count + 1) >> 3;
        const _Diff _Two_step = _Step << 1;

        _STD _Med3_unchecked(_First, _First + _Step, _First + _Two_step, _Pred);
        _STD _Med3_unchecked(_Mid - _Step, _Mid, _Mid + _Step, _Pred);
        _STD _Med3_unchecked(_Last - _Two_step, _Last - _Step, _Last, _Pred);
        _STD _Med3_unchecked(_First + _Step, _Mid, _Last - _Step, _Pred);
    } else {
        _STD _Med3_unchecked(_First, _Mid, _Last, _Pred);
    }
}

template <class _RanIt, class _Pr>
_CONSTEXPR20 void _Med3_unchecked(_RanIt _First, _RanIt _Mid, _RanIt _Last, _Pr _Pred) {
    // sort median of three elements to middle
    if (_DEBUG_LT_PRED(_Pred, *_Mid, *_First)) {
        swap(*_Mid, *_First); // intentional ADL
    }

    if (_DEBUG_LT_PRED(_Pred, *_Last, *_Mid)) { // swap middle and last, then test first again
        swap(*_Last, *_Mid); // intentional ADL

        if (_DEBUG_LT_PRED(_Pred, *_Mid, *_First)) {
            swap(*_Mid, *_First); // intentional ADL
        }
    }
}
```

---

#### 1. 小区间：median-of-3

当区间长度不超过 40 时，MSVC 使用三数取中：

```cpp
_STD _Med3_unchecked(_First, _Mid, _Last, _Pred);
```

也就是从三个位置取样：

```text
_First
_Mid
_Last
```

然后把三者的中位数放到 `_Mid`。

例如：

```text
索引:   0   1   2   3   4
值:    10   4   8   2   6
```

采样三个值：

```text
_First = 10
_Mid   = 8
_Last  = 6
```

三者排序后是：

```text
6, 8, 10
```

中位数是：

```text
8
```

所以 `_Mid` 上最终放的是 `8`。

再看另一个例子：

```text
索引:   0   1    2   3   4
值:    10   4  100   2   6
```

采样：

```text
_First = 10
_Mid   = 100
_Last  = 6
```

三者排序后是：

```text
6, 10, 100
```

中位数是：

```text
10
```

所以 `_Med3_unchecked` 会通过交换，把 `10` 放到 `_Mid` 位置。

---

#### 2. 大区间：Tukey's ninther

当区间长度大于 40 时：

```cpp
if (40 < _Count)
```

MSVC 使用 **Tukey's ninther**。

它的思想是：

```text
取 9 个采样点
分成 3 组
每组取中位数
再从这 3 个中位数里取中位数
最终放到 _Mid
```

也就是：

```text
median(
    median(a1, a2, a3),
    median(b1, b2, b3),
    median(c1, c2, c3)
)
```

---

#### 3. `_Step` 是什么？

源码中：

```cpp
const _Diff _Step     = (_Count + 1) >> 3;
const _Diff _Two_step = _Step << 1;
```

等价于：

```cpp
_Step = (_Count + 1) / 8;
_Two_step = _Step * 2;
```

它用来在区间中均匀取样。

假设区间大概是这样：

```text
_First                                                   _Last
  |--------------------------------------------------------|
  0      1/8     2/8      3/8     4/8     5/8      6/8     7/8     8/8
```

MSVC 大概会取这 9 个位置：

```text
第一组：
_First
_First + _Step
_First + _Two_step

第二组：
_Mid - _Step
_Mid
_Mid + _Step

第三组：
_Last - _Two_step
_Last - _Step
_Last
```

可以理解为：

```text
左边取 3 个
中间取 3 个
右边取 3 个
```

---

### GCC

```cpp
template<typename _RandomAccessIterator, typename _Compare>
    _GLIBCXX20_CONSTEXPR
    inline _RandomAccessIterator
    __unguarded_partition_pivot(_RandomAccessIterator __first,
				_RandomAccessIterator __last, _Compare __comp)
    {
      typedef iterator_traits<_RandomAccessIterator> _IterTraits;
      typedef typename _IterTraits::difference_type _Dist;

      _RandomAccessIterator __mid = __first + _Dist((__last - __first) / 2);
      _RandomAccessIterator __second = __first + _Dist(1);
      std::__move_median_to_first(__first, __second, __mid, __last - _Dist(1),
				  __comp);
      return std::__unguarded_partition(__second, __last, __first, __comp);
    }
```

采用的是`first + 1` `mid` `last - 1`, 注意last-1是因为last是左闭右开的**开**, first+1则是因为想把first空出来放pivot?

## 递归深度判断

### MSVC

允许`1.5 log2(N)`的递归深度
```cpp
template <class _RanIt, class _Pr>
_CONSTEXPR20 void _Sort_unchecked(_RanIt _First, _RanIt _Last, _Iter_diff_t<_RanIt> _Ideal, _Pr _Pred) {
    // order [_First, _Last)
    for (;;) {
        if (_Last - _First <= _ISORT_MAX) { // small
            _STD _Insertion_sort_unchecked(_First, _Last, _Pred);
            return;
        }

        if (_Ideal <= 0) { // heap sort if too many divisions
            _STD _Make_heap_unchecked(_First, _Last, _Pred);
            _STD _Sort_heap_unchecked(_First, _Last, _Pred);
            return;
        }

        // divide and conquer by quicksort
        auto _Mid = _STD _Partition_by_median_guess_unchecked(_First, _Last, _Pred);

        _Ideal = (_Ideal >> 1) + (_Ideal >> 2); // allow 1.5 log2(N) divisions

        if (_Mid.first - _First < _Last - _Mid.second) { // loop on second half
            _STD _Sort_unchecked(_First, _Mid.first, _Ideal, _Pred);
            _First = _Mid.second;
        } else { // loop on first half
            _STD _Sort_unchecked(_Mid.second, _Last, _Ideal, _Pred);
            _Last = _Mid.first;
        }
    }
}
```

### GCC

允许`2 log2(N)`的递归深度

```cpp
template<typename _RandomAccessIterator, typename _Compare>
    _GLIBCXX20_CONSTEXPR
    inline void
    __sort(_RandomAccessIterator __first, _RandomAccessIterator __last,
	   _Compare __comp)
    {
      if (__first != __last)
	{
	  std::__introsort_loop(__first, __last,
				std::__lg(__last - __first) * 2,
				__comp);
	  std::__final_insertion_sort(__first, __last, __comp);
	}
    }
```