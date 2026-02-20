---
title:      "C++ permutation 排列相关"
date:       2026-02-20
categories: [c++]
tag: [c++, std]
---

## 实现和原理

### next_permutation

> 生成当前序列的**下一个字典序排列（lexicographically next permutation）**。  
> 若不存在更大的排列（当前已是最大字典序），则将序列重排为最小字典序（通常升序排列），并返回 `false`。

#### 1. 算法步骤

以序列[2,4,3,5,1]为例，他的下一个排列是[2,4,5,1,3]

初步观察，这好像是类似于进位的一个计算，比如`13 + 9`，最后一位大于可以表示的最大值9，那么就需要进位了，同时最后一位又变成的较小的数`2`。

同理，[5,1]已经是能表示的最大的了，没法更大，只能进位，于是我们要修改前面的那位`3`，那`3`“进位”成几？能进位成`4`吗？显然不能，因为`4`在前面占着呢。那只能从后面的[5,1]里选择，选谁？显然要选比`3`大的，这里要注意了，如果比`3`大的有很多个，那要选最小的那个，毕竟是下一个排列，要尽可能地“进位”小。那“进位”之后呢？后面的数就剩下了[3,1]，为了保持后面的最小，那就需要升序排序了。

等等，真的需要升序排序吗？

我们为什么选`3`而不是选`5`？因为[5,1]已经是最大的了，只靠这几个数不能出来更大的排列，这也就意味着后面原本需要升序排序的序列，是一个非递增的序列，只需要reverse就ok了！

我们也可以采用反正法：
令 $a_i$ 是这个序列的修改点，此时 $a_i < a_{i+1}$，序列 $a_i,...,a_{n-1}$必定降序，否则，其中必存在 $a_j \ge a_{j+1}$，那此时j变成了修改点，与i为修改点矛盾。

总结实现步骤：
* 从右往左找出第一个i使得$a_i < a_{i+1}$
* 从右往左找出j使得$a_j$是第一个不小于$a_i$的，交换$a_i$ 和 $a_j$
* reverse $a_{i+1},...,a_{n-1}$
---

#### 2. 代码实现

```cpp
namespace miyan {
template<class BidirIt, class Compare>
bool next_permutation(BidirIt first, BidirIt last, Compare comp)
{
    if (first == last) return false;
    BidirIt i = last;
    if (first == --i) return false;

    while (true) {
        auto i1 = i;
        if (comp(*--i, *i1)) {
            auto j = last;
            // 找到第一个不小于 $a_i$ 的
            while (!comp(*i, *--j)) {
            }
            swap(*i, *j); // intentional ADL
            std::reverse(i1, last);
            return true;
        }
        if (i == first) {
            std::reverse(first, last);
            return false;
        }
    }
}

template<class BidirIt>
bool next_permutation(BidirIt first, BidirIt last)
{
    return miyan::next_permutation(first, last, less<>{});
}
}
```


#### 3. 复杂度

最多线性扫描 + 一次反转

时间复杂度：O(n)

额外空间：O(1)

### prev_permutation

> 生成当前序列的上一个字典序排列（lexicographically previous permutation）。   
> 若不存在更小的排列（当前已是最小字典序），则将序列重排为最大字典序，并返回 false。

#### 1. 算法步骤

以序列[2,4,3,5,1]为例，他的上一个排列是[2,4,3,1,5]

是下一个排列的反向操作

总结实现步骤：
* 从右往左找出第一个i使得$a_i > a_{i+1}$
* 从右往左找出j使得$a_j$是第一个小于$a_i$的，交换$a_i$ 和 $a_j$
* reverse $a_{i+1},...,a_{n-1}$
---

#### 2. 代码实现

```cpp
namespace miyan {
template<class BidirIt, class Compare>
bool prev_permutation(BidirIt first, BidirIt last, Compare comp)
{
    if (first == last) return false;
    BidirIt i = last;
    if (first == --i) return false;

    while (true) {
        auto i1 = i;
        if (comp(*i1, *--i)) {
            auto j = last;
            // 找到第一个小于 $a_i$ 的
            while (!comp(*--j, *i)) {
            }
            swap(*i, *j); // intentional ADL
            std::reverse(i1, last);
            return true;
        }
        if (i == first) {
            std::reverse(first, last);
            return false;
        }
    }
}

template<class BidirIt>
bool prev_permutation(BidirIt first, BidirIt last)
{
    return prev_permutation(first, last, less<>{});
}
}
```

#### 3. 复杂度

最多线性扫描 + 一次反转

时间复杂度：O(n)

额外空间：O(1)


## 有什么用？

似乎只能用在暴力遍历所有排列的时候，比如MSVC STL的维护者就曾说用来进行STL测试，但是我工作中完全用不到next_permutation/prev_permutation...

https://www.reddit.com/r/cpp/comments/8b3gml/comment/dx4092f/?utm_source=share&utm_medium=web3x&utm_name=web3xcss&utm_term=1&utm_content=share_button