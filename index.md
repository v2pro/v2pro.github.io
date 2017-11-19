---
layout: default
title: 总体架构
---

# V2PRO 

![v2pro](https://docs.google.com/drawings/d/e/2PACX-1vRT5h9AVqantCAi01hdSZkJ3u_YSrtUZKOox2jj_YQEnDdvr4-DtC0xB-v4CSpsrMZsGz3xNthuk3vX/pub?w=507&h=296)

开发，测试，运维，分布式业务全生命周期解决方案

功能特点：

* 基于流量录制和回放的测试解决方案
* 全链路分布式tracing
* 无需 sdk 接入服务注册和发现
* 灰度发布代码，配置和开关
* 代码和配置推送
* 逐级聚合的 metrics，让 monitor everything 成为可能
* 本地开发用的调试界面

构成模块：

* plz：标准化的上传下达接口，实现灰度能力。每个 Go 应用都应该使用的标准库
* koala：流量录制和回放，分布式tracing，服务注册和发现
* quokka：集群管理，配置和代码的下发接口
* quoll：海量数据上行通道，trace的存储和检索，metrics的逐级聚合
