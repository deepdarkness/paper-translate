# Amazon Aurora 

Design Considerations for High Throughput Cloud-Native Relational Databases
## 设计高吞吐量云原生关系数据库的思考


Alexandre Verbitski, Anurag Gupta, Debanjan Saha, Murali Brahmadesam, Kamal Gupta, Raman Mittal, Sailesh Krishnamurthy, Sandor Maurice, Tengiz Kharatishvili, Xiaofeng Bao

Amazon Web Services

## 摘要

> Amazon Aurora is a relational database service for OLTP workloads offered as part of Amazon Web Services (AWS). In this paper, we describe the architecture of Aurora and the design considerations leading to that architecture. We believe the central constraint in high throughput data processing has moved from compute and storage to the network. Aurora brings a novel architecture to the relational database to address this constraint, most notably by pushing redo processing to a multi-tenant scale- out storage service, purpose-built for Aurora. We describe how doing so not only reduces network traffic, but also allows for fast crash recovery, failovers to replicas without loss of data, and fault-tolerant, self-healing storage. We then describe how Aurora achieves consensus on durable state across numerous storage nodes using an efficient asynchronous scheme, avoiding expensive and chatty recovery protocols. Finally, having operated Aurora as a production service for over 18 months, we share lessons we have learned from our customers on what modern cloud applications expect from their database tier.

Amazon Aurora 是一种用于 OLTP 工作负载的关系数据库服务，作为 Amazon Web Services (AWS) 的一部分提供。在本文中，我们描述了 Aurora 的架构以及导致该架构的设计注意事项。我们认为高吞吐量数据处理的核心约束已经从计算和存储转移到网络。 Aurora 为关系数据库带来了一种新颖的架构来解决这一限制，最显着的方法是将重做处理推送到专为 Aurora 构建的多租户横向扩展存储服务。我们描述了这样做不仅可以减少网络流量，还可以实现快速崩溃恢复、故障转移到副本而不丢失数据，以及容错、自我修复的存储。然后，我们描述了 Aurora 如何使用高效的异步方案在众多存储节点上就持久状态达成共识，避免昂贵且繁琐的恢复协议。最后，在将 Aurora 作为生产服务运营超过 18 个月之后，我们分享了我们从客户那里学到的关于现代云应用程序对其数据库层的期望的经验教训。

## Keywords

> Databases; Distributed Systems; Log Processing; Quorum Models; Replication; Recovery; Performance; OLTP

数据库； 分布式系统； 日志处理； 法定人数模型； 复制； 恢复; 表现; OLTP

## 1. INTRODUCTION

> IT workloads are increasingly moving to public cloud providers. Significant reasons for this industry-wide transition include the ability to provision capacity on a flexible on-demand basis and to pay for this capacity using an operational expense as opposed to capital expense model. Many IT workloads require a relational OLTP database; providing equivalent or superior capabilities to on-premise databases is critical to support this secular transition.

IT 工作负载越来越多地转移到公共云提供商。这种全行业转型的重要原因包括能够在灵活的按需基础上提供容量，并使用运营费用而不是资本费用模型来支付此容量。许多 IT 工作负载需要关系型 OLTP 数据库；为本地数据库提供等效或卓越的功能对于支持这种长期过渡至关重要。

> In modern distributed cloud services, resilience and scalability are increasingly achieved by decoupling compute from storage [10][24][36][38][39] and by replicating storage across multiple nodes. Doing so lets us handle operations such as replacing misbehaving or unreachable hosts, adding replicas, failing over from a writer to a replica, scaling the size of a database instance up or down, etc.

在现代分布式云服务中，通过将计算与存储分离 [10][24][36][38][39] 和跨多个节点复制存储，越来越多地实现弹性和可扩展性。这样做可以让我们处理诸如替换行为不当或无法访问的主机、添加副本、从写入器故障转移到副本、向上或向下扩展数据库实例的大小等操作。

> The I/O bottleneck faced by traditional database systems changes in this environment. Since I/Os can be spread across many nodes and many disks in a multi-tenant fleet, the individual disks and nodes are no longer hot. Instead, the bottleneck moves to the network between the database tier requesting I/Os and the storage tier that performs these I/Os. Beyond the basic bottlenecks of packets per second (PPS) and bandwidth, there is amplification of traffic since a performant database will issue writes out to the storage fleet in parallel. The performance of the outlier storage node, disk or network path can dominate response time.

传统数据库系统在这种环境下变化所面临的I/O瓶颈。由于 I/O 可以分布在多租户队列中的多个节点和多个磁盘上，因此单个磁盘和节点不再是热的。相反，瓶颈转移到请求 I/O 的数据库层和执行这些 I/O 的存储层之间的网络。除了每秒数据包 (PPS) 和带宽的基本瓶颈之外，还有流量放大，因为高性能数据库将并行写入存储队列。异常存储节点、磁盘或网络路径的性能可以决定响应时间。

> Although most operations in a database can overlap with each other, there are several situations that require synchronous operations. These result in stalls and context switches. One such situation is a disk read due to a miss in the database buffer cache. A reading thread cannot continue until its read completes. A cache miss may also incur the extra penalty of evicting and flushing a dirty cache page to accommodate the new page. Background processing such as checkpointing and dirty page writing can reduce the occurrence of this penalty, but can also cause stalls, context switches and resource contention.

尽管数据库中的大多数操作可以相互重叠，但有几种情况需要同步操作。这些会导致停顿和上下文切换。一种这样的情况是由于数据库缓冲区缓存中的未命中而导致的磁盘读取。读取线程在读取完成之前无法继续。高速缓存未命中还可能导致驱逐和刷新脏高速缓存页面以容纳新页面的额外惩罚。检查点和脏页写入等后台处理可以减少这种惩罚的发生，但也会导致停顿、上下文切换和资源争用。

> Transaction commits are another source of interference; a stall in committing one transaction can inhibit others from progressing. Handling commits with multi-phase synchronization protocols such as 2-phase commit (2PC) [3][4][5] is challenging in a cloud- scale distributed system. These protocols are intolerant of failure and high-scale distributed systems have a continual “background noise” of hard and soft failures. They are also high latency, as high scale systems are distributed across multiple data centers.

事务提交是另一个干扰源；提交一个事务的停滞可能会阻止其他事务的进展。在云规模的分布式系统中，使用多阶段同步协议（例如两阶段提交（2PC）[3][4][5]）处理提交具有挑战性。这些协议不能容忍故障，并且大规模分布式系统具有持续的硬故障和软故障的“背景噪声”。它们也是高延迟的，因为大规模系统分布在多个数据中心。

> In this paper, we describe Amazon Aurora, a new database service that addresses the above issues by more aggressively leveraging the redo log across a highly-distributed cloud environment. We use a novel service-oriented architecture (see Figure 1) with a multi-tenant scale-out storage service that abstracts a virtualized segmented redo log and is loosely coupled to a fleet of database instances. Although each instance still includes most of the components of a traditional kernel (query processor, transactions, locking, buffer cache, access methods and undo management) several functions (redo logging, durable storage, crash recovery, and backup/restore) are off-loaded to the storage service.

在本文中，我们介绍了 Amazon Aurora，这是一种新的数据库服务，它通过在高度分布式的云环境中更积极地利用重做日志来解决上述问题。我们使用一种新颖的面向服务的架构（参见图 1）和一个多租户横向扩展存储服务，该服务抽象了一个虚拟化的分段重做日志，并与一组数据库实例松散耦合。尽管每个实例仍包含传统内核的大部分组件（查询处理器、事务、锁定、缓冲区缓存、访问方法和撤消管理），但有几个功能（重做日志记录、持久存储、崩溃恢复和备份/恢复）已关闭-加载到存储服务。

> Our architecture has three significant advantages over traditional approaches. First, by building storage as an independent fault- tolerant and self-healing service across multiple data-centers, we protect the database from performance variance and transient or permanent failures at either the networking or storage tiers. We observe that a failure in durability can be modeled as a long- lasting availability event, and an availability event can be modeled as a long-lasting performance variation – a well-designed system can treat each of these uniformly [42]. Second, by only writing redo log records to storage, we are able to reduce network IOPS by an order of magnitude. Once we removed this bottleneck, we were able to aggressively optimize numerous other points of contention, obtaining significant throughput improvements over the base MySQL code base from which we started. Third, we move some of the most complex and critical functions (backup and redo recovery) from one-time expensive operations in the database engine to continuous asynchronous operations amortized across a large distributed fleet. This yields near-instant crash recovery without checkpointing as well as inexpensive backups that do not interfere with foreground processing.

我们的架构与传统方法相比具有三个显着优势。首先，通过将存储构建为跨多个数据中心的独立容错和自我修复服务，我们可以保护数据库免受网络或存储层的性能差异和暂时或永久故障的影响。我们观察到，持久性故障可以建模为持久的可用性事件，而可用性事件可以建模为持久的性能变化——一个设计良好的系统可以统一处理每一个 [42]。其次，通过只将重做日志记录写入存储，我们能够将网络 IOPS 降低一个数量级。一旦我们消除了这个瓶颈，我们就能够积极优化许多其他争用点，与我们开始的基本 MySQL 代码库相比，获得了显着的吞吐量改进。第三，我们将一些最复杂和关键的功能（备份和重做恢复）从数据库引擎中的一次性昂贵操作转移到在大型分布式队列中分摊的连续异步操作。这产生了近乎即时的崩溃恢复，无需检查点，以及不干扰前台处理的廉价备份。

