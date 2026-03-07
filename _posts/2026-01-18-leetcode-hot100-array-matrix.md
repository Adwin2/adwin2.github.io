---
layout: post
title: LeetCode Hot 100 题解（二）：数组与矩阵
categories: [算法]
description: LeetCode Hot 100 系列第二篇，覆盖数组操作和矩阵相关题目。
keywords: LeetCode, 算法, Golang, 数组, 矩阵
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

Hot 100 系列第二篇，数组和矩阵。这类题目往往思路不难但细节容易出错。

---

## 数组

### 53. 最大子数组和

Kadane 算法。当前子数组和为负数时重新开始。

```go
func maxSubArray(nums []int) int {
    ans, cur := nums[0], nums[0]
    for i := 1; i < len(nums); i++ {
        if cur < 0 {
            cur = nums[i]
        } else {
            cur += nums[i]
        }
        if cur > ans {
            ans = cur
        }
    }
    return ans
}
// 时间 O(n)，空间 O(1)
```

### 56. 合并区间

按左端点排序，逐个合并有重叠的区间。

```go
func merge(intervals [][]int) [][]int {
    sort.Slice(intervals, func(i, j int) bool {
        return intervals[i][0] < intervals[j][0]
    })
    var res [][]int
    for _, v := range intervals {
        if len(res) > 0 && v[0] <= res[len(res)-1][1] {
            if v[1] > res[len(res)-1][1] {
                res[len(res)-1][1] = v[1]
            }
        } else {
            res = append(res, v)
        }
    }
    return res
}
// 时间 O(n*log(n))，空间 O(n)
```

### 189. 轮转数组

三次翻转：整体翻转 → 前 k 个翻转 → 后面翻转。

```go
func rotate(nums []int, k int) {
    k %= len(nums)
    reverse(nums)
    reverse(nums[:k])
    reverse(nums[k:])
}

func reverse(a []int) {
    for i, j := 0, len(a)-1; i < j; i, j = i+1, j-1 {
        a[i], a[j] = a[j], a[i]
    }
}
// 时间 O(n)，空间 O(1)
```

### 238. 除自身以外数组的乘积

两次遍历，先算左边前缀积，再从右边乘上后缀积。

```go
func productExceptSelf(nums []int) []int {
    n := len(nums)
    res := make([]int, n)
    res[0] = 1
    for i := 1; i < n; i++ {
        res[i] = res[i-1] * nums[i-1]
    }
    right := 1
    for i := n - 1; i >= 0; i-- {
        res[i] *= right
        right *= nums[i]
    }
    return res
}
// 时间 O(n)，空间 O(1)（输出数组不算额外空间）
```

### 41. 缺失的第一个正数

原地哈希：把每个正数 v 放到 nums[v-1] 的位置上。

```go
func firstMissingPositive(nums []int) int {
    n := len(nums)
    for i := 0; i < n; i++ {
        for nums[i] > 0 && nums[i] <= n && nums[nums[i]-1] != nums[i] {
            nums[i], nums[nums[i]-1] = nums[nums[i]-1], nums[i]
        }
    }
    for i := 0; i < n; i++ {
        if nums[i] != i+1 {
            return i + 1
        }
    }
    return n + 1
}
// 时间 O(n)，空间 O(1)
```

这题面试出现频率很高，核心思路是把数组本身当哈希表用，nums[i] 存 i+1。while 循环看着像 O(n²) 但每个元素最多被交换一次，均摊 O(n)。

---

## 矩阵

### 73. 矩阵置零

用第一行和第一列作为标记数组，避免额外空间。

```go
func setZeroes(matrix [][]int) {
    m, n := len(matrix), len(matrix[0])
    firstRowZero, firstColZero := false, false

    for j := 0; j < n; j++ {
        if matrix[0][j] == 0 {
            firstRowZero = true
        }
    }
    for i := 0; i < m; i++ {
        if matrix[i][0] == 0 {
            firstColZero = true
        }
    }

    for i := 1; i < m; i++ {
        for j := 1; j < n; j++ {
            if matrix[i][j] == 0 {
                matrix[i][0] = 0
                matrix[0][j] = 0
            }
        }
    }

    for i := 1; i < m; i++ {
        for j := 1; j < n; j++ {
            if matrix[i][0] == 0 || matrix[0][j] == 0 {
                matrix[i][j] = 0
            }
        }
    }

    if firstRowZero {
        for j := 0; j < n; j++ {
            matrix[0][j] = 0
        }
    }
    if firstColZero {
        for i := 0; i < m; i++ {
            matrix[i][0] = 0
        }
    }
}
// 时间 O(m*n)，空间 O(1)
```

### 54. 螺旋矩阵

定义上下左右四个边界，按顺时针方向遍历，每遍历完一条边就收缩对应边界。

```go
func spiralOrder(matrix [][]int) []int {
    top, bottom := 0, len(matrix)-1
    left, right := 0, len(matrix[0])-1
    var res []int

    for top <= bottom && left <= right {
        for j := left; j <= right; j++ {
            res = append(res, matrix[top][j])
        }
        top++
        for i := top; i <= bottom; i++ {
            res = append(res, matrix[i][right])
        }
        right--
        if top <= bottom {
            for j := right; j >= left; j-- {
                res = append(res, matrix[bottom][j])
            }
            bottom--
        }
        if left <= right {
            for i := bottom; i >= top; i-- {
                res = append(res, matrix[i][left])
            }
            left++
        }
    }
    return res
}
// 时间 O(m*n)，空间 O(1)
```

### 48. 旋转图像

先沿主对角线翻转，再水平翻转。

```go
func rotateImage(matrix [][]int) {
    n := len(matrix)
    // 沿主对角线翻转
    for i := 0; i < n; i++ {
        for j := i + 1; j < n; j++ {
            matrix[i][j], matrix[j][i] = matrix[j][i], matrix[i][j]
        }
    }
    // 水平翻转
    for i := 0; i < n; i++ {
        for l, r := 0, n-1; l < r; l, r = l+1, r-1 {
            matrix[i][l], matrix[i][r] = matrix[i][r], matrix[i][l]
        }
    }
}
// 时间 O(n²)，空间 O(1)
```

### 240. 搜索二维矩阵 II

从右上角开始，大了往左走，小了往下走。

```go
func searchMatrix(matrix [][]int, target int) bool {
    m, n := len(matrix), len(matrix[0])
    i, j := 0, n-1
    for i < m && j >= 0 {
        if matrix[i][j] == target {
            return true
        } else if matrix[i][j] > target {
            j--
        } else {
            i++
        }
    }
    return false
}
// 时间 O(m+n)，空间 O(1)
```

这个思路非常巧妙——右上角的元素相当于一个 BST 的根，左边的数都比它小，下边的数都比它大。
