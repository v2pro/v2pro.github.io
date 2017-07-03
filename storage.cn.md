---
layout: default
title: 存储平台
---

基于 Mysql 构建的文档数据库

* TOC
{:toc}

![architecture](https://docs.google.com/drawings/d/1_UjPesdIb8zaIfdkcs-qRwkZVQYxzLvuqRkXuK6uMuI/pub?w=820&amp;h=585)

数据库主要由两部分构成

* 主存储
* 各种视图

主存储需要以可靠的方式存储数据的原始权威拷贝。除此之外，它还需要按照业务规则来处理并发情况下的冲突（比如并发写入的时候，余额也不会变成负数）。

各种视图以不同的形式重新组织数据，使得查询可以变得更快。它也支持把多个数据源组合成一个统一的视图。

目标是构建一个文档数据库，把这两种角色的存储合并成一整套方案。它应该提供以下特性：

* 方案简单可依赖：核心只依赖于 Mysql
* 无 Schema：主存储基于 JSON 做为输入
* 任何写入更新都原生支持幂等：开发者就不用自己去想其他办法来保障幂等了
* 业务规则可以保障数据的完整性：用起来就类似数据库的 unique 索引一样简单
* 对同一个 entity 的高并发写入：能够达到 10k tps 的级别，避免业务层去做热点账户这样的“恶心优化”
* 可靠的视图更新：对主存储的写入不能在同步的时候被遗漏，所有的改动都要最终一致性地同步到视图侧
* 低延迟的视图更新：如果开发者需要，可以在主存储更新的实现同步更新视图，实现大部分情况下的即时可见
* 长期稳定性：冷数据沉降这样的事情应该自动完成，不依赖额外的运维操作

# 如何保障主存储的正确性

主存储的表结构类似这样（假设 entity 的名称叫 account，不同的 entity 创建独立的表）

```sql
CREATE TABLE `v2pro`.`account` (
  `event_id`     BIGINT       NOT NULL       AUTO_INCREMENT,
  `entity_id`    CHAR(20)     NOT NULL,
  `version`      BIGINT       NOT NULL,
  `command_id`   VARCHAR(256) NOT NULL,
  `command_name` VARCHAR(256) NOT NULL,
  `request`      JSON         NULL,
  `response`     JSON         NOT NULL,
  `state`        JSON         NOT NULL,
  `committed_at` DATETIME     NOT NULL       DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`event_id`),
  UNIQUE KEY `unique_version` (`entity_id`, `version`),
  UNIQUE KEY `unique_command` (`entity_id`, `command_id`)
);
```

更新一个 entity 的过程就是

* 加载旧的 state，并记得当时的版本
* 处理命令，从旧的 state 计算出新的 state
* 保存新的 state，并且在同一个SQL中要求版本号 + 1

如果多个人同时更新同一个entity，unique_version 这个数据库约束可以阻止第二个用户保存成功。这样通过一个通用的乐观锁可以保护状态不被并发修改，而命令的处理逻辑可以任意复杂，只需要考虑串行处理的场景。

另外一个关键的要求是支持幂等。只要 command id 相同，无论处理被处理多少遍，返回都应该是一样的。这个可以通过 unique_command 这个数据库约束来实现。如果同一个命令被处理了两边，我们可以总是返回第一次的结果。

只要 mysql 的唯一性约束不掉链子，我们就可以实现正确性的保证。这种实现很简单

* 我们不需要担心 etcd 做流量 sharding 的时候各种故障场景
* 我们也不依赖应用服务器的内存状态不是脏的
* 我们也不再应用服务器本地磁盘保存任何状态
* 我们选择把应用服务器仍然是无状态的，只依赖 Mysql 单表事务来保证并发下的正确性

所有的一切，都仅仅基于两个唯一性索引。

# 如何保障主存储的性能

这样的使用模式下，Mysql 是无法对单个 entity 的并发写入达到 10k tps的。所以我们需要做更多。本质上，我们需要让应用服务器“有状态起来”，从而可以实现：

* 请求在应用服务器上做排队，而不是直接把压力都怼到数据库上
* 在应用服务器上做批量，从而可以用更大块地写入来获得更好的吞吐量

然而，这里的“有状态”仅仅只是做为一种性能优化手段。在业务没有高tps写入的需求的情况，甚至这种性能优化都可以先不做。我们不依赖于应用服务器的状态来保证逻辑的正确性。这是和其他的 cqrs 方案的主要区别。因为我们不依赖于应用服务器的状态，我们在sharding搞乱了情况下，或者服务器故障的情况下都很容易保障数据本身的完整性不被破坏。即便数据没有被正确分配到正确的服务器上，也仅仅是性能受到影响。

所以，首先我们把应用服务器做sharding，让对同一个entity的更新请求总是打到同一个应用服务器上。然后，我们把请求排队起来批量处理。内部的队列就是一个golang的channel。批量可以用 channel 的“select”来实现。

```golang
func (worker *worker) fetchCommands() []*command {
	done := false
	commands := []*command{}
	for !done {
		select {
		case cmd := <-worker.commandQ:
			commands = append(commands, cmd)
		default:
			done = true
		}
		if len(commands) > 1000 {
			break
		}
	}
	return commands
}
```

sharding 也不用搞得太复杂

* 有条件的，可以用etcd来选leader。如果简单情况下，静态配置拓扑也可以
* 每个服务器和选举出的 leader 保持心跳
* 在心跳的同时，leader把数据的分布情况回包回去。这个分布情况在内存中做缓存
* 请求来的时候，会被重定向到正确的shard
* 如果服务器收到一个被重定向过的请求，无论这个请求该不该它来处理，它都应该处理，而不是再次重定向

关于组集群方面，关键的设计决策：
 
* 我们只用etcd 选举leader，而不用 etcd 记录数据分布。leader丢失不会让业务暂停，只是分布情况不再得到更新而已。最坏情况也仅仅是锁争抢导致的性能下降。
* 请求只会被重定向一次，避免被循环重定向
* 每一个应用服务器既可以是“gateway”也可以是“command handler”

# Reliable view update

We do not use mysql binlog to synchronize view. Instead the entity table is partitioned like kafka partition.
event_id is a auto increment big int works like kafka offset. 
In essence, every entity table partition is a event queue.
The view updater will scan the event table and write update to the view side. There two things to watch for

* How to ensure idempotence on view update, if the synchronization is retried?
* How to remember the last updated offset?

The updated offset can be recorded as a mysql table. Every entity table partition will have a highest sync offset.
We batch process the update, and only update the offset record every second.

Because the offset is not recorded in the view side with transaction, and committed on every write,
the update to the view will be retried. We record the entity version on the view side. 
When we update the view, we compare the event (which contains entity version) with the applied version.
If one event has already been applied, we skip it.

Having a separate view updater run in the background every 100ms, 
we can ensure we do not lose the update on the view side.
However, the table polling will result in high latency.

# Low latency view update

We allow synchronous view update directly from command handler, for latency sensitive view. 
When a command is committed, the view update is done immediately afterwards by same worker goroutine.
This way we can cut down the hand over time to nearly zero, there is no queue can beat this.
However, there things to care about 

* update view directly from command handler means there will be concurrent updates.
* update view might fail.

The concurrent update will be handled by the optimistic lock on the view side, same as view updater retry.
Update view might fail. We might lose update if only rely on direct update from command handler.
Luckily, we have separate view updater working in "pull" mode to ensure integrity.
It is just like a patch for "push" mode view update.

# Closing thought

Both the main storage and views are built upon optimistic lock. 
And sharding & queueing is a optimization to reduce the lock contention.
