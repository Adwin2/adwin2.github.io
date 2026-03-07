---
layout: post
title: LeetCode Hot 100 题解（三）：链表与栈
categories: [算法]
description: LeetCode Hot 100 系列第三篇，覆盖链表和栈的经典题目。
keywords: LeetCode, 算法, Golang, 链表, 栈
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

Hot 100 系列第三篇，链表和栈。链表题的关键是画图 + dummy 节点，栈的关键是想清楚单调性。

---

## 链表

### 160. 相交链表

两个指针分别从 A、B 头出发，走到末尾后跳到对方头部继续走，最终会在交点相遇。

```go
func getIntersectionNode(headA, headB *ListNode) *ListNode {
    a, b := headA, headB
    for a != b {
        if a != nil {
            a = a.Next
        } else {
            a = headB
        }
        if b != nil {
            b = b.Next
        } else {
            b = headA
        }
    }
    return a
}
// 时间 O(m+n)，空间 O(1)
```

### 206. 反转链表

迭代法，三指针原地翻转。

```go
func reverseList(head *ListNode) *ListNode {
    var prev *ListNode
    cur := head
    for cur != nil {
        next := cur.Next
        cur.Next = prev
        prev = cur
        cur = next
    }
    return prev
}
// 时间 O(n)，空间 O(1)
```

### 234. 回文链表

快慢指针找中点 → 反转后半段 → 逐一比较。

```go
func isPalindrome(head *ListNode) bool {
    slow, fast := head, head
    for fast != nil && fast.Next != nil {
        slow = slow.Next
        fast = fast.Next.Next
    }
    // 反转后半段
    var prev *ListNode
    for slow != nil {
        next := slow.Next
        slow.Next = prev
        prev = slow
        slow = next
    }
    // 比较
    for prev != nil {
        if head.Val != prev.Val {
            return false
        }
        head = head.Next
        prev = prev.Next
    }
    return true
}
// 时间 O(n)，空间 O(1)
```

### 141. 环形链表

快慢指针，快指针每次走两步，有环必相遇。

```go
func hasCycle(head *ListNode) bool {
    slow, fast := head, head
    for fast != nil && fast.Next != nil {
        slow = slow.Next
        fast = fast.Next.Next
        if slow == fast {
            return true
        }
    }
    return false
}
// 时间 O(n)，空间 O(1)
```

### 142. 环形链表 II

相遇后，一个指针回到头部，两个指针同速前进，再次相遇就是环的入口。

```go
func detectCycle(head *ListNode) *ListNode {
    slow, fast := head, head
    for fast != nil && fast.Next != nil {
        slow = slow.Next
        fast = fast.Next.Next
        if slow == fast {
            p := head
            for p != slow {
                p = p.Next
                slow = slow.Next
            }
            return p
        }
    }
    return nil
}
// 时间 O(n)，空间 O(1)
```

### 21. 合并两个有序链表

dummy 节点 + 逐一比较。

```go
func mergeTwoLists(list1 *ListNode, list2 *ListNode) *ListNode {
    dummy := &ListNode{}
    cur := dummy
    for list1 != nil && list2 != nil {
        if list1.Val <= list2.Val {
            cur.Next = list1
            list1 = list1.Next
        } else {
            cur.Next = list2
            list2 = list2.Next
        }
        cur = cur.Next
    }
    if list1 != nil {
        cur.Next = list1
    } else {
        cur.Next = list2
    }
    return dummy.Next
}
// 时间 O(m+n)，空间 O(1)
```

### 2. 两数相加

模拟竖式加法，注意进位。

```go
func addTwoNumbers(l1 *ListNode, l2 *ListNode) *ListNode {
    dummy := &ListNode{}
    cur := dummy
    carry := 0
    for l1 != nil || l2 != nil || carry > 0 {
        sum := carry
        if l1 != nil {
            sum += l1.Val
            l1 = l1.Next
        }
        if l2 != nil {
            sum += l2.Val
            l2 = l2.Next
        }
        carry = sum / 10
        cur.Next = &ListNode{Val: sum % 10}
        cur = cur.Next
    }
    return dummy.Next
}
// 时间 O(max(m,n))，空间 O(1)
```

### 19. 删除链表的倒数第 N 个结点

快指针先走 N 步，然后快慢一起走，慢指针到达待删除节点的前一个。

```go
func removeNthFromEnd(head *ListNode, n int) *ListNode {
    dummy := &ListNode{Next: head}
    fast, slow := dummy, dummy
    for i := 0; i <= n; i++ {
        fast = fast.Next
    }
    for fast != nil {
        fast = fast.Next
        slow = slow.Next
    }
    slow.Next = slow.Next.Next
    return dummy.Next
}
// 时间 O(n)，空间 O(1)
```

### 24. 两两交换链表中的节点

每次取两个节点交换，注意指针关系。

```go
func swapPairs(head *ListNode) *ListNode {
    dummy := &ListNode{Next: head}
    prev := dummy
    for prev.Next != nil && prev.Next.Next != nil {
        a := prev.Next
        b := a.Next
        prev.Next = b
        a.Next = b.Next
        b.Next = a
        prev = a
    }
    return dummy.Next
}
// 时间 O(n)，空间 O(1)
```

### 25. K 个一组翻转链表

先计数够不够 K 个，够就翻转这一组，不够直接结束。

```go
func reverseKGroup(head *ListNode, k int) *ListNode {
    dummy := &ListNode{Next: head}
    prev := dummy
    for {
        // 检查剩余节点是否够 k 个
        check := prev
        for i := 0; i < k; i++ {
            check = check.Next
            if check == nil {
                return dummy.Next
            }
        }
        // 翻转 k 个节点
        cur := prev.Next
        for i := 0; i < k-1; i++ {
            next := cur.Next
            cur.Next = next.Next
            next.Next = prev.Next
            prev.Next = next
        }
        prev = cur
    }
}
// 时间 O(n)，空间 O(1)
```

### 138. 随机链表的复制

三步法：在每个节点后面插入复制节点 → 设置 random 指针 → 拆分。

```go
func copyRandomList(head *Node) *Node {
    if head == nil {
        return nil
    }
    // 1. 在每个节点后面插入副本
    cur := head
    for cur != nil {
        copy := &Node{Val: cur.Val, Next: cur.Next}
        cur.Next = copy
        cur = copy.Next
    }
    // 2. 设置 random 指针
    cur = head
    for cur != nil {
        if cur.Random != nil {
            cur.Next.Random = cur.Random.Next
        }
        cur = cur.Next.Next
    }
    // 3. 拆分
    dummy := &Node{}
    copyCur := dummy
    cur = head
    for cur != nil {
        copyCur.Next = cur.Next
        copyCur = copyCur.Next
        cur.Next = copyCur.Next
        cur = cur.Next
    }
    return dummy.Next
}
// 时间 O(n)，空间 O(1)（不算输出）
```

### 148. 排序链表

归并排序：快慢指针找中点，递归排序两半，合并。

```go
func sortList(head *ListNode) *ListNode {
    if head == nil || head.Next == nil {
        return head
    }
    slow, fast := head, head.Next
    for fast != nil && fast.Next != nil {
        slow = slow.Next
        fast = fast.Next.Next
    }
    mid := slow.Next
    slow.Next = nil
    left := sortList(head)
    right := sortList(mid)
    return mergeTwoLists(left, right)
}
// 时间 O(n*log(n))，空间 O(log(n))（递归栈）
```

### 23. 合并 K 个升序链表

用最小堆，每次取堆顶最小节点，再把它的 next 入堆。

```go
func mergeKLists(lists []*ListNode) *ListNode {
    h := &minHeap{}
    for _, l := range lists {
        if l != nil {
            heap.Push(h, l)
        }
    }
    dummy := &ListNode{}
    cur := dummy
    for h.Len() > 0 {
        node := heap.Pop(h).(*ListNode)
        cur.Next = node
        cur = cur.Next
        if node.Next != nil {
            heap.Push(h, node.Next)
        }
    }
    return dummy.Next
}

type minHeap []*ListNode

func (h minHeap) Len() int            { return len(h) }
func (h minHeap) Less(i, j int) bool   { return h[i].Val < h[j].Val }
func (h minHeap) Swap(i, j int)        { h[i], h[j] = h[j], h[i] }
func (h *minHeap) Push(x interface{})  { *h = append(*h, x.(*ListNode)) }
func (h *minHeap) Pop() interface{} {
    old := *h
    n := len(old)
    x := old[n-1]
    *h = old[:n-1]
    return x
}
// 时间 O(n*log(k))，空间 O(k)
```

### 146. LRU 缓存

哈希表 + 双向链表。Get/Put 都是 O(1)。

```go
type LRUCache struct {
    cap        int
    m          map[int]*node
    head, tail *node
}

type node struct {
    key, val   int
    prev, next *node
}

func Constructor(capacity int) LRUCache {
    head := &node{}
    tail := &node{}
    head.next = tail
    tail.prev = head
    return LRUCache{cap: capacity, m: make(map[int]*node), head: head, tail: tail}
}

func (c *LRUCache) Get(key int) int {
    if n, ok := c.m[key]; ok {
        c.moveToFront(n)
        return n.val
    }
    return -1
}

func (c *LRUCache) Put(key int, value int) {
    if n, ok := c.m[key]; ok {
        n.val = value
        c.moveToFront(n)
        return
    }
    n := &node{key: key, val: value}
    c.m[key] = n
    c.addToFront(n)
    if len(c.m) > c.cap {
        removed := c.tail.prev
        c.remove(removed)
        delete(c.m, removed.key)
    }
}

func (c *LRUCache) addToFront(n *node) {
    n.next = c.head.next
    n.prev = c.head
    c.head.next.prev = n
    c.head.next = n
}

func (c *LRUCache) remove(n *node) {
    n.prev.next = n.next
    n.next.prev = n.prev
}

func (c *LRUCache) moveToFront(n *node) {
    c.remove(n)
    c.addToFront(n)
}
// Get/Put 均 O(1)
```

---

## 栈

### 20. 有效的括号

遇到左括号入栈，遇到右括号看栈顶是否匹配。

```go
func isValid(s string) bool {
    stack := []byte{}
    pairs := map[byte]byte{')': '(', ']': '[', '}': '{'}
    for i := range s {
        if s[i] == '(' || s[i] == '[' || s[i] == '{' {
            stack = append(stack, s[i])
        } else {
            if len(stack) == 0 || stack[len(stack)-1] != pairs[s[i]] {
                return false
            }
            stack = stack[:len(stack)-1]
        }
    }
    return len(stack) == 0
}
// 时间 O(n)，空间 O(n)
```

### 155. 最小栈

辅助栈同步存当前最小值。

```go
type MinStack struct {
    stack    []int
    minStack []int
}

func Constructor() MinStack {
    return MinStack{}
}

func (s *MinStack) Push(val int) {
    s.stack = append(s.stack, val)
    if len(s.minStack) == 0 || val <= s.minStack[len(s.minStack)-1] {
        s.minStack = append(s.minStack, val)
    }
}

func (s *MinStack) Pop() {
    top := s.stack[len(s.stack)-1]
    s.stack = s.stack[:len(s.stack)-1]
    if top == s.minStack[len(s.minStack)-1] {
        s.minStack = s.minStack[:len(s.minStack)-1]
    }
}

func (s *MinStack) Top() int {
    return s.stack[len(s.stack)-1]
}

func (s *MinStack) GetMin() int {
    return s.minStack[len(s.minStack)-1]
}
// 所有操作 O(1)
```

### 394. 字符串解码

两个栈，一个存数字一个存字符串。遇到 `[` 入栈，遇到 `]` 出栈拼接。

```go
func decodeString(s string) string {
    var numStack []int
    var strStack []string
    cur := ""
    num := 0
    for _, ch := range s {
        if ch >= '0' && ch <= '9' {
            num = num*10 + int(ch-'0')
        } else if ch == '[' {
            numStack = append(numStack, num)
            strStack = append(strStack, cur)
            num = 0
            cur = ""
        } else if ch == ']' {
            n := numStack[len(numStack)-1]
            numStack = numStack[:len(numStack)-1]
            prev := strStack[len(strStack)-1]
            strStack = strStack[:len(strStack)-1]
            cur = prev + strings.Repeat(cur, n)
        } else {
            cur += string(ch)
        }
    }
    return cur
}
// 时间 O(输出长度)，空间 O(嵌套深度)
```

### 739. 每日温度

单调递减栈，栈里存下标。

```go
func dailyTemperatures(temperatures []int) []int {
    n := len(temperatures)
    ans := make([]int, n)
    var stack []int
    for i, t := range temperatures {
        for len(stack) > 0 && temperatures[stack[len(stack)-1]] < t {
            j := stack[len(stack)-1]
            stack = stack[:len(stack)-1]
            ans[j] = i - j
        }
        stack = append(stack, i)
    }
    return ans
}
// 时间 O(n)，空间 O(n)
```

### 84. 柱状图中最大的矩形

单调递增栈。对于每根柱子，找左右第一个比它矮的位置，就能算出以它为高度的最大矩形。

```go
func largestRectangleArea(heights []int) int {
    n := len(heights)
    var stack []int
    ans := 0
    for i := 0; i <= n; i++ {
        h := 0
        if i < n {
            h = heights[i]
        }
        for len(stack) > 0 && heights[stack[len(stack)-1]] > h {
            height := heights[stack[len(stack)-1]]
            stack = stack[:len(stack)-1]
            width := i
            if len(stack) > 0 {
                width = i - stack[len(stack)-1] - 1
            }
            area := height * width
            if area > ans {
                ans = area
            }
        }
        stack = append(stack, i)
    }
    return ans
}
// 时间 O(n)，空间 O(n)
```
