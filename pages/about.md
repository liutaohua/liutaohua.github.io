---
layout: page
title: About
description: Deep Learning
keywords: liutaohua, Liutaohua, 刘涛华
comments: true
menu: 关于
permalink: /about/
---

坚信实践得真知，有人用半生学习，而我用半生实践。

## 联系

{% for website in site.data.social %}
* {{ website.sitename }}：[@{{ website.name }}]({{ website.url }})
{% endfor %}