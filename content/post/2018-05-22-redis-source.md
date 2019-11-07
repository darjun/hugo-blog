---
layout:		post
title:		"Redis源码阅读"
subtitle: 	"\"Learn from redis\""
date:		2018-05-22T9:48:00
author:		"darjun"
image:	"/img/post-bg-2015.jpg"
tags:
    - Redis源码阅读
URL: "/2018/05/22/redis-source/"
categories: [
    "Redis"
]
---

## 概述
使用Redis已经有很长一段时间了，期间也阅读过一些源码，但是一直没能坚持读完。最近一段时间比较空闲，下定决心系统地阅读一遍Redis源码，并记录下来。

Redis源码质量非常优秀，值得细细品读。此次基于Redis-2.8.19，新版本的改动主要在Redis集群方面。初次阅读，决定先不涉及这一块内容。

## 阅读顺序

阅读基本按照黄健宏建议的[顺序][0]。

首先是基本的数据结构实现：

1. [字符串][1]
2. [list][2]
3. [dict][3]
4. [skiplsit][4]

[0]: https://github.com/huangz1990/blog/blob/master/diary/2014/how-to-read-redis-source-code.rst
[1]: /2018/05/22/redis-sds/
[2]: /2018/05/23/redis-list/
[3]: /2018/05/23/redis-dict/
[4]: /2018/05/23/redis-skiplist/