---
layout: post
title: LeetCode Hot 100 题解（一）：哈希、双指针与滑动窗口
categories: [算法]
description: LeetCode Hot 100 系列第一篇，覆盖哈希表、双指针、滑动窗口和子串相关的经典题目，全部使用 Go 实现。
keywords: LeetCode, 算法, Golang, 哈希表, 双指针, 滑动窗口
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
topmost: true
---

面试前刷完了 LeetCode Hot 100，按专题整理成系列文章。所有题解用 Go 实现，重点写思路和关键细节。

本篇覆盖：哈希表、双指针、滑动窗口、子串。

---

## 哈希表

### 1. 两数之和

用哈希表存已遍历的值和下标，一次遍历即可。

```go
func twoSum(nums []int, target int) []int {
    m := make(map[int]int)
    for i, v := range nums {
        if j, ok := m[target-v]; ok {
            return []int{j, i}
        }
        m[v] = i
    }
    return nil
}
// 时间 O(n)，空间 O(n)
```

### 49. 字母异位词分组

把每个词排序后作为 key，相同 key 的词归为一组。

```go
func groupAnagrams(strs []string) [][]string {
    m := make(map[string][]string)
    for _, s := range strs {
        b := []byte(s)
        sort.Slice(b, func(i, j int) bool { return b[i] < b[j] })
        key := string(b)
        m[key] = append(m[key], s)
    }
    res := make([][]string, 0, len(m))
    for _, v := range m {
        res = append(res, v)
    }
    return res
}
// 时间 O(n * k*log(k))，空间 O(n*k)，k 为字符串平均长度
```

### 128. 最长连续序列

用 set 去重后，只从每个连续序列的起点开始计数（前一个数不在 set 中说明是起点）。

```go
func longestConsecutive(nums []int) int {
    set := make(map[int]bool, len(nums))
    for _, v := range nums {
        set[v] = true
    }
    ans := 0
    for v := range set {
        if set[v-1] {
            continue // 不是起点，跳过
        }
        length := 1
        for set[v+length] {
            length++
        }
        if length > ans {
            ans = length
        }
    }
    return ans
}
// 时间 O(n)，空间 O(n)
```

---

## 双指针

### 283. 移动零

快慢指针，慢指针指向下一个非零位置。

```go
func moveZeroes(nums []int) {
    slow := 0
    for fast := 0; fast < len(nums); fast++ {
        if nums[fast] != 0 {
            nums[slow], nums[fast] = nums[fast], nums[slow]
            slow++
        }
    }
}
// 时间 O(n)，空间 O(1)
```

### 11. 盛最多水的容器

左右指针往中间收缩，每次移动较短的那一边。

```go
func maxArea(height []int) int {
    l, r := 0, len(height)-1
    ans := 0
    for l < r {
        h := min(height[l], height[r])
        area := h * (r - l)
        if area > ans {
            ans = area
        }
        if height[l] < height[r] {
            l++
        } else {
            r--
        }
    }
    return ans
}
// 时间 O(n)，空间 O(1)
```

### 15. 三数之和

排序后固定一个数，双指针找另外两个。注意去重。

```go
func threeSum(nums []int) [][]int {
    sort.Ints(nums)
    var res [][]int
    n := len(nums)
    for i := 0; i < n-2; i++ {
        if i > 0 && nums[i] == nums[i-1] {
            continue
        }
        if nums[i] > 0 {
            break
        }
        l, r := i+1, n-1
        for l < r {
            s := nums[i] + nums[l] + nums[r]
            if s == 0 {
                res = append(res, []int{nums[i], nums[l], nums[r]})
                for l < r && nums[l] == nums[l+1] {
                    l++
                }
                for l < r && nums[r] == nums[r-1] {
                    r--
                }
                l++
                r--
            } else if s < 0 {
                l++
            } else {
                r--
            }
        }
    }
    return res
}
// 时间 O(n²)，空间 O(1)（不算输出）
```

### 42. 接雨水

左右指针 + 维护左右最大高度。当前位置能接的水取决于两侧较低的最大高度。

```go
func trap(height []int) int {
    l, r := 0, len(height)-1
    leftMax, rightMax := 0, 0
    ans := 0
    for l < r {
        if height[l] < height[r] {
            if height[l] > leftMax {
                leftMax = height[l]
            } else {
                ans += leftMax - height[l]
            }
            l++
        } else {
            if height[r] > rightMax {
                rightMax = height[r]
            } else {
                ans += rightMax - height[r]
            }
            r--
        }
    }
    return ans
}
// 时间 O(n)，空间 O(1)
```

---

## 滑动窗口

### 3. 无重复字符的最长子串

维护一个窗口，用 map 记录字符最后出现的位置，遇到重复字符就收缩左边界。

```go
func lengthOfLongestSubstring(s string) int {
    pos := make(map[byte]int)
    ans, left := 0, 0
    for right := 0; right < len(s); right++ {
        if p, ok := pos[s[right]]; ok && p >= left {
            left = p + 1
        }
        pos[s[right]] = right
        if right-left+1 > ans {
            ans = right - left + 1
        }
    }
    return ans
}
// 时间 O(n)，空间 O(字符集大小)
```

### 438. 找到字符串中所有字母异位词

固定窗口大小为 p 的长度，用计数数组比较。

```go
func findAnagrams(s string, p string) []int {
    if len(s) < len(p) {
        return nil
    }
    var sc, pc [26]int
    for i := range p {
        pc[p[i]-'a']++
        sc[s[i]-'a']++
    }
    var res []int
    if sc == pc {
        res = append(res, 0)
    }
    for i := len(p); i < len(s); i++ {
        sc[s[i]-'a']++
        sc[s[i-len(p)]-'a']--
        if sc == pc {
            res = append(res, i-len(p)+1)
        }
    }
    return res
}
// 时间 O(n)，空间 O(1)
```

---

## 子串

### 560. 和为 K 的子数组

前缀和 + 哈希表。当前前缀和为 sum，如果之前出现过 sum-k，说明中间这段子数组和为 k。

```go
func subarraySum(nums []int, k int) int {
    cnt := map[int]int{0: 1}
    sum, ans := 0, 0
    for _, v := range nums {
        sum += v
        ans += cnt[sum-k]
        cnt[sum]++
    }
    return ans
}
// 时间 O(n)，空间 O(n)
```

### 239. 滑动窗口最大值

单调递减队列，队首始终是当前窗口最大值。

```go
func maxSlidingWindow(nums []int, k int) []int {
    var q []int // 存下标
    var res []int
    for i, v := range nums {
        // 移除队尾所有比当前值小的元素
        for len(q) > 0 && nums[q[len(q)-1]] <= v {
            q = q[:len(q)-1]
        }
        q = append(q, i)
        // 移除超出窗口范围的队首
        if q[0] <= i-k {
            q = q[1:]
        }
        if i >= k-1 {
            res = append(res, nums[q[0]])
        }
    }
    return res
}
// 时间 O(n)，空间 O(k)
```

### 76. 最小覆盖子串

双指针滑动窗口，先扩展右边界直到覆盖所有字符，再收缩左边界找最小窗口。

```go
func minWindow(s string, t string) string {
    need := make(map[byte]int)
    for i := range t {
        need[t[i]]++
    }
    missing := len(t)
    left, start, minLen := 0, 0, len(s)+1

    for right := 0; right < len(s); right++ {
        if need[s[right]] > 0 {
            missing--
        }
        need[s[right]]--

        for missing == 0 {
            if right-left+1 < minLen {
                minLen = right - left + 1
                start = left
            }
            need[s[left]]++
            if need[s[left]] > 0 {
                missing++
            }
            left++
        }
    }
    if minLen > len(s) {
        return ""
    }
    return s[start : start+minLen]
}
// 时间 O(n)，空间 O(字符集大小)
```
