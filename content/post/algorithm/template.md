---
title:      "常用算法模板"
date:       2024-11-10
categories: [algorithm]
tag: [algorithm, leetcode]
---
## DP

### 数位DP

#### 是什么

数位是指把一个数字按照个、十、百、千等等一位一位地拆开，关注它每一位上的数字。如果拆的是十进制数，那么每一位数字都是 0~9，其他进制可类比十进制

#### 为什么

为什么要一位位的判断？因为对一个区间内的数的统计，往往有相似的地方。比如 [1000, 1999] 和 [2000, 2999] 只是第一位发生了变化，后三位变化类似，那么只需要后三位的变化能通用化，那么不就可以进行DP了。

数位DP一般有这些特征：

* 计数。
* 判断的条件可以转化为用每一位去考虑。
* 有数字的区间。常见的有 **小于x的** **大于x的小于y的** 等等
* 这个区间很大。否则可以直接暴力了。

#### 怎么做

首先因为区间很大，所以一般给定的是一个字符串形式的数字，我们假定为`s`，这个数字的总位数是`n`

[参考](https://leetcode.cn/problems/count-special-integers/solution/shu-wei-dp-mo-ban-by-endlesscheng-xtgx/)

dp[i][j]表示当前在第i位，前面维护了一个为j的值，且后面的位数可以随便选时的数字个数

定义函数```f(i, mask, isLimit, isNum)```表示构造从左往右第i位及其之后数位合法的方案数，其余参数的含义为：

* mask表示前面选过的数字集合，换句话说，第i位数字不能在mask中
* isLimit表示当前是否受到了约束，若为真，则第i位填入的数字至多为s[i]，否则可以是9。如果在受到约束的情况下填入9，则后面还会被约束
* isNum表示前i位是否填了数字，若为真，则填入的数字可以是从0开始，否则不能有前导0，即**可以跳过当前数字**或者从1开始

注意mask是可以变通的，比如

* [2719. 统计整数数目](https://leetcode.cn/problems/count-of-integers/description/), mask 是当前的数字和

则递归入口为:```f(0, 0, true, false)```

```c++
string nstr; // 给定数字的字符串形式
int n = nstr.size(); // 位数
int m = 1 << 10; // 0 - 9 的mask
vector<vector<int>> dp(n, vector<int>(m, -1));
function<int(int, int, bool, bool)> dfs = [&](int index, int mask, bool isLimit, bool isNum) {
    // 到头
    if (index == n) return ...;
    // 已缓存
    if (!isLimit && dp[index][mask] != -1) {
        return dp[index][mask];
    }
    int ans = 0;
    // 是否可以跳过
    if (!isNum) {
        ans += dfs(index + 1, mask, false, false);
    }
    // 确定枚举数字范围，最多[0, 9]
    int left = 1 - isNum;
    int right = isLimit ? nstr[index] - '0' : 9;
    for (int i = left; i <= right; ++i) {
        // 不在mask中
        if (mask >> i & 1) continue;
        ans += dfs(index + 1, mask | (1 << i), isLimit && i == right, true);
    }
    // 缓存
    if (!isLimit) {
        dp[index][mask] = ans;
    }
    return ans;
}; 
```
