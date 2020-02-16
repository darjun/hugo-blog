---
layout:  	post
title:		"Go Web 编程之 实战社交平台（一）"
subtitle: 	"入门 Go Web 编程"
date:		2020-12-15T16:31:00
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - Go Web 编程
URL: "2020/02/15/goweb/social1"
categories: [
	"Go"
]
---

## 概述

至此，我们已经学习了 Go Web 编程的基础知识。为了方便大家回顾，我将前面几篇文章放在下面：

* [Go Web 编程之 Hello World](https://darjun.github.io/2019/11/25/goweb/hello-world/)
* [Go Web 编程之 程序结构](https://darjun.github.io/2019/12/05/goweb/structure/)
* [Go Web 编程之 请求](https://darjun.github.io/2019/12/09/goweb/request/)
* [Go Web 编程之 响应](https://darjun.github.io/2019/12/18/goweb/response/)
* [Go Web 编程之 模板（一）](https://darjun.github.io/2019/12/31/goweb/template1/)
* [Go Web 编程之 模板（二）](https://darjun.github.io/2019/12/31/goweb/template2/)
* [Go Web 编程之 静态文件](https://darjun.github.io/2020/01/13/goweb/fileserver/)
* [Go Web 编程之 数据库](https://darjun.github.io/2020/01/16/goweb/db/)

接下来，我们来完成一个实战项目，编写一个简单的社交平台功能。我们的社交平台有下面几个基本功能：

* 用户注册和登录；
* 用户可以发布帖子，其他用户可以回复帖子；
* 用户可以关注其他用户，登录时主页上显示自己和其他用户的帖子信息；
* 用户可以编辑自己的个人信息和帖子内容。

今天我们先完成数据库表的设计。

## 数据库设计

根据上面我们列出的功能需求，可知系统中存在下面几个实体：

* 用户；
* 帖子；
* 评论。

我们还需要有一张表存储用户之间的关注关系。数据库表的设计推荐两个工具，[SQL Designer](https://ondras.zarovi.cz/sql/demo/)和[dbdiagram](https://dbdiagram.io/d/5e44b95c9e76504e0ef1622f)。前者比较轻量级，后者功能较为丰富，还可以根据设计导出 SQL 语句。我是使用后者进行设计：

![](/img/in-post/goweb/social1.png#center)

dbdiagram 的使用比较简单，在左侧用它规定的语法定义表，以及表与表之间的外链关系，添加索引等。右侧会可视化显示表结构和关系。

用户表`users`字段如下：

* `id`：自增id，主键；
* `username`：用户名，唯一；
* `email`：邮件，唯一；
* `register_at`：注册时间；
* `password`：密码。

为`username`和`email`设置唯一索引，用户名和邮件都不能重复。这里注意一点，**我这里表中直接存储了密码。一般在生产环境中，直接存储密码是非常不安全的，容易受到攻击**。这里只是为了简化处理。

帖子表`posts`字段如下：

* `id`：自增id，主键；
* `content`：内容；
* `created_at`：创建时间；
* `updated_at`：更新时间；
* `user_id`：作者id。

这里`user_id`外链到用户表`users`。

评论表`comments`字段如下：

* `id`：自增id，主键；
* `content`：内容；
* `created_at`：评论日期；
* `post_id`：帖子id；
* `user_id`：用户id。

这里`post_id`外链到帖子表`posts`，`user_id`外链到用户表`users`。

最后一个表`follows`表示用户之间的关注关系，因为每个用户可以关注多个其他用户，也可以被多个其他用户关注，所以不能放在用户表`users`中。

然后导出`social.sql`：

```sql
CREATE TABLE `users` (
  `id` int PRIMARY KEY AUTO_INCREMENT,
  `username` varchar(64),
  `email` varchar(128),
  `register_at` datetime,
  `password` varchar(128)
);

CREATE TABLE `posts` (
  `id` int PRIMARY KEY,
  `content` varchar(1024),
  `created_at` datetime,
  `updated_at` datetime,
  `user_id` int
);

CREATE TABLE `comments` (
  `id` int PRIMARY KEY AUTO_INCREMENT,
  `content` varchar(256),
  `created_at` datetime,
  `post_id` int,
  `user_id` int
);

CREATE TABLE `follows` (
  `follower_id` int,
  `followed_id` int,
  PRIMARY KEY (`follower_id`, `followed_id`)
);

ALTER TABLE `posts` ADD FOREIGN KEY (`user_id`) REFERENCES `users` (`id`);

ALTER TABLE `comments` ADD FOREIGN KEY (`post_id`) REFERENCES `posts` (`id`);

ALTER TABLE `comments` ADD FOREIGN KEY (`user_id`) REFERENCES `users` (`id`);

ALTER TABLE `follows` ADD FOREIGN KEY (`follower_id`) REFERENCES `users` (`id`);

ALTER TABLE `follows` ADD FOREIGN KEY (`followed_id`) REFERENCES `users` (`id`);

CREATE UNIQUE INDEX `users_index_0` ON `users` (`username`);

CREATE UNIQUE INDEX `users_index_1` ON `users` (`email`);

CREATE INDEX `posts_index_2` ON `posts` (`created_at`);

CREATE INDEX `comments_index_3` ON `comments` (`created_at`);
```

在启动命令行，切换到`social.sql`所在目录，连接mysql，创建数据库`social`，切换到该数据库，执行`source social.sql`，所有表就创建好了：

![](/img/in-post/goweb/social2.png#center)

## 总结

## 参考

1. SQL Designer, [https://ondras.zarovi.cz/sql/demo/](https://ondras.zarovi.cz/sql/demo/)
2. dbdiagram, [https://dbdiagram.io/d/5e44b95c9e76504e0ef1622f](https://dbdiagram.io/d/5e44b95c9e76504e0ef1622f)

## 我

[我的博客](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~

![](/img/wxgzh8.jpg#center)