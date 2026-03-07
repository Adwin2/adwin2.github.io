---
layout: post
title: LeetCode Hot 100 题解（五）：二分、回溯、贪心与堆
categories: [算法]
description: LeetCode Hot 100 系列第五篇，覆盖二分查找、回溯、贪心、堆和技巧类题目。
keywords: LeetCode, 算法, Golang, 二分查找, 回溯, 贪心, 堆
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

Hot 100 系列第五篇，二分查找、回溯、贪心、堆和技巧。

---

## 二分查找

### 35. 搜索插入位置

标准二分，找第一个 >= target 的位置。

```go
func searchInsert(nums []int, target int) int {
    l, r := 0, len(nums)
    for l < r {
        mid := l + (r-l)/2
        if nums[mid] < target {
            l = mid + 1
        } else {
            r = mid
        }
    }
    return l
}
// 时间 O(log(n))，空间 O(1)
```

### 74. 搜索二维矩阵

把二维矩阵看成一维有序数组做二分。

```go
func searchMatrix(matrix [][]int, target int) bool {
    m, n := len(matrix), len(matrix[0])
    l, r := 0, m*n
    for l < r {
        mid := l + (r-l)/2
        val := matrix[mid/n][mid%n]
        if val == target {
            return true
        } else if val < target {
            l = mid + 1
        } else {
            r = mid
        }
    }
    return false
}
// 时间 O(log(m*n))，空间 O(1)
```

### 34. 在排序数组中查找元素的第一个和最后一个位置

两次二分，分别找左边界和右边界。

```go
func searchRange(nums []int, target int) []int {
    left := lowerBound(nums, target)
    if left == len(nums) || nums[left] != target {
        return []int{-1, -1}
    }
    right := lowerBound(nums, target+1) - 1
    return []int{left, right}
}

func lowerBound(nums []int, target int) int {
    l, r := 0, len(nums)
    for l < r {
        mid := l + (r-l)/2
        if nums[mid] < target {
            l = mid + 1
        } else {
            r = mid
        }
    }
    return l
}
// 时间 O(log(n))，空间 O(1)
```

### 33. 搜索旋转排序数组

二分时判断哪半边是有序的，在有序的那半边判断 target 是否在范围内。

```go
func search(nums []int, target int) int {
    l, r := 0, len(nums)-1
    for l <= r {
        mid := l + (r-l)/2
        if nums[mid] == target {
            return mid
        }
        if nums[l] <= nums[mid] { // 左半有序
            if nums[l] <= target && target < nums[mid] {
                r = mid - 1
            } else {
                l = mid + 1
            }
        } else { // 右半有序
            if nums[mid] < target && target <= nums[r] {
                l = mid + 1
            } else {
                r = mid - 1
            }
        }
    }
    return -1
}
// 时间 O(log(n))，空间 O(1)
```

### 153. 寻找旋转排序数组中的最小值

和最后一个元素比较来决定搜索方向。

```go
func findMin(nums []int) int {
    l, r := 0, len(nums)-1
    for l < r {
        mid := l + (r-l)/2
        if nums[mid] > nums[r] {
            l = mid + 1
        } else {
            r = mid
        }
    }
    return nums[l]
}
// 时间 O(log(n))，空间 O(1)
```

### 4. 寻找两个正序数组的中位数

二分找分割线位置，使得左半部分的最大值 <= 右半部分的最小值。

```go
func findMedianSortedArrays(nums1 []int, nums2 []int) float64 {
    if len(nums1) > len(nums2) {
        nums1, nums2 = nums2, nums1
    }
    m, n := len(nums1), len(nums2)
    l, r := 0, m
    for l <= r {
        i := l + (r-l)/2
        j := (m+n+1)/2 - i
        leftMax1, leftMax2 := math.MinInt64, math.MinInt64
        rightMin1, rightMin2 := math.MaxInt64, math.MaxInt64
        if i > 0 {
            leftMax1 = nums1[i-1]
        }
        if j > 0 {
            leftMax2 = nums2[j-1]
        }
        if i < m {
            rightMin1 = nums1[i]
        }
        if j < n {
            rightMin2 = nums2[j]
        }
        if leftMax1 <= rightMin2 && leftMax2 <= rightMin1 {
            leftMax := max(leftMax1, leftMax2)
            if (m+n)%2 == 1 {
                return float64(leftMax)
            }
            rightMin := min(rightMin1, rightMin2)
            return float64(leftMax+rightMin) / 2.0
        } else if leftMax1 > rightMin2 {
            r = i - 1
        } else {
            l = i + 1
        }
    }
    return 0
}
// 时间 O(log(min(m,n)))，空间 O(1)
```

---

## 回溯

### 46. 全排列

经典回溯，用 visited 数组标记已使用的元素。

```go
func permute(nums []int) [][]int {
    var res [][]int
    var path []int
    used := make([]bool, len(nums))
    var backtrack func()
    backtrack = func() {
        if len(path) == len(nums) {
            tmp := make([]int, len(path))
            copy(tmp, path)
            res = append(res, tmp)
            return
        }
        for i, v := range nums {
            if used[i] {
                continue
            }
            used[i] = true
            path = append(path, v)
            backtrack()
            path = path[:len(path)-1]
            used[i] = false
        }
    }
    backtrack()
    return res
}
// 时间 O(n*n!)，空间 O(n)
```

### 78. 子集

每个元素选或不选。

```go
func subsets(nums []int) [][]int {
    var res [][]int
    var path []int
    var backtrack func(start int)
    backtrack = func(start int) {
        tmp := make([]int, len(path))
        copy(tmp, path)
        res = append(res, tmp)
        for i := start; i < len(nums); i++ {
            path = append(path, nums[i])
            backtrack(i + 1)
            path = path[:len(path)-1]
        }
    }
    backtrack(0)
    return res
}
// 时间 O(n*2^n)，空间 O(n)
```

### 17. 电话号码的字母组合

映射表 + 回溯。

```go
func letterCombinations(digits string) []string {
    if len(digits) == 0 {
        return nil
    }
    m := map[byte]string{
        '2': "abc", '3': "def", '4': "ghi", '5': "jkl",
        '6': "mno", '7': "pqrs", '8': "tuv", '9': "wxyz",
    }
    var res []string
    var backtrack func(idx int, cur []byte)
    backtrack = func(idx int, cur []byte) {
        if idx == len(digits) {
            res = append(res, string(cur))
            return
        }
        for _, ch := range m[digits[idx]] {
            backtrack(idx+1, append(cur, byte(ch)))
        }
    }
    backtrack(0, nil)
    return res
}
```

### 39. 组合总和

同一个数可以重复选，所以递归不是 i+1 而是 i。

```go
func combinationSum(candidates []int, target int) [][]int {
    sort.Ints(candidates)
    var res [][]int
    var path []int
    var backtrack func(start, remain int)
    backtrack = func(start, remain int) {
        if remain == 0 {
            tmp := make([]int, len(path))
            copy(tmp, path)
            res = append(res, tmp)
            return
        }
        for i := start; i < len(candidates); i++ {
            if candidates[i] > remain {
                break
            }
            path = append(path, candidates[i])
            backtrack(i, remain-candidates[i])
            path = path[:len(path)-1]
        }
    }
    backtrack(0, target)
    return res
}
```

### 22. 括号生成

回溯，维护左括号和右括号的数量。

```go
func generateParenthesis(n int) []string {
    var res []string
    var backtrack func(cur []byte, open, close int)
    backtrack = func(cur []byte, open, close int) {
        if len(cur) == 2*n {
            res = append(res, string(cur))
            return
        }
        if open < n {
            backtrack(append(cur, '('), open+1, close)
        }
        if close < open {
            backtrack(append(cur, ')'), open, close+1)
        }
    }
    backtrack(nil, 0, 0)
    return res
}
```

### 79. 单词搜索

在网格上做 DFS 回溯。

```go
func exist(board [][]byte, word string) bool {
    m, n := len(board), len(board[0])
    var dfs func(i, j, k int) bool
    dfs = func(i, j, k int) bool {
        if k == len(word) {
            return true
        }
        if i < 0 || i >= m || j < 0 || j >= n || board[i][j] != word[k] {
            return false
        }
        tmp := board[i][j]
        board[i][j] = '#'
        found := dfs(i+1, j, k+1) || dfs(i-1, j, k+1) ||
            dfs(i, j+1, k+1) || dfs(i, j-1, k+1)
        board[i][j] = tmp
        return found
    }
    for i := 0; i < m; i++ {
        for j := 0; j < n; j++ {
            if dfs(i, j, 0) {
                return true
            }
        }
    }
    return false
}
// 时间 O(m*n*3^L)，空间 O(L)
```

### 131. 分割回文串

回溯 + 判断回文。

```go
func partition(s string) [][]string {
    var res [][]string
    var path []string
    var backtrack func(start int)
    backtrack = func(start int) {
        if start == len(s) {
            tmp := make([]string, len(path))
            copy(tmp, path)
            res = append(res, tmp)
            return
        }
        for end := start + 1; end <= len(s); end++ {
            sub := s[start:end]
            if isPalindrome(sub) {
                path = append(path, sub)
                backtrack(end)
                path = path[:len(path)-1]
            }
        }
    }
    backtrack(0)
    return res
}

func isPalindrome(s string) bool {
    for i, j := 0, len(s)-1; i < j; i, j = i+1, j-1 {
        if s[i] != s[j] {
            return false
        }
    }
    return true
}
```

### 51. N 皇后

逐行放置，用三个集合记录列、左对角线、右对角线的占用情况。

```go
func solveNQueens(n int) [][]string {
    var res [][]string
    board := make([][]byte, n)
    for i := range board {
        board[i] = make([]byte, n)
        for j := range board[i] {
            board[i][j] = '.'
        }
    }
    cols := make(map[int]bool)
    diag1 := make(map[int]bool) // 主对角线 i-j
    diag2 := make(map[int]bool) // 副对角线 i+j

    var backtrack func(row int)
    backtrack = func(row int) {
        if row == n {
            snapshot := make([]string, n)
            for i := range board {
                snapshot[i] = string(board[i])
            }
            res = append(res, snapshot)
            return
        }
        for col := 0; col < n; col++ {
            if cols[col] || diag1[row-col] || diag2[row+col] {
                continue
            }
            board[row][col] = 'Q'
            cols[col] = true
            diag1[row-col] = true
            diag2[row+col] = true
            backtrack(row + 1)
            board[row][col] = '.'
            cols[col] = false
            diag1[row-col] = false
            diag2[row+col] = false
        }
    }
    backtrack(0)
    return res
}
```

---

## 贪心

### 121. 买卖股票的最佳时机

维护历史最低价，每天算当前卖出的利润。

```go
func maxProfit(prices []int) int {
    minPrice := prices[0]
    ans := 0
    for _, p := range prices {
        if p < minPrice {
            minPrice = p
        } else if p-minPrice > ans {
            ans = p - minPrice
        }
    }
    return ans
}
// 时间 O(n)，空间 O(1)
```

### 55. 跳跃游戏

维护能到达的最远位置。

```go
func canJump(nums []int) bool {
    maxReach := 0
    for i, v := range nums {
        if i > maxReach {
            return false
        }
        if i+v > maxReach {
            maxReach = i + v
        }
    }
    return true
}
// 时间 O(n)，空间 O(1)
```

### 45. 跳跃游戏 II

贪心，每次在当前范围内找能跳最远的位置。

```go
func jump(nums []int) int {
    steps, curEnd, farthest := 0, 0, 0
    for i := 0; i < len(nums)-1; i++ {
        if i+nums[i] > farthest {
            farthest = i + nums[i]
        }
        if i == curEnd {
            steps++
            curEnd = farthest
        }
    }
    return steps
}
// 时间 O(n)，空间 O(1)
```

### 763. 划分字母区间

先记录每个字母最后出现的位置，然后贪心地扩展当前片段的右边界。

```go
func partitionLabels(s string) []int {
    last := [26]int{}
    for i, ch := range s {
        last[ch-'a'] = i
    }
    var res []int
    start, end := 0, 0
    for i, ch := range s {
        if last[ch-'a'] > end {
            end = last[ch-'a']
        }
        if i == end {
            res = append(res, end-start+1)
            start = end + 1
        }
    }
    return res
}
// 时间 O(n)，空间 O(1)
```

---

## 堆

### 215. 数组中的第 K 个最大元素

用大小为 K 的最小堆，堆顶就是第 K 大。

```go
func findKthLargest(nums []int, k int) int {
    h := &intHeap{}
    for _, v := range nums {
        heap.Push(h, v)
        if h.Len() > k {
            heap.Pop(h)
        }
    }
    return (*h)[0]
}

type intHeap []int

func (h intHeap) Len() int           { return len(h) }
func (h intHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h intHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *intHeap) Push(x interface{}) { *h = append(*h, x.(int)) }
func (h *intHeap) Pop() interface{} {
    old := *h
    x := old[len(old)-1]
    *h = old[:len(old)-1]
    return x
}
// 时间 O(n*log(k))，空间 O(k)
```

### 347. 前 K 个高频元素

统计频率后用最小堆。

```go
func topKFrequent(nums []int, k int) []int {
    freq := make(map[int]int)
    for _, v := range nums {
        freq[v]++
    }
    h := &freqHeap{}
    for val, cnt := range freq {
        heap.Push(h, [2]int{val, cnt})
        if h.Len() > k {
            heap.Pop(h)
        }
    }
    res := make([]int, k)
    for i := k - 1; i >= 0; i-- {
        res[i] = heap.Pop(h).([2]int)[0]
    }
    return res
}

type freqHeap [][2]int

func (h freqHeap) Len() int            { return len(h) }
func (h freqHeap) Less(i, j int) bool  { return h[i][1] < h[j][1] }
func (h freqHeap) Swap(i, j int)       { h[i], h[j] = h[j], h[i] }
func (h *freqHeap) Push(x interface{}) { *h = append(*h, x.([2]int)) }
func (h *freqHeap) Pop() interface{} {
    old := *h
    x := old[len(old)-1]
    *h = old[:len(old)-1]
    return x
}
// 时间 O(n*log(k))，空间 O(n)
```

### 295. 数据流的中位数

大顶堆存较小的一半，小顶堆存较大的一半，堆顶就是中间的数。

```go
type MedianFinder struct {
    small *maxHeap // 较小的一半，大顶堆
    large *minIntHeap // 较大的一半，小顶堆
}

func Constructor() MedianFinder {
    return MedianFinder{small: &maxHeap{}, large: &minIntHeap{}}
}

func (mf *MedianFinder) AddNum(num int) {
    heap.Push(mf.small, num)
    heap.Push(mf.large, heap.Pop(mf.small))
    if mf.large.Len() > mf.small.Len() {
        heap.Push(mf.small, heap.Pop(mf.large))
    }
}

func (mf *MedianFinder) FindMedian() float64 {
    if mf.small.Len() > mf.large.Len() {
        return float64((*mf.small)[0])
    }
    return float64((*mf.small)[0]+(*mf.large)[0]) / 2.0
}

type maxHeap []int
func (h maxHeap) Len() int           { return len(h) }
func (h maxHeap) Less(i, j int) bool { return h[i] > h[j] }
func (h maxHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *maxHeap) Push(x interface{}) { *h = append(*h, x.(int)) }
func (h *maxHeap) Pop() interface{} {
    old := *h; x := old[len(old)-1]; *h = old[:len(old)-1]; return x
}

type minIntHeap []int
func (h minIntHeap) Len() int           { return len(h) }
func (h minIntHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h minIntHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *minIntHeap) Push(x interface{}) { *h = append(*h, x.(int)) }
func (h *minIntHeap) Pop() interface{} {
    old := *h; x := old[len(old)-1]; *h = old[:len(old)-1]; return x
}
// AddNum O(log(n))，FindMedian O(1)
```

---

## 技巧

### 136. 只出现一次的数字

全部异或，相同的数会抵消。

```go
func singleNumber(nums []int) int {
    res := 0
    for _, v := range nums {
        res ^= v
    }
    return res
}
// 时间 O(n)，空间 O(1)
```

### 169. 多数元素

Boyer-Moore 投票法。

```go
func majorityElement(nums []int) int {
    candidate, count := 0, 0
    for _, v := range nums {
        if count == 0 {
            candidate = v
        }
        if v == candidate {
            count++
        } else {
            count--
        }
    }
    return candidate
}
// 时间 O(n)，空间 O(1)
```

### 75. 颜色分类

三指针（荷兰国旗问题）。

```go
func sortColors(nums []int) {
    p0, cur, p2 := 0, 0, len(nums)-1
    for cur <= p2 {
        if nums[cur] == 0 {
            nums[p0], nums[cur] = nums[cur], nums[p0]
            p0++
            cur++
        } else if nums[cur] == 2 {
            nums[cur], nums[p2] = nums[p2], nums[cur]
            p2--
        } else {
            cur++
        }
    }
}
// 时间 O(n)，空间 O(1)
```

### 31. 下一个排列

从右往左找第一个下降的位置 i，然后找右边比 nums[i] 大的最小值与之交换，最后翻转 i+1 到末尾。

```go
func nextPermutation(nums []int) {
    n := len(nums)
    i := n - 2
    for i >= 0 && nums[i] >= nums[i+1] {
        i--
    }
    if i >= 0 {
        j := n - 1
        for nums[j] <= nums[i] {
            j--
        }
        nums[i], nums[j] = nums[j], nums[i]
    }
    // 翻转 i+1 到末尾
    for l, r := i+1, n-1; l < r; l, r = l+1, r-1 {
        nums[l], nums[r] = nums[r], nums[l]
    }
}
// 时间 O(n)，空间 O(1)
```

### 287. 寻找重复数

快慢指针（Floyd 判圈），把数组值当成 next 指针。

```go
func findDuplicate(nums []int) int {
    slow, fast := nums[0], nums[nums[0]]
    for slow != fast {
        slow = nums[slow]
        fast = nums[nums[fast]]
    }
    slow = 0
    for slow != fast {
        slow = nums[slow]
        fast = nums[fast]
    }
    return slow
}
// 时间 O(n)，空间 O(1)
```
