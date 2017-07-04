---
layout: default
title: 拼SQL
---

拼SQL

* TOC
{:toc}

Go 裸写 SQL 的体验非常差。ORM 不是答案。很多时候用 ORM 需要先知道 SQL 预期是怎样的，然后再来猜测怎样用 ORM 才能拼出想要的 SQL 来。所以不如直接用类似 html 模板的方案来解决拼 SQL 的效率问题吧。

* 分库分表：需要支持表名里带区号，需要用字符串拼接出表名
* IN查询：默认的变量替换对于 IN 查询支持很不方便。有多少个成员就要写多少个?
* 命名参数：默认的sql库不支持命名参数。sqlx支持，所以sqlx的写法直接移植过来了。
* 快速拼SQL：更新多个字段，或者插入新记录的时候需要手拼SQL。sqlxx 直接帮你拼了
* 轻量级：直接用 database/sql 的 driver，连 database/sql 这一层都没有走。所有的对象绑定，类型转换，map创建的开销都被省去了。
