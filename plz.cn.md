---
layout: default
title: plz
---

PLZ：通过 API 和 SPI 定义非功能性需求的边界

* TOC
{:toc}

![边界](https://user-images.githubusercontent.com/40541/35490660-366d23a0-04dc-11e8-871e-2025da349c3b.png)

边界列表

* [Service（服务接口）：服务器 <-> 功能性需求代码 <-> 客户端](/plz/service.cn.html)
* [Instrumentation（埋点）：功能性需求代码 <-> 日志/指标/事件，功能性需求代码 <-> expvar/状态页面](/plz/instrumentation.cn.html)
* [Config（配置读取）：功能性需求代码 <-> 配置/AB测试](/plz/config.cn.html)
* Plug-and-Play（集群管理）：功能性需求代码 <-> 进程和服务的集群管理
