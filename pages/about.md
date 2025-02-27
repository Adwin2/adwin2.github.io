---
layout: page
title: About
description: 打码改变世界
keywords: Ryan Mendez
comments: true
menu: 关于
permalink: /about/
---
Entelechy
## 联系

Skill Keywords
{% for skill in site.data.skills %}

{{ skill.name }}
{% for keyword in skill.keywords %} {{ keyword }} {% endfor %}
{% endfor %}
