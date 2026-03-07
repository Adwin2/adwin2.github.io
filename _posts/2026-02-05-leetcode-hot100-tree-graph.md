---
layout: post
title: LeetCode Hot 100 题解（四）：二叉树与图
categories: [算法]
description: LeetCode Hot 100 系列第四篇，覆盖二叉树遍历、构造、路径和图论相关题目。
keywords: LeetCode, 算法, Golang, 二叉树, 图, BFS, DFS
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

Hot 100 系列第四篇，二叉树和图。树的题目递归居多，关键是定义好递归函数的返回值含义。

---

## 二叉树

### 94. 二叉树的中序遍历

迭代法，用栈模拟递归。

```go
func inorderTraversal(root *TreeNode) []int {
    var res []int
    var stack []*TreeNode
    cur := root
    for cur != nil || len(stack) > 0 {
        for cur != nil {
            stack = append(stack, cur)
            cur = cur.Left
        }
        cur = stack[len(stack)-1]
        stack = stack[:len(stack)-1]
        res = append(res, cur.Val)
        cur = cur.Right
    }
    return res
}
// 时间 O(n)，空间 O(n)
```

### 104. 二叉树的最大深度

```go
func maxDepth(root *TreeNode) int {
    if root == nil {
        return 0
    }
    l := maxDepth(root.Left)
    r := maxDepth(root.Right)
    if l > r {
        return l + 1
    }
    return r + 1
}
// 时间 O(n)，空间 O(h)
```

### 226. 翻转二叉树

```go
func invertTree(root *TreeNode) *TreeNode {
    if root == nil {
        return nil
    }
    root.Left, root.Right = invertTree(root.Right), invertTree(root.Left)
    return root
}
// 时间 O(n)，空间 O(h)
```

### 101. 对称二叉树

递归比较左右子树是否镜像。

```go
func isSymmetric(root *TreeNode) bool {
    return isMirror(root.Left, root.Right)
}

func isMirror(a, b *TreeNode) bool {
    if a == nil || b == nil {
        return a == b
    }
    return a.Val == b.Val && isMirror(a.Left, b.Right) && isMirror(a.Right, b.Left)
}
// 时间 O(n)，空间 O(h)
```

### 543. 二叉树的直径

任意路径长度 = 经过某个节点时的左子树深度 + 右子树深度。递归求深度的过程中更新最大值。

```go
func diameterOfBinaryTree(root *TreeNode) int {
    ans := 0
    var depth func(*TreeNode) int
    depth = func(node *TreeNode) int {
        if node == nil {
            return 0
        }
        l := depth(node.Left)
        r := depth(node.Right)
        if l+r > ans {
            ans = l + r
        }
        if l > r {
            return l + 1
        }
        return r + 1
    }
    depth(root)
    return ans
}
// 时间 O(n)，空间 O(h)
```

### 102. 二叉树的层序遍历

BFS，逐层处理。

```go
func levelOrder(root *TreeNode) [][]int {
    if root == nil {
        return nil
    }
    var res [][]int
    queue := []*TreeNode{root}
    for len(queue) > 0 {
        size := len(queue)
        var level []int
        for i := 0; i < size; i++ {
            node := queue[i]
            level = append(level, node.Val)
            if node.Left != nil {
                queue = append(queue, node.Left)
            }
            if node.Right != nil {
                queue = append(queue, node.Right)
            }
        }
        queue = queue[size:]
        res = append(res, level)
    }
    return res
}
// 时间 O(n)，空间 O(n)
```

### 108. 将有序数组转换为二叉搜索树

取中间元素作根，递归构建左右子树。

```go
func sortedArrayToBST(nums []int) *TreeNode {
    if len(nums) == 0 {
        return nil
    }
    mid := len(nums) / 2
    return &TreeNode{
        Val:   nums[mid],
        Left:  sortedArrayToBST(nums[:mid]),
        Right: sortedArrayToBST(nums[mid+1:]),
    }
}
// 时间 O(n)，空间 O(log(n))
```

### 98. 验证二叉搜索树

递归传递上下界。

```go
func isValidBST(root *TreeNode) bool {
    return validate(root, math.MinInt64, math.MaxInt64)
}

func validate(node *TreeNode, low, high int) bool {
    if node == nil {
        return true
    }
    if node.Val <= low || node.Val >= high {
        return false
    }
    return validate(node.Left, low, node.Val) && validate(node.Right, node.Val, high)
}
// 时间 O(n)，空间 O(h)
```

### 230. 二叉搜索树中第 K 小的元素

中序遍历 BST 就是有序的，遍历到第 K 个停。

```go
func kthSmallest(root *TreeNode, k int) int {
    var stack []*TreeNode
    cur := root
    for {
        for cur != nil {
            stack = append(stack, cur)
            cur = cur.Left
        }
        cur = stack[len(stack)-1]
        stack = stack[:len(stack)-1]
        k--
        if k == 0 {
            return cur.Val
        }
        cur = cur.Right
    }
}
// 时间 O(H+k)，空间 O(H)
```

### 199. 二叉树的右视图

层序遍历取每层最后一个。

```go
func rightSideView(root *TreeNode) []int {
    if root == nil {
        return nil
    }
    var res []int
    queue := []*TreeNode{root}
    for len(queue) > 0 {
        size := len(queue)
        for i := 0; i < size; i++ {
            node := queue[i]
            if i == size-1 {
                res = append(res, node.Val)
            }
            if node.Left != nil {
                queue = append(queue, node.Left)
            }
            if node.Right != nil {
                queue = append(queue, node.Right)
            }
        }
        queue = queue[size:]
    }
    return res
}
// 时间 O(n)，空间 O(n)
```

### 114. 二叉树展开为链表

后序遍历的思路：先展开右子树，再展开左子树，然后把左子树接到右边。

```go
func flatten(root *TreeNode) {
    for root != nil {
        if root.Left != nil {
            // 找左子树最右节点
            prev := root.Left
            for prev.Right != nil {
                prev = prev.Right
            }
            prev.Right = root.Right
            root.Right = root.Left
            root.Left = nil
        }
        root = root.Right
    }
}
// 时间 O(n)，空间 O(1)
```

### 105. 从前序与中序遍历序列构造二叉树

前序第一个是根，在中序中找到根的位置，左边是左子树，右边是右子树。

```go
func buildTree(preorder []int, inorder []int) *TreeNode {
    if len(preorder) == 0 {
        return nil
    }
    root := &TreeNode{Val: preorder[0]}
    idx := 0
    for i, v := range inorder {
        if v == preorder[0] {
            idx = i
            break
        }
    }
    root.Left = buildTree(preorder[1:idx+1], inorder[:idx])
    root.Right = buildTree(preorder[idx+1:], inorder[idx+1:])
    return root
}
// 时间 O(n²) 最坏，用 map 优化到 O(n)
```

### 437. 路径总和 III

前缀和 + DFS。类似 560 题的思路，在树上做前缀和。

```go
func pathSum(root *TreeNode, targetSum int) int {
    prefix := map[int64]int{0: 1}
    var dfs func(*TreeNode, int64) int
    dfs = func(node *TreeNode, cur int64) int {
        if node == nil {
            return 0
        }
        cur += int64(node.Val)
        cnt := prefix[cur-int64(targetSum)]
        prefix[cur]++
        cnt += dfs(node.Left, cur) + dfs(node.Right, cur)
        prefix[cur]--
        return cnt
    }
    return dfs(root, 0)
}
// 时间 O(n)，空间 O(n)
```

### 236. 二叉树的最近公共祖先

递归：当前节点为 nil 或等于 p/q 就返回，左右子树都找到就是当前节点。

```go
func lowestCommonAncestor(root, p, q *TreeNode) *TreeNode {
    if root == nil || root == p || root == q {
        return root
    }
    left := lowestCommonAncestor(root.Left, p, q)
    right := lowestCommonAncestor(root.Right, p, q)
    if left != nil && right != nil {
        return root
    }
    if left != nil {
        return left
    }
    return right
}
// 时间 O(n)，空间 O(h)
```

### 124. 二叉树中的最大路径和

递归返回经过当前节点的单边最大贡献值，过程中更新全局最大路径和。

```go
func maxPathSum(root *TreeNode) int {
    ans := math.MinInt64
    var dfs func(*TreeNode) int
    dfs = func(node *TreeNode) int {
        if node == nil {
            return 0
        }
        left := max(0, dfs(node.Left))
        right := max(0, dfs(node.Right))
        // 经过当前节点的完整路径
        if node.Val+left+right > ans {
            ans = node.Val + left + right
        }
        // 返回单边最大贡献
        return node.Val + max(left, right)
    }
    dfs(root)
    return ans
}
// 时间 O(n)，空间 O(h)
```

---

## 图论

### 200. 岛屿数量

DFS 或 BFS，遇到 '1' 就把整个岛标记为已访问。

```go
func numIslands(grid [][]byte) int {
    m, n := len(grid), len(grid[0])
    count := 0
    var dfs func(i, j int)
    dfs = func(i, j int) {
        if i < 0 || i >= m || j < 0 || j >= n || grid[i][j] != '1' {
            return
        }
        grid[i][j] = '0'
        dfs(i+1, j)
        dfs(i-1, j)
        dfs(i, j+1)
        dfs(i, j-1)
    }
    for i := 0; i < m; i++ {
        for j := 0; j < n; j++ {
            if grid[i][j] == '1' {
                count++
                dfs(i, j)
            }
        }
    }
    return count
}
// 时间 O(m*n)，空间 O(m*n) 最坏
```

### 994. 腐烂的橘子

多源 BFS，所有腐烂橘子同时入队。

{% raw %}
```go
func orangesRotting(grid [][]int) int {
    m, n := len(grid), len(grid[0])
    var queue [][2]int
    fresh := 0
    for i := 0; i < m; i++ {
        for j := 0; j < n; j++ {
            if grid[i][j] == 2 {
                queue = append(queue, [2]int{i, j})
            } else if grid[i][j] == 1 {
                fresh++
            }
        }
    }
    if fresh == 0 {
        return 0
    }
    dirs := [4][2]int{{1, 0}, {-1, 0}, {0, 1}, {0, -1}}
    minutes := 0
    for len(queue) > 0 {
        size := len(queue)
        for k := 0; k < size; k++ {
            p := queue[k]
            for _, d := range dirs {
                ni, nj := p[0]+d[0], p[1]+d[1]
                if ni >= 0 && ni < m && nj >= 0 && nj < n && grid[ni][nj] == 1 {
                    grid[ni][nj] = 2
                    fresh--
                    queue = append(queue, [2]int{ni, nj})
                }
            }
        }
        queue = queue[size:]
        minutes++
    }
    if fresh > 0 {
        return -1
    }
    return minutes - 1
}
// 时间 O(m*n)，空间 O(m*n)
```
{% endraw %}

### 207. 课程表

拓扑排序（BFS），入度为 0 的先修完。

```go
func canFinish(numCourses int, prerequisites [][]int) bool {
    indegree := make([]int, numCourses)
    graph := make([][]int, numCourses)
    for _, p := range prerequisites {
        graph[p[1]] = append(graph[p[1]], p[0])
        indegree[p[0]]++
    }
    var queue []int
    for i, d := range indegree {
        if d == 0 {
            queue = append(queue, i)
        }
    }
    count := 0
    for len(queue) > 0 {
        cur := queue[0]
        queue = queue[1:]
        count++
        for _, next := range graph[cur] {
            indegree[next]--
            if indegree[next] == 0 {
                queue = append(queue, next)
            }
        }
    }
    return count == numCourses
}
// 时间 O(V+E)，空间 O(V+E)
```

### 208. 实现 Trie（前缀树）

```go
type Trie struct {
    children [26]*Trie
    isEnd    bool
}

func Constructor() Trie {
    return Trie{}
}

func (t *Trie) Insert(word string) {
    cur := t
    for _, ch := range word {
        idx := ch - 'a'
        if cur.children[idx] == nil {
            cur.children[idx] = &Trie{}
        }
        cur = cur.children[idx]
    }
    cur.isEnd = true
}

func (t *Trie) Search(word string) bool {
    node := t.searchPrefix(word)
    return node != nil && node.isEnd
}

func (t *Trie) StartsWith(prefix string) bool {
    return t.searchPrefix(prefix) != nil
}

func (t *Trie) searchPrefix(s string) *Trie {
    cur := t
    for _, ch := range s {
        idx := ch - 'a'
        if cur.children[idx] == nil {
            return nil
        }
        cur = cur.children[idx]
    }
    return cur
}
// Insert/Search/StartsWith 均 O(L)，L 为字符串长度
```
