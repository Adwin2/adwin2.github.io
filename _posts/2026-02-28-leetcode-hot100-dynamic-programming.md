---
layout: post
title: LeetCode Hot 100 题解（六）：动态规划
categories: [算法]
description: LeetCode Hot 100 系列最后一篇，覆盖一维和多维动态规划的经典题目。
keywords: LeetCode, 算法, Golang, 动态规划, DP
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

Hot 100 系列最后一篇，动态规划。DP 题的关键是定义好状态和转移方程，想清楚了代码很短。

---

## 一维动态规划

### 70. 爬楼梯

`dp[i] = dp[i-1] + dp[i-2]`，空间优化到两个变量。

```go
func climbStairs(n int) int {
    if n <= 2 {
        return n
    }
    a, b := 1, 2
    for i := 3; i <= n; i++ {
        a, b = b, a+b
    }
    return b
}
// 时间 O(n)，空间 O(1)
```

### 118. 杨辉三角

每行的元素等于上一行相邻两个元素之和。

```go
func generate(numRows int) [][]int {
    res := make([][]int, numRows)
    for i := 0; i < numRows; i++ {
        res[i] = make([]int, i+1)
        res[i][0], res[i][i] = 1, 1
        for j := 1; j < i; j++ {
            res[i][j] = res[i-1][j-1] + res[i-1][j]
        }
    }
    return res
}
// 时间 O(n²)，空间 O(1)（不算输出）
```

### 198. 打家劫舍

偷或不偷当前房子：`dp[i] = max(dp[i-1], dp[i-2] + nums[i])`。

```go
func rob(nums []int) int {
    if len(nums) == 1 {
        return nums[0]
    }
    a, b := nums[0], max(nums[0], nums[1])
    for i := 2; i < len(nums); i++ {
        a, b = b, max(b, a+nums[i])
    }
    return b
}
// 时间 O(n)，空间 O(1)
```

### 279. 完全平方数

`dp[i]` = 组成 i 的最少完全平方数个数。

```go
func numSquares(n int) int {
    dp := make([]int, n+1)
    for i := 1; i <= n; i++ {
        dp[i] = i // 最坏情况全是 1
        for j := 1; j*j <= i; j++ {
            if dp[i-j*j]+1 < dp[i] {
                dp[i] = dp[i-j*j] + 1
            }
        }
    }
    return dp[n]
}
// 时间 O(n*sqrt(n))，空间 O(n)
```

### 322. 零钱兑换

完全背包问题。`dp[i]` = 凑成金额 i 的最少硬币数。

```go
func coinChange(coins []int, amount int) int {
    dp := make([]int, amount+1)
    for i := range dp {
        dp[i] = amount + 1
    }
    dp[0] = 0
    for i := 1; i <= amount; i++ {
        for _, c := range coins {
            if c <= i && dp[i-c]+1 < dp[i] {
                dp[i] = dp[i-c] + 1
            }
        }
    }
    if dp[amount] > amount {
        return -1
    }
    return dp[amount]
}
// 时间 O(amount * len(coins))，空间 O(amount)
```

### 139. 单词拆分

`dp[i]` 表示 s[:i] 能否由字典中的单词拼出。

```go
func wordBreak(s string, wordDict []string) bool {
    wordSet := make(map[string]bool)
    for _, w := range wordDict {
        wordSet[w] = true
    }
    dp := make([]bool, len(s)+1)
    dp[0] = true
    for i := 1; i <= len(s); i++ {
        for j := 0; j < i; j++ {
            if dp[j] && wordSet[s[j:i]] {
                dp[i] = true
                break
            }
        }
    }
    return dp[len(s)]
}
// 时间 O(n²)，空间 O(n)
```

### 300. 最长递增子序列

经典 DP + 贪心二分优化。`tails[i]` 存长度为 i+1 的递增子序列的最小末尾。

```go
func lengthOfLIS(nums []int) int {
    var tails []int
    for _, v := range nums {
        pos := sort.SearchInts(tails, v)
        if pos == len(tails) {
            tails = append(tails, v)
        } else {
            tails[pos] = v
        }
    }
    return len(tails)
}
// 时间 O(n*log(n))，空间 O(n)
```

### 152. 乘积最大子数组

同时维护以当前元素结尾的最大积和最小积（因为负数乘负数会变大）。

```go
func maxProduct(nums []int) int {
    ans, curMax, curMin := nums[0], nums[0], nums[0]
    for i := 1; i < len(nums); i++ {
        if nums[i] < 0 {
            curMax, curMin = curMin, curMax
        }
        curMax = max(nums[i], curMax*nums[i])
        curMin = min(nums[i], curMin*nums[i])
        if curMax > ans {
            ans = curMax
        }
    }
    return ans
}
// 时间 O(n)，空间 O(1)
```

### 416. 分割等和子集

转化为 0-1 背包：能否从数组中选出若干元素使其和等于 sum/2。

```go
func canPartition(nums []int) bool {
    sum := 0
    for _, v := range nums {
        sum += v
    }
    if sum%2 != 0 {
        return false
    }
    target := sum / 2
    dp := make([]bool, target+1)
    dp[0] = true
    for _, v := range nums {
        for j := target; j >= v; j-- {
            dp[j] = dp[j] || dp[j-v]
        }
    }
    return dp[target]
}
// 时间 O(n*target)，空间 O(target)
```

### 32. 最长有效括号

`dp[i]` 表示以 s[i] 结尾的最长有效括号长度。

```go
func longestValidParentheses(s string) int {
    n := len(s)
    dp := make([]int, n)
    ans := 0
    for i := 1; i < n; i++ {
        if s[i] == ')' {
            if s[i-1] == '(' {
                dp[i] = 2
                if i >= 2 {
                    dp[i] += dp[i-2]
                }
            } else if dp[i-1] > 0 {
                j := i - dp[i-1] - 1
                if j >= 0 && s[j] == '(' {
                    dp[i] = dp[i-1] + 2
                    if j >= 1 {
                        dp[i] += dp[j-1]
                    }
                }
            }
            if dp[i] > ans {
                ans = dp[i]
            }
        }
    }
    return ans
}
// 时间 O(n)，空间 O(n)
```

---

## 多维动态规划

### 62. 不同路径

`dp[i][j] = dp[i-1][j] + dp[i][j-1]`，空间优化到一维。

```go
func uniquePaths(m int, n int) int {
    dp := make([]int, n)
    for j := range dp {
        dp[j] = 1
    }
    for i := 1; i < m; i++ {
        for j := 1; j < n; j++ {
            dp[j] += dp[j-1]
        }
    }
    return dp[n-1]
}
// 时间 O(m*n)，空间 O(n)
```

### 64. 最小路径和

```go
func minPathSum(grid [][]int) int {
    m, n := len(grid), len(grid[0])
    dp := make([]int, n)
    dp[0] = grid[0][0]
    for j := 1; j < n; j++ {
        dp[j] = dp[j-1] + grid[0][j]
    }
    for i := 1; i < m; i++ {
        dp[0] += grid[i][0]
        for j := 1; j < n; j++ {
            dp[j] = min(dp[j], dp[j-1]) + grid[i][j]
        }
    }
    return dp[n-1]
}
// 时间 O(m*n)，空间 O(n)
```

### 5. 最长回文子串

中心扩展法，遍历每个可能的中心（奇数和偶数两种）。

```go
func longestPalindrome(s string) string {
    start, maxLen := 0, 0
    expand := func(l, r int) {
        for l >= 0 && r < len(s) && s[l] == s[r] {
            if r-l+1 > maxLen {
                start = l
                maxLen = r - l + 1
            }
            l--
            r++
        }
    }
    for i := range s {
        expand(i, i)   // 奇数长度
        expand(i, i+1) // 偶数长度
    }
    return s[start : start+maxLen]
}
// 时间 O(n²)，空间 O(1)
```

### 1143. 最长公共子序列

经典二维 DP：匹配则 `dp[i][j] = dp[i-1][j-1] + 1`，否则取左和上的最大值。

```go
func longestCommonSubsequence(text1 string, text2 string) int {
    m, n := len(text1), len(text2)
    dp := make([][]int, m+1)
    for i := range dp {
        dp[i] = make([]int, n+1)
    }
    for i := 1; i <= m; i++ {
        for j := 1; j <= n; j++ {
            if text1[i-1] == text2[j-1] {
                dp[i][j] = dp[i-1][j-1] + 1
            } else {
                dp[i][j] = max(dp[i-1][j], dp[i][j-1])
            }
        }
    }
    return dp[m][n]
}
// 时间 O(m*n)，空间 O(m*n)
```

### 72. 编辑距离

三种操作（插入、删除、替换）对应三个子问题。

```go
func minDistance(word1 string, word2 string) int {
    m, n := len(word1), len(word2)
    dp := make([][]int, m+1)
    for i := range dp {
        dp[i] = make([]int, n+1)
        dp[i][0] = i
    }
    for j := 0; j <= n; j++ {
        dp[0][j] = j
    }
    for i := 1; i <= m; i++ {
        for j := 1; j <= n; j++ {
            if word1[i-1] == word2[j-1] {
                dp[i][j] = dp[i-1][j-1]
            } else {
                dp[i][j] = 1 + min(dp[i-1][j-1], min(dp[i-1][j], dp[i][j-1]))
            }
        }
    }
    return dp[m][n]
}
// 时间 O(m*n)，空间 O(m*n)
```
