---
layout: post
title: Git Commit 信息规范
categories: Git
description: git commit 常用黑话
keywords: Git, regulation
---

# Git Commit 信息规范

---

### **1. 常用前缀（语义化提交）**
遵循 **Angular Commit Guidelines** 等规范，用于自动化生成 CHANGELOG 或触发版本管理：
• **`feat:`** 新增功能（Feature）
  ```bash
  feat: add user authentication API
  ```
• **`fix:`** 修复 Bug
  ```bash
  fix: resolve memory leak in data parser
  ```
• **`docs:`** 文档变更
  ```bash
  docs: update installation guide for macOS
  ```
• **`style:`** 代码样式调整（空格、缩进等，不涉及逻辑）
  ```bash
  style: format CSS with Prettier
  ```
• **`refactor:`** 代码重构（功能不变）
  ```bash
  refactor: simplify database connection logic
  ```
• **`perf:`** 性能优化
  ```bash
  perf: reduce image loading time by 40%
  ```
• **`test:`** 测试相关变更
  ```bash
  test: add unit tests for login module
  ```
• **`chore:`** 杂项（构建脚本、依赖更新等）
  ```bash
  chore: upgrade webpack to v5
  ```
• **`revert:`** 回滚代码
  ```bash
  revert: rollback experimental feature X
  ```

---

### **2. 常见缩写与符号**
• **`WIP`**（Work In Progress）：标记为“进行中”的提交（通常用于草稿）
  ```bash
  WIP: draft of new dashboard UI
  ```
• **`LGTM`**（Looks Good To Me）：代码审查通过（多在 Pull Request 中使用）
• **`PR`**（Pull Request）：关联到某个合并请求
  ```bash
  Update README.md (#123)  # 关联到 Issue 123
  ```
• **`#123`**：关联到 Issue 或 PR 的编号
• **`!45`**：在 GitLab 中表示关联到合并请求 45（GitHub 使用 `#`）
• **`🚑`（急救车）**：紧急修复
  ```bash
  🚑 Fix critical security vulnerability
  ```

---

### **3. 幽默/非正式表达**
（慎用，常见于个人项目或开源社区）
• **`pebcak`**（Problem Exists Between Chair And Keyboard）：调侃用户操作错误
  ```bash
  fix(login): handle pebcak error when password is empty
  ```
• **`yolo`**（You Only Live Once）：临时或冒险的代码变更
  ```bash
  yolo: disable input validation for demo
  ```
• **`DEADBEEF`**：调试占位符（源自十六进制常量 `0xDEADBEEF`）
• **`🤦♂️`**（捂脸表情）：修复低级错误
  ```bash
  🤦♂️ Fix typo in config file
  ```

---

### **4. 技术术语**
• **`BREAKING CHANGE`**：标记不兼容的变更（影响语义化版本号 `MAJOR`）
  ```bash
  feat: migrate to React 18
  BREAKING CHANGE: Drop support for legacy context API
  ```
• **`hotfix`**：紧急修复生产环境问题
• **`backport`**：将代码变更移植到旧版本分支
  ```bash
  backport: apply security patch to v1.x
  ```
• **`nit`**（Nitpick）：指无关紧要的小修改（如拼写错误）
  ```bash
  nit: fix typo in comment
  ```

---

### **5. 特殊场景**
• **`Initial commit`**：项目首次提交（经典模板）
• **`Merge branch 'dev'`**：合并分支的默认消息（可优化为更清晰的描述）
• **`wip:`**（Work In Progress）：临时保存进度
  ```bash
  wip: experiment with new caching strategy
  ```

---

### **如何写好 Commit 信息？**
1. **清晰简洁**：首行不超过 50 字符，正文每行不超过 72 字符。
2. **关联上下文**：使用 `#123` 或 `Closes #45` 关联 Issue/PR。
3. **遵循规范**：团队协作时统一前缀（如 `feat`/`fix`）。
4. **避免废话**：删除无意义的提交（如 `update`、`fix bug`）。

---

通过理解这些“黑话”，你可以更高效地阅读和编写 Commit 信息，提升协作效率！🔍
