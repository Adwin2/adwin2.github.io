---
layout: page
title: About
description: CS Junior | Go Backend Intern @ Baidu
keywords: Ryan Mendez, Golang, Backend, Intern
comments: true
menu: 关于
permalink: /about/
---

## Ryan Mendez

大三在读，坐标太原，Go 后端方向。

有 6 个月百度 Golang 后端实习经历，目前在准备暑期实习。对后端系统开发、LLM 应用（RAG/Agent）和算法有持续的学习热情。永远秉持主体性。

### 技术方向

- **后端开发**：Go 为主，有 C++ 和 Python 基础
- **AI 工程**：学习并实践了 RAG 和 LLM Agent 的完整链路
- **算法**：系统刷完 LeetCode Hot 100

### 实习经历

- **百度** — Golang 后端实习（6 个月）

## 技能关键词

{% for skill in site.data.skills %}
### {{ skill.name }}
<div class="btn-inline">
{% for keyword in skill.keywords %}
<button class="btn btn-outline" type="button">{{ keyword }}</button>
{% endfor %}
</div>
{% endfor %}

## 联系方式

{% for social in site.data.social %}
- **{{ social.sitename }}**：[{{ social.name }}]({{ social.url }})
{% endfor %}
