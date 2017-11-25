---
layout: default
title: 创新的主存储方案
---

创新的主存储方案

* TOC
{:toc}

# 创新的目标

在分析了[现有的解决方案的不足](http://v2p.ro/storage/why.cn.html)之后。我们来看一种创新的存储方案。它的目标是

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
  
# 总体设计

为了支撑以上目标，实现上的几个核心技术选型：

* 状态都在MySQL，自身是无状态的模块
* 用Javascript写的存储过程托管业务逻辑，“数据微服务”可以快速复制，只需要部署一个dcostore的实例，挂上自己的javascript逻辑就可以了。
* 文档数据库的数据模型，按entity id进行hash分区
* 主存储存储的是event（变更日志），而不是entity（最终的状态）。event 以类似kafka队列的方式保存，按序号严格递增。
* 不额外用zookeeper之类的东西进行选主，把变更日志无冲突写入就算抢到了主
* 利用保存的历史event做为并发问题的解决方案，同时代替了MySQL binlog做为数据同步的数据源

![components](https://go.gliffy.com/go/share/image/srp8w54cn0n75xuemfik.png?utm_medium=live-embed&utm_source=custom)

总体分为两层：

* kvstore：一个由mysql实现的kv存储。api设计上来说支持普通的点读，和正向反向的scan。对于主键冲突的情况，数据库可以明确阻止写入，并返回对应的错误。
* docstore：实现文档数据库的业务逻辑。自身没有任何持久化的状态，只在内存中做一些缓存，用于提高速度。数据的可靠性和冲突检测完全依赖kvstore。

分为以下概念：

* entity：一个有id的实体。文档数据读写的就是entity。
* entity id：全局唯一的id，格式为字符串，由调用方生成
* entity version：实体的版本号，单调递增，从1开始
* entity type：对entity的分类，不同类型的entity支持的command不同。
* command type：读写entity的操作名
* command handler：由 entity type 和 command type 可以映射到一个具体的操作的实现
* command id：一次读写操作的全局唯一id，由调用方生成。相同的command id代表这个操作被重试了。数据库要能够对同样的command id进行幂等的处理，返回完全相同的response。
* command request：读写操作的正文，格式为 JSON，可以有任意的字段
* state：entity的状态，格式为 JSON，可以有任意的字段
* delta：当前版本的entity状态相对上一个版本的增量，格式为 Delta JSON。
* doc：文档对象，entity当前状态在内存中的表示。是command操作的目标。
* command response：读写操作返回给调用方的结果。
* event：一次读写操作产生的副作用的记录，做为一条记录插入到kvstore里。包括两部分，一部分是command response。另外一部分是对entity的修改，体现为一个新的state，或者是一个delta。

# 如何保障并发写入下的正确性

![process](https://go.gliffy.com/go/share/image/sudta8m6xkde7e8g9t3o.png?utm_medium=live-embed&utm_source=custom)

一次写入操作的过程，注意这里略去了和command id幂等相关的处理。

1. 客户端通过http请求docstore的endpoint，要求对某个entity执行一个command。
1. 根据entity id进行hash，获得对应的partition，以及这个partition的master是谁。这个信息并不一定正确，写入的正确性不依赖于master信息的完全准确。
1. 判断是不是当前服务器负责的。如果不是，则转发给目前的master的endpoint去处理。
1. endpoint通过go channel发给对应partition的command processor去处理。通过go channel，保证了同一个partition的entity command始终发送给同一个processor串行处理。
1. processor拿到了entity id之后，需要加载entity的状态。processor在自己的内存里有一个entity的缓存，先查这里。同时获得processor任何当前partition的last event id。
1. 如果自己缓存里没有，则需要从kvstore里查找entity id对应的最后一次改动的event id，这个东西叫event lookup，后面会说这个event lookup的实时性问题。
1. 用event id从kvstore里加载entity的最新的版本，也就是一条event。如果取得的event上有state，则直接读取到了。如果没有，就往前追溯，直到找到为止。然后把entity状态重建出来，这个东西叫doc。
1. 根据event type和command type取得javascript定义的command handler，传入doc对象和command request进行执行。command handler直接对doc进行修改，并最终返回command response。
1. doc自身记录了所有改动，直接序列化doc的状态变化为delta。如果已经连续数个版本只记录了delta，为了降低状态重建成本，需要把doc整体序列化为state。生成一条event，并插入到kvstore的下一个版本里，也就是event id+1。如果没有冲突，则继续。如果event产生了冲突，说明发生了未符合预期的并发写入。刷新集群拓扑，从头开始整个过程。
1. 写入了event kv store之后，改动已经可靠落盘了。然后要刷新一下processor内存里的状态缓存，包括entity缓存，以及当前partition的版本号，last event id。
1. 把新建的event写入event notification channel，通知event processor去做entity lookup的同步，和外部view的同步。然后command processor就完成了这个command的处理了，接着去处理下一个event。注意如果写入event notification channel失败，并不阻塞，直接丢弃。
1. event processor被唤醒，去处理新进来的event。它要验证这个event和之前处理的上一个event的event id是不是连续的。
1. 如果不连续，则说明中间发生了阻塞导致的丢弃。这个时候需要把丢掉的event从kv store里读取出来。
1. 根据event，更新kvstore上记录的entity lookup。然后更新entity lookup的version，也就是last event id。
1. 起新的goroutine，触发javascript定义的event handler，然后把外部的view更新了。

entity lookup其实就是用entity id维度建立了event历史的二级索引。所以必然会有一个更新不及时，导致不一致的问题。

entity lookup也是和event的kvstore一样，分partition的。每个entity lookup partition都有一个版本号，记录了最后更新到的last event id。读取entity lookup之前先读一下这个版本号就可以知道，这个partition的数据是不是及时更新了。如果有滞后，则报错，说明这个时候因为无法加载entity从而无法处理请求。一般来说最近刚写入的entity是缓存在内存里的，就不会从kvstore上去读。所以entity lookup更新稍微滞后一点是没有关系的。

另外一个可能照成entity lookup不一致的情况是docstore刚完成了主的切换。前一个主还在刷新自己的状态到entity lookup，后面切换上来的主已经开始往entity lookup了。这个时候就可能照成一个entity id对应的event id刚被写入就被覆盖了。为了应对这种情况。entity lookup在写入的时候需要保证值是递增的，如果新值小于现值就丢弃。

# MySQL表结构

实际的kvstore是用mysql来模拟的。因为在大部分公司的运维水平下，mysql是唯一可以做可靠主存储的物理介质。

分表为997个，以支持水平扩展。建表的 SQL 是

```sql
CREATE TABLE event_912 (
  `event_id`         BIGINT       NOT NULL,
  `base_event_id`    BIGINT       NOT NULL,
  `entity_id`        CHAR(12)     NOT NULL,
  `entity_version`   BIGINT       NOT NULL,
  `command_id`       CHAR(12)     NOT NULL,
  `command_name`     VARCHAR(256) NOT NULL,
  `command_request`  TEXT         NOT NULL,
  `command_response` TEXT         NOT NULL,
  `state`            TEXT         NULL,
  `delta`            TEXT         NULL,
  `committed_at`     DATETIME     NOT NULL       DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`rowkey`)
);
```

997个mysql表实际上就是997个独立的队列。event表除了支持kv读写之外，还起到类似kafka队列的作用，用于支持数据同步。

# 如何解决写入放大问题

存储JSON的一个大问题是JSON表示本身就很大。而且无论每次改动有多么少，都需要重新存储一份完整的JSON会数倍数十倍的放大写入量。这个就完全取决于JSON本身的大小了。如果业务上一个entity本身非常巨大，就会使得写入的效率很低。

为了解决尺寸的问题，可以尝试用短的JSON KEY，或者干脆用类似protobuf/thrift的int表示field的模式，或者用LZ4压缩。但是这些都无法根本上改变“写入放大的现象”。要根除问题，还是要实现一次改动了多少，就保存多少，而不是每次都全量写入整个JSON。为了达到这个目的，需要解决以下问题：

* delta的格式如何定义
* 如何知道一次操作产生了什么样的delta
* 如何应用delta，进行增量更新

首先来看第一个问题，delta的格式

```json
{"p":{"leaf":{"u":{"hello":"world"}}}}
```

我们定义如下的delta格式：

* `u`: updated，代表object或者array的key被完全更新
* `p`：patched，代表object或者array的key被部分更新，具体更新了什么，由同样的delta json格式嵌套表达
* `r`：removed，代表object或者array的key被删除

对于这样的 JSON 

```json
{"leaf":{"origKey":"origValue"}}
```

应用前面的 delta 得到的结果是

```json
{"leaf":{"hello":"world","origKey":"origValue"}}
```

接下来的问题是，如何产生前面这样的 delta 格式呢？如果要让开发者手工维护这个 delta，肯定会被人骂死，因为代码写起来太别扭了。做一个修改不能直接去改变量，而是先产生一个所谓的 event，然后再 apply 这个 event（大部分的 cqrs 方案就是这么要求开发者的），是非常反人类的。

解决的办法是让开发者直接修改对象，而对象自己来跟踪自己属性的变化，这个和前端领域的 mvvm 的方案是一样的。我们定义这样的 struct 来代表 `map[string]interface{}`

```go
type DObject struct {
	data    map[string]interface{} // 实际的数据
	updated map[string]interface{} // 更新的key
	patched map[string]interface{} // 部分更新的key
	removed map[string]interface{} // 删除了的key
}
```

然后在 set 的时候去捕捉改动

```go
func (obj *DObject) Set(key interface{}, value interface{}) {
	obj.data[key.(string)] = value
	if obj.updated == nil {
		obj.updated = map[string]interface{}{}
	}
	obj.updated[key.(string)] = value
}
```

但是部分更新的key怎么知道呢？总不能每次改动的时候都通知父对象去标记吧。这样还要维护父子关系。解决办法是在最终序列化delta的时候去询问每个key是不是dirty了，如果dirty，则父对象把这个key设置到部分更新了的key的map里。

```go
func (obj *DObject) calcPatched() map[string]interface{} {
	if obj.patched != nil {
		return obj.patched
	}
	obj.patched = map[string]interface{}{}
	for k, v := range obj.data {
		switch typedValue := v.(type) {
		case *DObject:
			if typedValue.isDirty() {
				obj.patched[k] = typedValue
			}
		}
	}
	return obj.patched
}
```

解决了跟踪改动的问题之后，又带来了严重的开发体验问题。如果让开发者不能写 `obj["key"]="value"`，而一定要 `obj.(*DObject).Set("key", "value")`，这样的体验是非常糟糕的。而 go 又不向 javascript 一样可以加各种各样的钩子来解决这个问题。解决办法就是不用 go 来写 command handler，而是用 javascript 来写 command handler。通过把 javascript 里的 `obj["key"]="value"` 翻译成对应的 go 源码 `AsObj(obj).Set("key", "value")`，来解决开发体验的问题。

下一个要解决的问题是有了这样的delta之后，如何增量更新呢？如果需要从一个 `[]byte` 叠加到另外一个 `[]byte`，这样会带来非常多的开销。解决办法是delta直接流式地读出来，每读一点就应用一点到内存里的doc上（一个对象图）。如果要从头实现这样一个 JSON 读取过程还是很浩大的工作量的。我们通过 json-iterator 的可扩展性，复用了现有的json解析器。把 delta 的增量应用问题，等价成了一个定制化的 JSON 反序列化过程。能够这么做的根本原因在于，json-iterator 反序列化时接收的对象的指针不一定是一个全空的对象。如果传入的指针是一个有值的对象，这个反序列过程，本身其实就是在做增量更新。

示意代码如下 

```go
func (decoder *objectDeltaDecoder) Decode(ptr unsafe.Pointer, iter *jsoniter.Iterator) {
	obj := (*DObject)(ptr)
	iter.ReadMapCB(func(iter *jsoniter.Iterator, field string) bool {
		switch field {
		case "u":
			iter.ReadMapCB(func(iter *jsoniter.Iterator, field string) bool {
				input := iter.SkipAndReturnBytes()
				subIter := Json.BorrowIterator(input) // switch from DeltaJson to Json
				defer Json.ReturnIterator(subIter)
				obj.data[field] = subIter.Read()
				return true
			})
		case "p":
			iter.ReadMapCB(func(iter *jsoniter.Iterator, field string) bool {
				fieldValue := obj.data[field]
				iter.ReadVal(&fieldValue)
				obj.data[field] = fieldValue
				return true
			})
		default:
			iter.Skip()
		}
		return true
	})
}
```

这个patch应用的过程并不是特别的快。但是常态下docstore也不应该从kvstore里去读取entity，而是缓存在自己的内存中。因为docstore本身进行了hash和主从的集群协调，每个docstore只需要负责自己对应的entity，这个缓存的效率还是非常高的。所以JSON反序列的成本，delta的增量应用成本不会是性能的瓶颈。

# 如果提供RPC幂等性

如果同一个操作被重试了会怎样？比如给用户转账，我们显然不希望每重试一次，就转账一次。所有网络上暴露的RPC操作都应该是幂等的，否则对端就无法进行重试。也就是对于相同的command id，无论执行多少次，返回的 response 应该是一样的，同时对 entity 的状态修改只做一次。

因为我们存储了entity上所有的event，event上有command id。所以这个幂等性的问题可以直接在存储的层面支持，仅仅需要能够支持从command id，查询到对应的event。也就是把幂等性问题变成了一个索引的问题。但是在前面的设计中，为了支持历史event的批量同步，我们选择了用event id做为rowkey。在kvstore不支持二级索引的前提下，我们需要额外维护一个从command id（全局唯一的随机字符串）到event id（单调递增的uint64）的映射关系。这个反向的索引和前面从entity id到event id是一样的。所以这里就不再重复赘述了。

TODO：补充一张图

# 如何保障视图的更新是可靠的

主存储保存的是历史的event。视图是根据这些历史上发生的事情，构建出自己的视角。这个过程分为三个问题

* 进度跟踪问题：知道之前同步到哪里了，这次要同步多少。对历史的解读肯定是严格有序来做。
* 批量同步问题：一次性解读一批历史上的event，产生出对视图的更新。批量可以提高同步的效率。
* 幂等问题：同步可能出问题，就会导致重试。在重试发生的时候，对视图的更新应该保持幂等。比如说统计总订单量。我们不能每次同步的时候简单的加加。要排除掉已经计算过的event。

TODO：具体的"面同步"例子
TODO：具体的"点同步"例子

# 如何保障视图的更新是低延迟的

对于点同步的视图，每个entity的主存储和视图的记录是有一一对应的关系的。比如主存储是订单，entity id是订单id。视图是用户历史订单，用户id是索引字段，而且是分库分表的hash字段。这个例子里，视图的没一行记录都对应了一个entity。这样视图的行级别是有明确的版本号的概念的。也就是说可以做到行级别的幂等更新，这使得我们可以实现低延迟的优化。当订单被更新的之后，直接在主存储发起对历史订单视图的更新。这个操作和普通的pull的同步操作不同，它是push的。在视图的行上记录对应entity的版本。如果更新的时候发现视图的版本已经高于要同步的版本了，这丢弃这次更新，避免因为并发更新导致的写脏了的问题。

对于面同步的视图，比如统计每小时的订单量。这种主动push更新的方式是不行的。因为视图的一行记录"每小时订单量"和主存储的entity版本没有对应关系，从而无法实现行级别的幂等更新。只能用严格一致消费历史event的方式进行同步。

# 视图结构的修改

因为主存储现在是schemaless的了，也就没有了因为要保持数据结构一致带来的data migration问题（mysql就需要改写旧的记录都给加上一个空字段）。而主存储的数据本身并没有data migration的需求，因为过去发生的事情，如果没有记录下来的字段，现在是不可能增补的。

但是视图仍然会有data migration的需求。因为同样的一批数据，可以有无数种视图的可能性。随着业务的发展，总是会有超出之前设计的视图需求的。这个时候我们就需要根据主存储记录的历史，重新解读一遍，建立新的视图。可能是给原有的视图表加个字段，也可能是完全建立另外一个视角的视图。

如果是全新的视图，这个和前面的视图同步其实是一个问题。只是的同步的offset从0开始罢了。如果是在现有的视图上添加了字段，这个问题可以等价为一个视图分成两个视图，一个原有的字段，一个是新的字段。只是这两个视图底层共享了一个存储罢了。那么我们可以独立地进行这两个视图的同步。等offset追齐了之后再做一个视图的切换。

# 结语

这个方案的面子是文档数据库，里子是CQRS，用起来就像串行读写的内存中的对象一样简单。


