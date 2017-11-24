---
layout: default
title: 创新的存储方案
---

创新的存储方案

* TOC
{:toc}

在分析了现有的解决方案的不足之后。我们来看一种创新的存储方案。它的目标是

* 支持快速地业务创新
  * 快速地添加或者修改主存储：避免数据迁移带来的停机。
  * 快速地添加或者修改视图：同样地历史数据可以随时用不同的视角重新解读。
  * 快速地支持复杂的业务逻辑：Javascript表达的存储过程，让数据的业务规则一致性不再受限于数据库提供的查询语言的能力。
  * 不是所谓的CQRS：现有的CQRS方案需要业务层做很多的额外努力。业务逻辑开发应该是简单直接的修改对象，而不是创建所谓的Event
  * 避免重复封装类似的数据服务：数据库不再提供通用的查询和修改能力，所有的修改都经过Javascript定义的存储过程，保证了业务逻辑的收口。
* 并发问题的打包解决方案
  * 复杂数据结构的强一致性：不依赖复杂的多行事务，分布式事务，MVCC就能保持一个复杂数据结构的强一致性
  * 业务开发不需要考虑并发冲突：内建乐观锁，业务逻辑开发只需要考虑串行执行的情况。通过协调集群拓扑，避免乐观锁冲突引起的性能下降。
  * 直接解决RPC操作的幂等性问题：同一个操作执行多次，数据库自动去重，并严格保证幂等
* 良好的集成能力
  * 不尝试一个数据库解决所有问题：假设用户需要用不同的异构存储去支持不同的数据视图。异构数据的同步能力直接在数据库层面就支持。
  * 保证最终一致性：能够保证数据同步到任意异构数据库之后的不重不丢。不依赖对端数据库提供特殊支持。
  * 极低延迟的同步：通过推拉结合的方式保证在正常情况下，数据同步的速度要高于通过kafka等消息队列复制的速度。降低业务层为了支持最终一致性引起的容错成本。