---
layout: page
title: About
description: Backend Engineer | Golang, C++, Python
keywords: Ryan Mendez, Backend Engineer, Golang, C++, Python
comments: true
menu: 关于
permalink: /about/
---

## Ryan Mendez

后端工程师，坐标太原。专注于 Golang、C++ 和 Python 方向的系统编程与后端开发。

对高性能系统、分布式架构和简洁的代码设计有持续的热情。永远秉持主体性。

### 技术方向

- **后端开发**：使用 Go 和 C++ 构建高可用服务
- **系统编程**：性能优化、内存管理、并发控制
- **工程实践**：设计模式、代码审查、CI/CD、Git 工作流

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
