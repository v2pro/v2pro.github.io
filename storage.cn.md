---
layout: default
title: 文档数据库
---

基于 Mysql 构建的文档数据库

* TOC
{:toc}

数据库主要由两部分构成

* 主存储
* 各种视图

主存储需要以可靠的方式存储数据的原始权威拷贝。除此之外，它还需要按照业务规则来处理并发情况下的冲突（比如并发写入的时候，余额也不会变成负数）。

各种视图以不同的形式重新组织数据，使得查询可以变得更快。它也支持把多个数据源组合成一个统一的视图。

目标是构建一个文档数据库，把这两种角色的存储合并成一整套方案。它应该提供以下特性：

* 方案简单可依赖：核心的可靠性只依赖于 Mysql
* 无 Schema：读写的就是 JSON 
* 任何写入更新都原生支持幂等：开发者就不用自己去想其他办法来保障幂等了
* 业务规则可以保障数据的完整性：用起来就类似数据库的 unique 索引一样简单
* 对同一个 entity 的高并发写入：能够达到 10k tps 的级别，避免业务层去做热点账户这样的“恶心优化”
* 可靠的视图更新：对主存储的写入不能在同步的时候被遗漏，所有的改动都要最终一致性地同步到视图侧
* 低延迟的视图更新：如果开发者需要，可以在主存储更新的实现同步更新视图，实现大部分情况下的即时可见

# 总体设计

![aritecture](https://docs.google.com/drawings/d/e/2PACX-1vScYQzqv2-cINbIloWrm7G9A88cTWFcdtaUABKvBf8fiUyFPmRd5AblIhwceuv1L85_5uWmKylVwZ13/pub?w=888&h=870)

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

一次写入操作的过程

* 根据entity id进行hash，获得对应的partition，判断是不是当前服务器负责的。如果不是，则转发。
* docstore从kvstore里加载entity的最新的版本，也就是一条event。如果取得的event上有state，则直接读取到了。
* 如果event上没有state，只有delta，则向前搜索之前的event，直到取到一条event包含state为止
* 把state，叠加其后的所有delta得到一个完整的doc对象
* 根据event type和command type取得command handler，传入doc对象和command request进行执行
* command handler直接对doc进行修改，并最终返回command response
* doc自身记录了所有改动，要么直接序列化doc为state，要么把doc上的改动序列化成delta。
* 生成一条event，并插入到kvstore。如果没有冲突，则继续
* 如果event产生了冲突，说明发生了未符合预期的并发写入。刷新集群拓扑，从头开始整个过程。

整个数据库对外提供的主动可调用的api是两个

* exec：写操作，保证一致性。传入event type, command type, entity id等信息
* query：读操作，不保证读取到是最新的。传入event type, command type, entity id。query操作不能对doc进行修改，否则会报错。query 可以用于读取state中的部分值，或者做一些视图计算逻辑。如果需要保证读取的数据是最新的，使用exec进行读操作，这样每次读操作也会产生一条event。query主要的用途是提高读取的性能，因为可以使用非master来进行响应。默认支持的command type是"get"，读一个entity的整个state。

数据库同时提供视图同步能力，由外部注册handler。

TODO：定义视图同步的spi

# 如何保障并发写入下的正确性

正确性是首先需要保证的。因为整个handler执行都是在内存中进行的，handler执行的过程中，底层的状态就由可能被其他的线程或者进程提前修改了。要回答这个问题，首先要了解为什么需要这样的数据库。

* 传统的并发保障是由数据库保证的，让业务把操作下推到数据库内部来进行，比如写成SQL。这种做法的问题是SQL的表达力是有限的，无法做很复杂的业务判断。
* 如果不想把整个操作下推到数据库内完成，另外一个常见的选择是在业务层做大部分的计算。然后在数据库层面实现一个悲观锁（读取时就上锁），或者乐观锁（写入时检查base版本是否变化）。但是在并发比较高的情况下，由于业务的application server之间缺少协调（分区选主），乐观锁冲突会严重限制写入的tps。

所以我们的目标是最大化开发效率，同时保证性能。

* 开发效率：数据模型可以是任意的JSON对象，而不用建模成关系模型。业务代码可以直接用javascript来写，而不用翻译成sql。相比其他的cqrs解决方案，不需要业务代码里出现event/command等字眼，写逻辑的时候就是对普通的对象的直接读写。doc跟踪自身的改动，自动生成event。
* 保证性能：采用乐观锁的模型，但是进程之间进行主从的协调，减少锁冲突的概率。同时利用内存中的doc对象缓存，避免读取数据库和反序列化的成本。正常情况下，一个写操作的成本是 JSON 序列化加上一次 mysql insert。虽然 JSON 确实是一个性能堪忧的格式，但是序列化的速度相对于反序列化来说要快得多。

然后我们来讨论正确性

* 最根本得正确性保证是kvstore使用 `entity_id+entity_version` 做为主键。这样可以保证一个entity的一个版本只会有一个插入成功。
* 读取event的时候，得知base的version。command执行完之后，写入的时候用base version+1。如果同时有另外一个线程或者进程已经写入成功了，base version+1就一定会产生主键冲突的错误。
* 一些内存中的缓存（比如entity id对应一个内存中的doc对象，避免每次写入都重新读取）的更新无法保证和kvstore的更新是原子发生的。所以在一个进程内，对同一个entity的操作只能在一个线程内串行发生。这样就可以保证进程内不会产生并发的冲突问题，从而在kvstore写入之后，再进行内存缓存的刷新即可。

关于kvstore的主键rowkey的选择

* entity id是定长的全局唯一的字符串
* entity version是一个uint64
* 需要保证rowkey对于同一个entity id来说，version增长等价于rowkey的增长。也就是我们可以通过正向的rowkey scan获得更新的entity版本。同时在scan的过程中必须读到的是同一个entity的数据，不会插入其他entity的event。同时scan到最末尾可以知道这个entity没有更多的版本了。
* 需要支持迅速获得某个entity的最高版本。如果通过正向scan来做，这个操作的代价就会随着entity的版本数的增长而增长，这样是不可以接受的。需要kvstore支持逆向的scan，从而可以从尾部开始读取。
* rowkey自身用字母表的顺序排序，把entity id和version是拼成一个字符串来处理的
* 为了方便debug时的查看数据，要求rowkey都是printable的ascii，不能直接用big endian来表示uint64
* 在以上的约束下，rowkey的设计是 `[entity id]_[entity version big endian hex]`

比如 entity id 是 asdxcv，entity version是1024，那么对应的rowkey就是 `asdxcv_0000000000000400`

再比如 entity id 是 jixcfe, entity version 是971，那么对应的rowkey就是 `jixcfe_00000000000003cb`

实际的kvstore是用mysql来模拟的。因为在大部分公司的运维水平下，mysql是唯一可以做可靠主存储的物理介质。分表为997个，以支持水平扩展。建表的 SQL 是

```sql
CREATE TABLE event_912 (
  `rowkey`           CHAR(29)     NOT NULL, --12+16+1
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

997个mysql表实际上就是把一个完整的队列，切分成997段进行了存储。event表除了支持kv读写之外，还可以用于扫描一个entity的一段时间内的改动，进行数据的复制同步。起到类似kafka队列，或者mysql binlog的作用。

# 如何解决写入放大问题

TODO

# 如果提供RPC幂等性

TODO

# 如何保障视图的更新是可靠的

TODO

# 如何保障视图的更新是低延迟的

TODO

# 结语

TODO
