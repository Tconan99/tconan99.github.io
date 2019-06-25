---
layout: page
title: About
description: 逗逼使我快乐
keywords: 薛恩鹏
comments: true
menu: 关于
permalink: /about/
---

我是薛恩鹏，一只技术宅。

## 联系

{% for website in site.data.social %}
* {{ website.sitename }}：[@{{ website.name }}]({{ website.url }})
{% endfor %}

## Skill Keywords

{% for category in site.data.skills %}
### {{ category.name }}
<div class="btn-inline">
{% for keyword in category.keywords %}
<button class="btn btn-outline" type="button">{{ keyword }}</button>
{% endfor %}
</div>
{% endfor %}
