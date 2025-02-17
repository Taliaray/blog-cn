---
title: TiDB 在摩拜单车在线数据业务的应用和实践
author: ['丁宬杰','胡明']
date: 2017-12-25
summary: 部署 TiDB 近一年来，摩拜单车经历了用户数量近十倍、日骑行数据数十倍的增长，依靠其在线扩容的能力，完成了多次数据库扩容与服务器更换。
tags: ['互联网']
category: case
url: /case/user-case-mobike/
weight: 13
logo: /images/blog-cn/customers/mobike-logo.png
customer: 摩拜单车
customerCategory: 出行
---

> 作者：丁宬杰 / 胡明，Mobike 技术研发部基础平台中心

## 背景

摩拜单车于 2015 年 1 月成立，2016 年 4 月 22 日地球日当天正式推出智能共享单车服务，截至 2017 年 11 月中旬，已先后进入国内外超过 180 个城市，运营着超过 700 万辆摩拜单车，为全球超过 2 亿用户提供着智能出行服务，日订单量超过 3000 万，成为全球最大的智能共享单车运营平台和移动物联网平台。摩拜每天产生的骑行数据超过 30TB，在全球拥有最为全面的骑行大数据，飞速增长的业务使摩拜面临数据库扩展与运维的巨大挑战。

面对飞速增长的并发数与数据量，单机数据库终将因无法支撑业务压力而罢工。在摩拜正式上线以来，我们就在不断思考数据库扩展和运维的未来，近年来业内对数据库进行扩展的常见的方案是通过中间件把数据库表进行水平拆分，将表内数据按照规则拆分到多个物理数据库中。使用这样的中间件方案，在数据库扩容时需要先停下业务，再重构代码，之后进行数据迁移，对于摩拜这样与时间赛跑的创业公司来讲代价巨大，中间件方案对业务过强的侵入性，不支持跨分片的分布式事务，无法保证强一致性事务的特性都使我们望而却步。

摩拜单车于 2017 年初开始使用 TiDB，从最早的 RC3、RC4、PreGA、到现在的 1.0 正式版，一步步见证了 TiDB 的成熟和稳定。目前支撑着摩拜内部的实时分析和部分线上业务，同时正在规划迁移更多的线上业务至 TiDB。

目前，TiDB 在摩拜部署了数套集群，近百个节点，承载着数十 TB 的各类数据。

## TiDB 在摩拜的角色和主要应用场景

在摩拜，TiDB 是一个核心的数据交易与存储支撑平台，引入它的主要目的是用来解决海量数据的在线存储、大规模实时数据分析和处理。

在我们看来，TiDB 的好处主要有：

*   弹性扩容。具有 NoSQL 类似的扩容能力，在数据量和访问流量持续增长的情况下能够通过水平扩容提高系统的业务支撑能力，并且响应延迟稳定；
*   简单易用。兼容 MySQL 协议，基本上开箱即用，完全不用担心传统分库分表方案带来的心智负担和复杂的维护成本，而且用户界面友好，常规的技术技术人员都可以很高地进行维护和管理；
*   响应及时。因为和 PingCAP 团队有非常深入的合作关系，所以有任何问题都可以第一时间和 PingCAP 团队直接沟通交流，遇到问题都能很快的处理和解决。

下面介绍 TiDB 的应用场景：

### 场景一：开关锁日志成功率统计

开关锁成功率是摩拜业务监控的重点指标之一。

在每次开、关锁过程中，用户和锁信息会在关键业务节点产生海量日志，通过对线上日志的汇总分析，我们把用户的行为规整为人和车两个维度，通过分布式、持久化消息队列，导入并存放到 TiDB 里。在此过程中，通过对不同的实体添加不同的标签，我们就能方便地按照地域、应用版本、终端类型、用户、自行车等不同的维度，分别统计各个类别的开锁成功率。

按照我们的估计，这个业务一年的量在数百亿，所以使用单机的 MySQL 库需要频繁的进行归档，特别是遇到单机数据库瓶颈的情况下，扩容更是带来了非常大的挑战，这在我们有限的人力情况下，完全是个灾难。所以要支撑整个 Mobike 的后端数据库，我们必须要寻找简单易用的方案，极大地减少在单个业务上的人力成本开销。其次，根据我们之前使用分库分表的经验，对于这类需要频繁更新表结构进行 DDL 操作的业务，一旦数据量过大，很很容易出现数据库假死的情况，不仅影响服务的可用性，更严重的是很可能导致数据不一致的情况出现。最后，我们希望不管今后的业务量如何激增，业务需求如何变化，都可以保持业务逻辑可以很方便地升级支持。

在方案设计时，我们进行考察了 MySQL 分库分表的方案和 TiDB 方案的对比。

我们先估计了可能的情况：

*   新业务上线，在线变动肯定是经常发生的；
*   尽可能存长时间的数据，以备进行统计比较；
*   数据要支持经常性的关联查询，支撑运营组的临时需求；
*   要能支撑业务的快速增长或者一些特殊活动造成的临时流量。

考虑到这些情况，MySQL 分库分表的方案就出现了一些问题，首先频繁变动表结构就比较麻烦，而 TiDB 可以进行在线 DDL。数据生命期比较长，可以设计之初做一个比较大的集群，但是弹性就比较差，针对这个问题，TiDB 可以根据需要，弹性的增加或者减少节点，这样的灵活性是 MySQL 分库分表没有的。另外，数据要支持频繁的复杂关联查询，MySQL 分库分表方案完全没办法做到这一点，而这恰恰是 TiDB 的优势，通过以上的对比分析，我们选择了 TiDB 作为开关锁日志成功率统计项目的支撑数据库。

![](media/user-case-mobike/1.png)

目前，大致可以将到端到端的延时控制在分钟级，即，若有开锁成功率下降，监控端可即时感知，此外，还能通过后台按用户和车查询单次故障骑行事件，帮助运维人员快速定位出故障的具体位置。

### 场景二：实时数据分析

数据分析场景中，TiDB 可以从线上所有的 MySQL 实例中实时同步各类数据，通过 TiDB 周边工具 Syncer 导入到 TiDB 进行存储。

这个业务的需求很简单，我们线上有数十个 MySQL 集群，有的是分库分表的，有的是独立的实例。这些孤立的数据要进行归集以便供业务方进行数据分析。我们一开始计划把这些库同步到 Hive 中，考察了两种方式，一种是每日全量同步，这么做，对线上的库压力以及 Hive 的资源开销都会越来越大。另一种是增量同步，这种方式非常复杂，因为 HDFS 不支持 update，需要把每日增量的部分和之前的部分做 merge 计算，这种方法的优点是在数据量比较大的情况下，增量同步对比全量同步要更快、更节省空间，缺点是占用相当一部分 Hadoop 平台的计算资源，影响系统稳定性。

TiDB 本身有很多不错的工具，可以和 MySQL 的生态方便的连接到一起。这里我们主要使用了 TiDB 的 syncer 工具，这个工具可以方便的把 MySQL 实例或者 MySQL 分库分表的集群都同步到 TiDB 集群。因为 TiDB 本身可以 update，所以不存在 Hive 里的那些问题。同时有 TiSpark 项目，数据进入 TiDB 以后，可以直接通过 Spark 进行非常复杂的 OLAP 查询。有了这套系统，运营部门提出的一些复杂在线需求，都能够快速简洁的完成交付，这些在 Hadoop 平台上是无法提供这样的实时性的。

![](media/user-case-mobike/2.png)

目前，该集群拥有数十个节点，存储容量数十 T，受益于 TiDB 天然的高可用构架，该系统运行稳定，日后集群规模日益变大也仅需简单增加 x86 服务器即可扩展。后台开发、运维、业务方等都可以利用 TiDB 的数据聚合能力汇总数据，进行数据的汇总和分析。

### 场景三：实时在线 OLTP 业务

如前所述，对比传统的分库分表方案，TiDB 的灵活性和可扩展性在实时在线业务上优势更加明显。

根据我们的测试，TiDB 在数据量超过 5 千万时，对比 MySQL 优势较大，同时协议层高度兼容 MySQL，几乎不用修改业务代码就能直接使用，所以 TiDB 集群对于数据量大的实时在线业务非常适合。

目前，摩拜主要上线了两套在线 OLTP 业务，分别是摩豆信用分业务和摩豆商城业务。

#### 摩豆信用分业务

![](media/user-case-mobike/3.png)

摩拜单车信用分业务与用户骑行相关，用户扫码开锁时先行查询用户信用积分判断是否符合骑行条件，待骑行完成后，系统会根据用户行为进行信用分评估并进行修改。当单车无法骑行，上报故障核实有效后增加信用分、举报违停核实有效后降低信用分；但是如果不遵守使用规范，则会扣除相应的信用分；例如用户将自行车停在禁停区域内，系统就会扣除该用户的部分信用分作为惩罚，并存档该违停记录。当用户的信用分低于 80 分时，骑行费用将会大幅上升。

#### 摩豆商城业务（APP 中的摩拜成就馆）

![](media/user-case-mobike/4.png)

魔豆商城业务即摩拜成就馆，用户的每一次骑行结束后，系统会根据骑行信息赠送数量不等的省时币、环保币、健康币作为积分，通过积累这些积分可以在摩拜成就馆内兑换相应积分的实物礼品。

这些业务的共同特点：

*   7 * 24 * 365 在线，需要系统非常健壮，在任何状况下保证稳定运行；
*   数据不希望删除，希望能一直保存全量数据；
*   平时高峰期并发就非常大，搞活动的时候并发会有几倍的增长；
*   即便有业务变更，业务也不能暂停。

由于是典型 OLTP 场景，可选项并不多，而且数据量增长极快，这些数据库的数据在一年内轻松达到数百亿量级。这些场景在我们有了 TiDB 的使用经验以后，发现 TiDB 的所有特性都非常契合这种海量高并发的 OLTP 场景。TiDB 的容量/并发可随意扩展的特性不在赘述，支持在线 DDL 这个特性特别适合这些业务，有需要业务更改不会阻塞业务，这是我们业务快速迭代比较需要的特性。

目前，这两个在线 OLTP 集群拥有数十个节点，百亿级数据，上线以后非常稳定，PingCAP 客户支持团队也协助我们进行该集群的日常运维工作。

### 场景四：违章停车记录/开锁短信库等日志归集库

相对于传统的针对不同的业务分别部署 MySQL 集群的方案，TiDB 在可扩展性和在线跨库分析方面有较大优势。

在部署 TiDB 之前，摩拜面对新增的业务需要对其进行单独的规划和设计，并根据业务的数据量，增速以及并发量设计 MySQL 的分库分表方案，这些重复的预先设计工作在所难免。另一方面，不同业务之间往往是有关联的，当运营部门需要不同业务的汇总数据时，就变得异常麻烦，需要新建一个临时的数据汇总中心，例如新建一套 MySQL 分库分表的集群，或者临时向大数据组申请 Hive 的空间，这都让数据提供的工作变得麻烦。

有了 TiDB 以后，此类业务的开发和数据提供都变得非常简单，每次新需求下来，只需要按照新增数据量增加 TiKV 节点的数量即可，因为整个集群变得比较大，并发承载能力非常强，基本不需要考虑并发承载能力。特别的好处是，因为这些业务有相关性的业务，放在一个独立的数据库中，运营需要提供某几类某段时间的数据时就变得极为方便。

基于 TiSpark 项目，Spark 集群可以直接读取 TiDB 集群的数据，在一些运营需要实时数据提供的场景，不再需要按照原有的提供数据到大数据平台，设计 ETL 方案，运营再去大数据部门沟通运算逻辑。而是直接在 TiDB 现有数据的基础上，直接提出复杂的分析需求，设计 Spark 程序进行在线的直接分析即可。这样做，我们非常容易就可以实现一些实时状态的分析需求，让数据除了完成自己的工作，还能更好的辅助运营团队。

![](media/user-case-mobike/5.png)

## 使用过程中遇到的问题和优化

在说优化问题之前，先看 TiDB 的架构图，整个系统大致分为几个部分。

![](media/user-case-mobike/6.png)

其中：

*   PD 是整个集群的管理模块，负责：元信息管理、集群调度和分配全局递增非连续ID。
*   TiDB，是客户端接入层，负责 SQL 解析、执行计划优化，通过 PD 定位存储计算所需数据的 TiKV 地址。
*   TiKV，是数据的存储层，底层是基于 RocksDB 的 KV 引擎，并在其上分别封装 MVCC 和 Raft 协议，保证数据的安全、一致。
*   TiSpark，是 Spark 接入层，负责把 Spark 和 TiKV 连接到一起，在执行非常重的 OLAP 业务时可以利用到 Spark 集群的优势。

在使用过程中，遇到过不少问题，但是在我方和 PingCAP 技术团队的充分交流和协作下，都得到了比较完善的解决，下面挑选最为重要的资源隔离与优化展开。

TiKV 中数据存储的基本单位是 Region，每个 Region 都会按顺序存储一部分信息。当一个 Region 包含多个表的数据，或一台机器上有多个 Region 同时为热点数据时，就容易产生资源瓶颈。

PD 在设计之初考虑了这方面的问题（专门设计了 HotRegionBalance），但是，它的调度粒度是单个 Region，并且，整个调度基于这样的假设：即每个 Region 的资源消耗对等，不同 Region 之间没有关联，同时尽量保持 Region 均摊在所有 Store。

但当一个集群同时承载多个库，或一个库中包含多个表时，发生资源瓶颈的概率会明显提升。

针对这个问题，我们和 PingCAP 技术团队合作，对 TiDB 做了以下优化。

### 优化一：基于 Table 的分裂

这个修改的目的是解决小表数据的相互影响的问题。

当有新表数据插入某一 Region 时，TiKV 会根据当前 Region 的 Key Range 计算出 TableID，如果发现插入的 Key 不在这个 KeyRange 中，会对这个 Region 提前分裂，这就保证了每个 Region 只包含一个表的数据。

### 优化二：表级别的资源隔离

与此同时，我们在 PD 增加了 TableID 和 Namespace 之间的映射关系以及 NameSpace 和 TiKV Store 的映射关系，通过把上述关系持久化到 eEtcd 里，保证该映射关系的安全。

当数据插入时，可以在 TiDB 层面拿到 TableID，进而从 PD 找出目标 Region 所在的 TiKV，保证新插入的数据不会放到其他 TiKV。

另外，我们还与 PingCAP 团队共同开发实现了一个 NameSpace 调度器，把未规整的 Region 调度回它应在的 TiKV 里，进而在表级别保证数据不会相互干扰。

### 优化三：管理工具

最后的问题是管理 NameSpace 的问题。

好在 TiDB 在早期设计时保留了足够的灵活性，通过 TiDB 原有接口，我们只需要调用相关 API 即能通过表名拿到 TableID。

同时我们在 PD 的命令行管理台 pc-ctl 中增加了 HTTP 接口，管理确认 Table Name 和 TableID 之间的对应关系。

## 后记

部署 TiDB 近一年来，摩拜单车经历了用户数量近十倍，日骑行数据数十倍的增长，依靠 TiDB 在线扩容的能力，我们完成了多次数据库扩容与服务器更换，而且这些操作对业务是完全透明的，我们可以更专注于业务程序的开发与优化，而无须了解数据库的分片规则，对于快速成长的初创公司，这有着很强的借鉴意义。另外深度参与 TiDB 的开发并和开源社区紧密的互动，也使我们获得了很多有益的反馈，极大降低了代码维护成本。

未来，我们会联合 PingCAP 进一步丰富多集群的管理工具，进行更深入的研究和开发，持续提升 TiDB 的性能，将 TiDB 应用到更多的业务中。


