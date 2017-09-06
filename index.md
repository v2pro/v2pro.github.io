---
layout: default
title: 总体架构
---

# V2PRO 

探索专业化的业务软件开发之道

![v2pro architecture](https://docs.google.com/drawings/d/e/2PACX-1vR4gK59DB8KuNP_5CmrZJrGkjkeaYukA1T-bKRSVqDfe3HuYZ2XqW7UG6EFD4dRCpUEHXkG1Pdimal7/pub?w=1084&h=422)

# 微服务模板化

从多样化的业务需求里提取出以下四大类微服务，以及对应的技术挑战。预期以框架化的方式给一个通用解决方案。

* Orchestrator：实现业务流程。主要的痛点：分布式事务的最终一致性。碎片化代码的可维护性。
* Domain Model：业务数据的主存储。主要的痛点：数据一致性按业务规则保护。幂等性。热点数据写入性能。
* Strategy：实现策略逻辑。主要痛点：和主存储的低延迟同步。和离线训练流程的集成。
* Configurator：实现运营配置功能。主要的痛点：自动生成的界面和数据模型。配置推送。规则引擎。

# 效率提升工具

用创新实现开发效率提升。

* 基于流量录制和回放的单元测试解决方案
* 基于流量录制的监控解决方案
* Go Generics，以及以此为基础的 validation/codec/functional programming 等通用utility
* 拼 SQL 的库
