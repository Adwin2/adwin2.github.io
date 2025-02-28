---
layout: page
title: About
description: 热爱所以坚持
keywords: Ryan Mendez
comments: true
menu: 关于
permalink: /about/
---
Ryan Mendez

永远秉持主体性
## 联系

## Skill Keywords
{% for skill in site.data.skills %}
### {{ skill.name }}
<div class="btn-inline">
{% for keyword in skill.keywords %}
<button class="btn btn-outline" type="button">{{ keyword }}</button>
{% endfor %}
</div>
{% endfor %}
