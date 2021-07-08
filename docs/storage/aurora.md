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

数据库; 分布式系统; 日志处理; Quorum 模型; 复本; 恢复; 性能; OLTP

## 1. INTRODUCTION

> IT workloads are increasingly moving to public cloud providers. Significant reasons for this industry-wide transition include the ability to provision capacity on a flexible on-demand basis and to pay for this capacity using an operational expense as opposed to capital expense model. Many IT workloads require a relational OLTP database; providing equivalent or superior capabilities to on-premise databases is critical to support this secular transition.

越来越多的IT工作负载转移到公有云提供商。这种全行业转型的重要原因包括能够在灵活的按需基础上提供容量，并使用运营费用而不是资产费用模型来支付此容量。许多 IT 工作负载需要关系型 OLTP 数据库；为本地数据库提供等效或卓越的功能对于支持这种长期过渡至关重要。

> In modern distributed cloud services, resilience and scalability are increasingly achieved by decoupling compute from storage [10][24][36][38][39] and by replicating storage across multiple nodes. Doing so lets us handle operations such as replacing misbehaving or unreachable hosts, adding replicas, failing over from a writer to a replica, scaling the size of a database instance up or down, etc.

在现代分布式云服务中，通过将计算与存储分离 [10][24][36][38][39] 和跨多个节点复制存储实现弹性和可扩展性。这样做可以让我们处理诸如替换行为不当或无法访问的主机、添加副本、从写入器故障转移到副本、向上或向下扩展数据库实例的大小等操作。

> The I/O bottleneck faced by traditional database systems changes in this environment. Since I/Os can be spread across many nodes and many disks in a multi-tenant fleet, the individual disks and nodes are no longer hot. Instead, the bottleneck moves to the network between the database tier requesting I/Os and the storage tier that performs these I/Os. Beyond the basic bottlenecks of packets per second (PPS) and bandwidth, there is amplification of traffic since a performant database will issue writes out to the storage fleet in parallel. The performance of the outlier storage node, disk or network path can dominate response time.

传统数据库系统所面临的 I/O 瓶颈在这种环境下发生了变化。由于 I/Os 可以分布在多租户中的多个节点和多个磁盘上，因此单个磁盘和节点不再是热点。相反，瓶颈转移到请求 I/O 的数据库层和执行这些 I/O 的存储层之间的网络。除了每秒数据包 (PPS) 和带宽的基本瓶颈之外，还有流量放大，因为高性能数据库将并行写入存储队列。异常存储节点、磁盘或网络路径的性能可以决定响应时间。

> Although most operations in a database can overlap with each other, there are several situations that require synchronous operations. These result in stalls and context switches. One such situation is a disk read due to a miss in the database buffer cache. A reading thread cannot continue until its read completes. A cache miss may also incur the extra penalty of evicting and flushing a dirty cache page to accommodate the new page. Background processing such as checkpointing and dirty page writing can reduce the occurrence of this penalty, but can also cause stalls, context switches and resource contention.

尽管数据库中的大多数操作可以相互重叠，但有几种情况需要同步操作。这些会导致停顿和上下文切换。一种这样的情况是由于数据库缓冲区缓存中的未命中而导致的磁盘读取。读取线程在读取完成之前无法继续。高速缓存未命中还可能导致驱逐和刷新脏高速缓存页面以容纳新页面的额外开销。检查点和脏页写入等后台处理可以减少这种开销的发生，但也会导致停顿、上下文切换和资源争用。

> Transaction commits are another source of interference; a stall in committing one transaction can inhibit others from progressing. Handling commits with multi-phase synchronization protocols such as 2-phase commit (2PC) [3][4][5] is challenging in a cloud- scale distributed system. These protocols are intolerant of failure and high-scale distributed systems have a continual “background noise” of hard and soft failures. They are also high latency, as high scale systems are distributed across multiple data centers.

事务提交是另一个影响因素；提交一个事务的停滞可能会阻止其他事务的进展。在云规模的分布式系统中，使用多阶段同步协议（例如两阶段提交（2PC）[3][4][5]）处理提交具有挑战性。这些协议不能容忍故障，并且大规模分布式系统具有持续的硬故障和软故障的“背景噪声”。它们也是高延迟的，因为大规模系统分布在多个数据中心。

> In this paper, we describe Amazon Aurora, a new database service that addresses the above issues by more aggressively leveraging the redo log across a highly-distributed cloud environment. We use a novel service-oriented architecture (see Figure 1) with a multi-tenant scale-out storage service that abstracts a virtualized segmented redo log and is loosely coupled to a fleet of database instances. Although each instance still includes most of the components of a traditional kernel (query processor, transactions, locking, buffer cache, access methods and undo management) several functions (redo logging, durable storage, crash recovery, and backup/restore) are off-loaded to the storage service.

在本文中，我们介绍了 Amazon Aurora，这是一种新的数据库服务，它通过在高度分布式的云环境中更积极地利用重做日志来解决上述问题。我们使用一种新颖的面向服务的架构（参见图 1）和一个多租户横向扩展存储服务，该服务抽象了一个虚拟化的分段重做日志，并与一组数据库实例松散耦合。尽管每个实例仍包含传统内核的大部分组件（查询处理器、事务、锁定、缓冲区缓存、访问方法和撤消管理），但有几个功能（重做日志记录、持久存储、崩溃恢复和备份/恢复）已关闭-加载到存储服务。

> Our architecture has three significant advantages over traditional approaches. First, by building storage as an independent fault- tolerant and self-healing service across multiple data-centers, we protect the database from performance variance and transient or permanent failures at either the networking or storage tiers. We observe that a failure in durability can be modeled as a long- lasting availability event, and an availability event can be modeled as a long-lasting performance variation – a well-designed system can treat each of these uniformly [42]. Second, by only writing redo log records to storage, we are able to reduce network IOPS by an order of magnitude. Once we removed this bottleneck, we were able to aggressively optimize numerous other points of contention, obtaining significant throughput improvements over the base MySQL code base from which we started. Third, we move some of the most complex and critical functions (backup and redo recovery) from one-time expensive operations in the database engine to continuous asynchronous operations amortized across a large distributed fleet. This yields near-instant crash recovery without checkpointing as well as inexpensive backups that do not interfere with foreground processing.

我们的架构与传统方法相比具有三个显着优势。首先，通过将存储构建为跨多个数据中心的独立容错和自我修复服务，我们可以保护数据库免受网络或存储层的性能差异和暂时或永久故障的影响。我们观察到，持久性故障可以建模为持久的可用性事件，而可用性事件可以建模为持久的性能变化——一个设计良好的系统可以统一处理每一个 [42]。其次，通过只将重做日志记录写入存储，我们能够将网络 IOPS 降低一个数量级。一旦我们消除了这个瓶颈，我们就能够积极优化许多其他争用点，与我们开始的基本 MySQL 代码库相比，获得了显着的吞吐量改进。第三，我们将一些最复杂和关键的功能（备份和重做恢复）从数据库引擎中的一次性昂贵操作转移到在大型分布式队列中分摊的连续异步操作。这产生了近乎即时的崩溃恢复，无需检查点，以及不干扰前台处理的廉价备份。

> In this paper, we describe three contributions:
> 1. How to reason about durability at cloud scale and how to design quorum systems that are resilient to correlated failures. (Section 2).
> 2. How to leverage smart storage by offloading the lower quarter of a traditional database to this tier. (Section 3).
> 3. How to eliminate multi-phase synchronization, crash recovery and checkpointing in distributed storage (Section 4).

在本文中，我们讨论这三种措施：
1. 如何推理云规模的持久性以及如何设计对相关故障具有弹性的仲裁系统（第 2 节）。
2. 如何通过将传统数据库的下四分之一卸载到这一层来利用智能存储（第 3 节）。
3. 如何消除分布式存储中的多阶段同步、崩溃恢复和检查点（第 4 节）。 

> We then show how we bring these three ideas together to design the overall architecture of Aurora in Section 5, followed by a review of our performance results in Section 6 and the lessons we have learned in Section 7. Finally, we briefly survey related work in Section 8 and present concluding remarks in Section 9.

然后，我们将在第 5 节展示如何将这三个想法结合起来设计 Aurora 的整体架构，然后在第 6 节回顾我们的性能结果以及我们在第 7 节中学到的经验教训。最后，我们简要调查了相关工作 第 8 节和第 9 节的总结性评论。

## 2 DURABILITY AT SCALE

> If a database system does nothing else, it must satisfy the contract that data, once written, can be read. Not all systems do. In this section, we discuss the rationale behind our quorum model, why we segment storage, and how the two, in combination, provide not only durability, availability and reduction of jitter, but also help us solve the operational issues of managing a storage fleet at scale.

如果数据库系统什么都不做，它必须满足数据一旦写入就可以读取的约定。 并非所有系统都这样。 在本节中，我们将讨论仲裁模型背后的基本原理、为什么要对存储进行分段，以及两者结合如何不仅提供持久性、可用性和减少抖动，还可以帮助我们解决管理存储队列的操作问题 规模。

### 2.1 ReplicationandCorrelatedFailures

> Instance lifetime does not correlate well with storage lifetime. Instances fail. Customers shut them down. They resize them up and down based on load. For these reasons, it helps to decouple the storage tier from the compute tier.

实例寿命与存储寿命没有很好的相关性。实例失败。客户关闭了它们。他们根据负载上下调整它们的大小。由于这些原因，它有助于将存储层与计算层分离。

> Once you do so, those storage nodes and disks can also fail. They therefore must be replicated in some form to provide resiliency to failure. In a large-scale cloud environment, there is a continuous low level background noise of node, disk and network path failures. Each failure can have a different duration and a different blast radius. For example, one can have a transient lack of network availability to a node, temporary downtime on a reboot, or a permanent failure of a disk, a node, a rack, a leaf or a spine network switch, or even a data center.

一旦这样做，这些存储节点和磁盘也可能出现故障。因此，它们必须以某种形式复制以提供故障恢复能力。在大规模云环境中，节点、磁盘和网络路径故障持续存在低水平的背景噪声。每次失败可能有不同的持续时间和不同的爆炸半径。例如，一个节点可能暂时缺乏网络可用性，重新启动时临时停机，或者磁盘、节点、机架、叶或主干网络交换机甚至数据中心的永久性故障。

> One approach to tolerate failures in a replicated system is to use a quorum-based voting protocol as described in [6]. If each of the V copies of a replicated data item is assigned a vote, a read or write operation must respectively obtain a read quorum of Vr votes or a write quorum of Vw votes. To achieve consistency, the quorums must obey two rules. First, each read must be aware of the most recent write, formulated as Vr + Vw > V. This rule ensures the set of nodes used for a read intersects with the set of nodes used for a write and the read quorum contains at least one location with the newest version. Second, each write must be aware of the most recent write to avoid conflicting writes, formulated as Vw > V/2.

在复制系统中容忍故障的一种方法是使用 [6] 中描述的基于仲裁的投票协议。如果每个复制数据项的 V 个副本都被分配了一个投票，则读取或写入操作必须分别获得 Vr 投票的读仲裁或 Vw 投票的写入仲裁。为了实现一致性，法定人数必须遵守两个规则。首先，每次读取都必须知道最近的写入，公式为 Vr + Vw > V。此规则确保用于读取的节点集与用于写入的节点集相交，并且读取仲裁包含至少一个最新版本的位置。其次，每次写入必须知道最近的写入以避免写入冲突，公式为 Vw > V/2。

> A common approach to tolerate the loss of a single node is to replicate data to (V = 3) nodes and rely on a write quorum of 2/3 (Vw=2)andareadquorumof2/3(Vr=2).

容忍单个节点丢失的常见方法是将数据复制到 (V = 3) 个节点，并依赖于 2/3 (Vw=2) 和 readquorumof2/3(Vr=2) 的写入仲裁。

> We believe 2/3 quorums are inadequate. To understand why, let’s first understand the concept of an Availability Zone (AZ) in AWS. An AZ is a subset of a Region that is connected to other AZs in the region through low latency links but is isolated for most faults, including power, networking, software deployments, flooding, etc. Distributing data replicas across AZs ensures that typical failure modalities at scale only impact one data replica. This implies that one can simply place each of the three replicas in a different AZ, and be tolerant to large-scale events in addition to the smaller individual failures.

我们认为 2/3 的法定人数不足。要了解原因，让我们首先了解 AWS 中可用区 (AZ) 的概念。 AZ 是 Region 的子集，通过低延迟链路连接到该区域中的其他 AZ，但对大多数故障（包括电源、网络、软件部署、泛洪等）进行隔离。大规模只影响一个数据副本。这意味着可以简单地将三个副本中的每一个放置在不同的可用区中，并且除了较小的单个故障之外还可以容忍大规模事件。

> However, in a large storage fleet, the background noise of failures implies that, at any given moment in time, some subset of disks or nodes may have failed and are being repaired. These failures may be spread independently across nodes in each of AZ A, B and C. However, the failure of AZ C, due to a fire, roof failure, flood, etc, will break quorum for any of the replicas that concurrently have failures in AZ A or AZ B. At that point, in a 2/3 read quorum model, we will have lost two copies and will be unable to determine if the third is up to date. In other words, while the individual failures of replicas in each of the AZs are uncorrelated, the failure of an AZ is a correlated failure of all disks and nodes in that AZ. Quorums need to tolerate an AZ failure as well as concurrently occuring background noise failures.

然而，在大型存储群中，故障的背景噪音意味着，在任何给定的时间，某些磁盘或节点子集可能已经发生故障并正在修复中。这些故障可能独立地分布在每个可用区 A、B 和 C 中的节点上。但是，由于火灾、屋顶故障、洪水等原因，可用区 C 的故障将破坏同时发生故障的任何副本的法定人数在 AZ A 或 AZ B 中。此时，在 2/3 读取仲裁模型中，我们将丢失两个副本，并且无法确定第三个是否是最新的。换句话说，虽然每个可用区中副本的单个故障是不相关的，但一个可用区的故障是该可用区中所有磁盘和节点的相关故障。仲裁需要容忍 AZ 故障以及同时发生的背景噪声故障。

> In Aurora, we have chosen a design point of tolerating (a) losing an entire AZ and one additional node (AZ+1) without losing data, and (b) losing an entire AZ without impacting the ability to write data. We achieve this by replicating each data item 6 ways across 3 AZs with 2 copies of each item in each AZ. We use a quorum model with 6 votes (V = 6), a write quorum of 4/6 (Vw = 4), and a read quorum of 3/6 (Vr = 3). With such a model, we can (a) lose a single AZ and one additional node (a failure of 3 nodes) without losing read availability, and (b) lose any two nodes, including a single AZ failure and maintain write availability. Ensuring read quorum enables us to rebuild write quorum by adding additional replica copies.

在 Aurora 中，我们选择的设计点允许 (a) 丢失整个 AZ 和一个附加节点 (AZ+1) 而不丢失数据，以及 (b) 丢失整个 AZ 而不影响写入数据的能力。我们通过跨 3 个 AZ 以 6 种方式复制每个数据项来实现这一点，每个 AZ 中的每个项目有 2 个副本。我们使用具有 6 票 (V = 6)、写入仲裁为 4/6 (Vw = 4) 和读取仲裁为 3/6 (Vr = 3) 的仲裁模型。使用这样的模型，我们可以 (a) 丢失单个可用区和一个额外的节点（3 个节点发生故障）而不会丢失读取可用性，以及 (b) 丢失任意两个节点，包括单个可用区故障并保持写入可用性。确保读取仲裁使我们能够通过添加额外的副本来重建写入仲裁。

### 2.2 Segmented Storage

> Let’s consider the question of whether AZ+1 provides sufficient durability. To provide sufficient durability in this model, one must ensure the probability of a double fault on uncorrelated failures (Mean Time to Failure – MTTF) is sufficiently low over the time it takes to repair one of these failures (Mean Time to Repair – MTTR). If the probability of a double fault is sufficiently high, we may see these on an AZ failure, breaking quorum. It is difficult, past a point, to reduce the probability of MTTF on independent failures. We instead focus on reducing MTTR to shrink the window of vulnerability to a double fault. We do so by partitioning the database volume into small fixed size segments, currently 10GB in size. These are each replicated 6 ways into Protection Groups (PGs) so that each PG consists of six 10GB segments, organized across three AZs, with two segments in each AZ. A storage volume is a concatenated set of PGs, physically implemented using a large fleet of storage nodes that are provisioned as virtual hosts with attached SSDs using Amazon Elastic Compute Cloud (EC2). The PGs that constitute a volume are allocated as the volume grows. We currently support volumes that can grow up to 64 TB on an unreplicated basis.

让我们考虑 AZ+1 是否提供足够的耐用性的问题。为了在此模型中提供足够的耐用性，必须确保在修复这些故障之一（平均修复时间 – MTTR）所需的时间内，不相关故障（平均故障时间 – MTTF）出现双重故障的概率足够低.如果双重故障的概率足够高，我们可能会在 AZ 故障上看到这些，从而破坏法定人数。很难在某一点上降低 MTTF 对独立故障的概率。相反，我们专注于减少 MTTR 以缩小双故障的漏洞窗口。为此，我们将数据库卷分区为固定大小的小段，目前大小为 10GB。这些每个都以 6 种方式复制到保护组 (PG) 中，以便每个 PG 由六个 10GB 段组成，跨三个 AZ 组织，每个 AZ 中有两个段。存储卷是一组串联的 PG，使用大量存储节点在物理上实现，这些节点使用 Amazon Elastic Compute Cloud (EC2) 配置为带有附加 SSD 的虚拟主机。构成卷的 PG 随着卷的增长而分配。我们目前支持的卷可以在非复制的基础上增长到 64 TB。

> Segments are now our unit of independent background noise failure and repair. We monitor and automatically repair faults as part of our service. A 10GB segment can be repaired in 10 seconds on a 10Gbps network link. We would need to see two such failures in the same 10 second window plus a failure of an AZ not containing either of these two independent failures to lose quorum. At our observed failure rates, that’s sufficiently unlikely, even for the number of databases we manage for our customers.

段现在是我们独立的背景噪音故障和修复单元。作为我们服务的一部分，我们监控并自动修复故障。在 10Gbps 的网络链路上，10GB 的网段可以在 10 秒内修复。我们需要在同一个 10 秒窗口中看到两个这样的故障，加上一个不包含这两个独立故障中的任何一个的可用区故障，才能失去仲裁。根据我们观察到的故障率，即使对于我们为客户管理的数据库数量，这也是不太可能的。

### 2.3 Operational Advantages of Resilience

> Once one has designed a system that is naturally resilient to long failures, it is naturally also resilient to shorter ones. A storage system that can handle the long-term loss of an AZ can also handle a brief outage due to a power event or bad software deployment requiring rollback. One that can handle a multi- second loss of availability of a member of a quorum can handle a brief period of network congestion or load on a storage node.

一旦设计了一个对长期故障自然有弹性的系统，它自然也对较短的故障有弹性。可以处理 AZ 长期丢失的存储系统还可以处理由于电源事件或需要回滚的错误软件部署而导致的短暂中断。可以处理仲裁成员的可用性在几秒钟内丢失的情况可以处理短时间的网络拥塞或存储节点上的负载。

> Since our system has a high tolerance to failures, we can leverage this for maintenance operations that cause segment unavailability. For example, heat management is straightforward. We can mark one of the segments on a hot disk or node as bad, and the quorum will be quickly repaired by migration to some other colder node in the fleet. OS and security patching is a brief unavailability event for that storage node as it is being patched. Even software upgrades to our storage fleet are managed this way. We execute them one AZ at a time and ensure no more than one member of a PG is being patched simultaneously. This allows us to use agile methodologies and rapid deployments in our storage service.

由于我们的系统对故障具有很高的容忍度，我们可以将其用于导致段不可用的维护操作。例如，热量管理很简单。我们可以将热磁盘或节点上的一个段标记为坏的，并且仲裁将通过迁移到队列中的某个其他较冷节点来快速修复。操作系统和安全修补是该存储节点在修补时的短暂不可用事件。甚至我们存储设备的软件升级也是通过这种方式进行管理的。我们一次执行一个 AZ，并确保同时修补 PG 的成员不超过一个。这使我们能够在我们的存储服务中使用敏捷方法和快速部署。

## 3 THE LOG IS THE DATABASE

> In this section, we explain why using a traditional database on a segmented replicated storage system as described in Section 2 imposes an untenable performance burden in terms of network IOs and synchronous stalls. We then explain our approach where we offload log processing to the storage service and experimentally demonstrate how our approach can dramatically reduce network IOs. Finally, we describe various techniques we use in the storage service to minimize synchronous stalls and unnecessary writes.

在本节中，我们将解释为什么在第 2 节中描述的分段复制存储系统上使用传统数据库会在网络 IO 和同步停顿方面带来难以承受的性能负担。 然后我们解释了我们将日志处理卸载到存储服务的方法，并通过实验演示我们的方法如何显着减少网络 IO。 最后，我们描述了我们在存储服务中使用的各种技术，以最大限度地减少同步停顿和不必要的写入。

### 3.1 The Burden of Amplified Writes

> Our model of segmenting a storage volume and replicating each segment 6 ways with a 4/6 write quorum gives us high resilience. Unfortunately, this model results in untenable performance for a traditional database like MySQL that generates many different actual I/Os for each application write. The high I/O volume is amplified by replication, imposing a heavy packets per second (PPS) burden. Also, the I/Os result in points of synchronization that stall pipelines and dilate latencies. While chain replication [8] and its alternatives can reduce network cost, they still suffer from synchronous stalls and additive latencies.

我们使用 4/6 写入仲裁对存储卷进行分段并以 6 种方式复制每个段的模型为我们提供了高弹性。不幸的是，对于像 MySQL 这样的传统数据库来说，这种模型会导致性能不稳定，它会为每次应用程序写入生成许多不同的实际 I/O。复制会放大高 I/O 量，从而造成沉重的每秒数据包 (PPS) 负担。此外，I/O 会导致同步点，这会导致管道停滞并延长延迟。虽然链复制 [8] 及其替代方案可以降低网络成本，但它们仍然受到同步停顿和附加延迟的影响。

> Let’s examine how writes work in a traditional database. A system like MySQL writes data pages to objects it exposes (e.g., heap files, b-trees etc.) as well as redo log records to a write-ahead log (WAL). Each redo log record consists of the difference between the after-image and the before-image of the page that was modified. A log record can be applied to the before-image of the page to produce its after-image.

让我们来看看写在传统数据库中是如何工作的。像 MySQL 这样的系统将数据页写入它公开的对象（例如，堆文件、b 树等），并将重做日志记录写入预写日志 (WAL)。每个重做日志记录都包含被修改页面的后映像和前映像之间的差异。可以将日志记录应用于页面的前映像以生成其后映像。

> In practice, other data must also be written. For instance, consider a synchronous mirrored MySQL configuration that achieves high availability across data-centers and operates in an active-standby configuration as shown in Figure 2. There is an active MySQL instance in AZ1 with networked storage on Amazon Elastic Block Store (EBS). There is also a standby MySQL instance in AZ2, also with networked storage on EBS. The writes made to the primary EBS volume are synchronized with the standby EBS volume using software mirroring.

实际上，还必须写入其他数据。例如，考虑一个同步镜像 MySQL 配置，它实现了跨数据中心的高可用性，并在如图 2 所示的主备配置中运行。 AZ1 中有一个活动 MySQL 实例，在 Amazon Elastic Block Store (EBS) 上有网络存储.在 AZ2 中还有一个备用 MySQL 实例，也在 EBS 上具有网络存储。对主 EBS 卷的写入使用软件镜像与备用 EBS 卷同步。

> Figure 2 shows the various types of data that the engine needs to write: the redo log, the binary (statement) log that is archived to Amazon Simple Storage Service (S3) in order to support point-in- time restores, the modified data pages, a second temporary write of the data page (double-write) to prevent torn pages, and finally the metadata (FRM) files. The figure also shows the order of the actual IO flow as follows. In Steps 1 and 2, writes are issued to EBS, which in turn issues it to an AZ-local mirror, and the acknowledgement is received when both are done. Next, in Step 3, the write is staged to the standby instance using synchronous block-level software mirroring. Finally, in steps 4 and 5, writes are written to the standby EBS volume and associated mirror.

图 2 显示了引擎需要写入的各种类型的数据：重做日志、存档到 Amazon Simple Storage Service (S3) 以支持时间点还原的二进制（语句）日志、修改后的数据页，第二次临时写入数据页（双写）以防止页面撕裂，最后是元数据 (FRM) 文件。图中还展示了实际IO流程的顺序如下。在第 1 步和第 2 步中，将写入发送到 EBS，后者又将其发送到 AZ 本地镜像，并在两者都完成后收到确认。接下来，在步骤 3 中，使用同步块级软件镜像将写入暂存到备用实例。最后，在步骤 4 和 5 中，将写入写入备用 EBS 卷和关联的镜像。

> The mirrored MySQL model described above is undesirable not only because of how data is written but also because of what data is written. First, steps 1, 3, and 5 are sequential and synchronous. Latency is additive because many writes are sequential. Jitter is amplified because, even on asynchronous writes, one must wait for the slowest operation, leaving the system at the mercy of outliers. From a distributed system perspective, this model can be viewed as having a 4/4 write quorum, and is vulnerable to failures and outlier performance. Second, user operations that are a result of OLTP applications cause many different types of writes often representing the same information in multiple ways – for example, the writes to the double write buffer in order to prevent torn pages in the storage infrastructure.

上面描述的镜像 MySQL 模型是不可取的，不仅因为数据的写入方式，还因为写入的数据。首先，步骤 1、3 和 5 是顺序和同步的。延迟是附加的，因为许多写入是连续的。抖动会被放大，因为即使是异步写入，也必须等待最慢的操作，使系统受异常值的支配。从分布式系统的角度来看，该模型可以被视为具有 4/4 的写入仲裁，并且容易出现故障和异常性能。其次，作为 OLTP 应用程序结果的用户操作会导致许多不同类型的写入，通常以多种方式表示相同的信息 - 例如，写入双写缓冲区以防止存储基础架构中的页面撕裂。

### 3.2 Offloading Redo Processing to Storage

> When a traditional database modifies a data page, it generates a redo log record and invokes a log applicator that applies the redo log record to the in-memory before-image of the page to produce its after-image. Transaction commit requires the log to be written, but the data page write may be deferred.

当传统数据库修改数据页时，它会生成重做日志记录并调用日志应用程序，该应用程序将重做日志记录应用于页面的内存前映像以生成其后映像。事务提交需要写入日志，但数据页写入可能会延迟。

> In Aurora, the only writes that cross the network are redo log records. No pages are ever written from the database tier, not for background writes, not for checkpointing, and not for cache eviction. Instead, the log applicator is pushed to the storage tier where it can be used to generate database pages in background or on demand. Of course, generating each page from the complete chain of its modifications from the beginning of time is prohibitively expensive. We therefore continually materialize database pages in the background to avoid regenerating them from scratch on demand every time. Note that background materialization is entirely optional from the perspective of correctness: as far as the engine is concerned, the log is the database, and any pages that the storage system materializes are simply a cache of log applications. Note also that, unlike checkpointing, only pages with a long chain of modifications need to be rematerialized. Checkpointing is governed by the length of the entire redo log chain. Aurora page materialization is governed by the length of the chain for a given page.

在 Aurora 中，唯一跨网络写入的是重做日志记录。永远不会从数据库层写入页面，不是用于后台写入，不是用于检查点，也不是用于缓存驱逐。相反，日志应用程序被推送到存储层，在那里它可用于在后台或按需生成数据库页面。当然，从一开始就从其修改的完整链中生成每个页面的成本高得令人望而却步。因此，我们不断在后台具体化数据库页面，以避免每次都按需从头开始重新生成它们。请注意，从正确性的角度来看，后台实体化是完全可选的：就引擎而言，日志就是数据库，存储系统实体化的任何页面都只是日志应用程序的缓存。另请注意，与检查点不同，只有具有长链修改的页面需要重新实现。检查点由整个重做日志链的长度控制。 Aurora 页面实现由给定页面的链长度控制。

> Our approach dramatically reduces network load despite amplifying writes for replication and provides performance as well as durability. The storage service can scale out I/Os in an embarrassingly parallel fashion without impacting write throughput of the database engine. For instance, Figure 3 shows an Aurora cluster with one primary instance and multiple replicas instances deployed across multiple AZs. In this model, the primary only writes log records to the storage service and streams those log records as well as metadata updates to the replica instances. The IO flow batches fully ordered log records based on a common destination (a logical segment, i.e., a PG) and delivers each batch to all 6 replicas where the batch is persisted on disk and the database engine waits for acknowledgements from 4 out of 6 replicas in order to satisfy the write quorum and consider the log records in question durable or hardened. The replicas use the redo log records to apply changes to their buffer caches.

尽管放大了复制写入，但我们的方法显着降低了网络负载，并提供了性能和持久性。存储服务可以以令人尴尬的并行方式横向扩展 I/O，而不会影响数据库引擎的写入吞吐量。例如，图 3 显示了一个 Aurora 集群，其中一个主实例和多个副本实例部署在多个可用区。在此模型中，主节点仅将日志记录写入存储服务，并将这些日志记录和元数据更新流式传输到副本实例。 IO 流根据公共目标（逻辑段，即 PG）对完全有序的日志记录进行批处理，并将每个批处理交付给所有 6 个副本，其中该批处理保留在磁盘上，并且数据库引擎等待来自 6 个中的 4 个的确认副本，以满足写入仲裁并考虑有问题的日志记录持久或硬化。副本使用重做日志记录将更改应用于其缓冲区缓存。

> To measure network I/O, we ran a test using the SysBench [9] write-only workload with a 100GB data set for both configurations described above: one with a synchronous mirrored MySQL configuration across multiple AZs and the other with RDS Aurora (with replicas across multiple AZs). In both instances, the test ran for 30 minutes against database engines running on an r3.8xlarge EC2 instance.

为了测量网络 I/O，我们使用 SysBench [9] 只写工作负载和 100GB 数据集对上述两种配置进行了测试：一种具有跨多个可用区的同步镜像 MySQL 配置，另一种具有 RDS Aurora（具有跨多个可用区的副本）。在这两个实例中，测试针对在 r3.8xlarge EC2 实例上运行的数据库引擎运行了 30 分钟。

> The results of our experiment are summarized in Table 1. Over the 30-minute period, Aurora was able to sustain 35 times more transactions than mirrored MySQL. The number of I/Os per transaction on the database node in Aurora was 7.7 times fewer than in mirrored MySQL despite amplifying writes six times with Aurora and not counting the chained replication within EBS nor the cross-AZ writes in MySQL. Each storage node sees unamplified writes, since it is only one of the six copies, resulting in 46 times fewer I/Os requiring processing at this tier. The savings we obtain by writing less data to the network allow us to aggressively replicate data for durability and availability and issue requests in parallel to minimize the impact of jitter.

我们的实验结果总结在表 1 中。在 30 分钟的时间内，Aurora 能够维持比镜像 MySQL 多 35 倍的事务。尽管使用 Aurora 将写入放大了六倍，并且不计算 EBS 内的链式复制和 MySQL 中的跨可用区写入，但 Aurora 中数据库节点上每个事务的 I/O 数量比镜像 MySQL 少 7.7 倍。每个存储节点都看到未放大的写入，因为它只是六个副本中的一个，因此需要在该层处理的 I/O 减少了 46 倍。我们通过向网络写入更少的数据而获得的节省使我们能够积极复制数据以实现持久性和可用性，并并行发出请求以最小化抖动的影响。

> Moving processing to a storage service also improves availability by minimizing crash recovery time and eliminates jitter caused by background processes such as checkpointing, background data page writing and backups.

将处理转移到存储服务还可以通过最大限度地减少崩溃恢复时间来提高可用性，并消除由检查点、后台数据页面写入和备份等后台进程引起的抖动。

> Let’s examine crash recovery. In a traditional database, after a crash the system must start from the most recent checkpoint and replay the log to ensure that all persisted redo records have been applied. In Aurora, durable redo record application happens at the storage tier, continuously, asynchronously, and distributed across the fleet. Any read request for a data page may require some redo records to be applied if the page is not current. As a result, the process of crash recovery is spread across all normal foreground processing. Nothing is required at database startup.

让我们来看看崩溃恢复。在传统数据库中，崩溃后系统必须从最近的检查点开始并重放日志以确保已应用所有持久化重做记录。在 Aurora 中，持久重做记录应用程序发生在存储层，连续、异步并分布在整个队列中。如果页面不是当前页面，则对数据页面的任何读取请求可能需要应用一些重做记录。因此，崩溃恢复的过程分布在所有正常的前台处理中。数据库启动时什么都不需要。

### 3.3 Storage Service Design Points

> A core design tenet for our storage service is to minimize the latency of the foreground write request. We move the majority of storage processing to the background. Given the natural variability between peak to average foreground requests from the storage tier, we have ample time to perform these tasks outside the foreground path. We also have the opportunity to trade CPU for disk. For example, it isn’t necessary to run garbage collection (GC) of old page versions when the storage node is busy processing foreground write requests unless the disk is approaching capacity. In Aurora, background processing has negative correlation with foreground processing. This is unlike a traditional database, where background writes of pages and checkpointing have positive correlation with the foreground load on the system. If we build up a backlog on the system, we will throttle foreground activity to prevent a long queue buildup. Since segments are placed with high entropy across the various storage nodes in our system, throttling at one storage node is readily handled by our 4/6 quorum writes, appearing as a slow node.

我们存储服务的核心设计原则是尽量减少前台写入请求的延迟。我们将大部分存储处理移至后台。鉴于来自存储层的峰值与平均前台请求之间的自然可变性，我们有足够的时间在前台路径之外执行这些任务。我们也有机会用 CPU 换磁盘。例如，当存储节点忙于处理前台写入请求时，除非磁盘接近容量，否则不需要运行旧页面版本的垃圾收集（GC）。在 Aurora 中，后台处理与前台处理呈负相关。这与传统数据库不同，在传统数据库中，页面的后台写入和检查点与系统的前台负载呈正相关。如果我们在系统上积压了一个积压，我们将限制前台活动以防止长队列积聚。由于段在我们系统中的各个存储节点上以高熵放置，因此我们的 4/6 仲裁写入可以轻松处理一个存储节点的节流，表现为慢速节点

> Let’s examine the various activities on the storage node in more detail. As seen in Figure 4, it involves the following steps: (1) receive log record and add to an in-memory queue, (2) persist record on disk and acknowledge, (3) organize records and identify gaps in the log since some batches may be lost, (4) gossip with peers to fill in gaps, (5) coalesce log records into new data pages, (6) periodically stage log and new pages to S3, (7) periodically garbage collect old versions, and finally (8) periodically validate CRC codes on pages.

让我们更详细地检查存储节点上的各种活动。如图 4 所示，它包括以下步骤：(1) 接收日志记录并添加到内存队列中，(2) 将记录保存在磁盘上并确认，(3) 组织记录并识别日志中的间隙，因为某些批次可能丢失，(4) 与 peer 闲聊以填补空白，(5) 将日志记录合并到新的数据页面中，(6) 定期将日志和新页面暂存到 S3，(7) 定期对旧版本进行垃圾收集，最后(8) 定期验证页面上的 CRC 代码。

> Note that not only are each of the steps above asynchronous, only steps (1) and (2) are in the foreground path potentially impacting latency.

请注意，不仅上述每个步骤都是异步的，只有步骤 (1) 和 (2) 位于可能影响延迟的前台路径中。

## 4. THE LOG MARCHES FORWARD
> In this section, we describe how the log is generated from the database engine so that the durable state, the runtime state, and the replica state are always consistent. In particular, we will describe how consistency is implemented efficiently without an expensive 2PC protocol. First, we show how we avoid expensive redo processing on crash recovery. Next, we explain normal operation and how we maintain runtime and replica state. Finally, we provide details of our recovery process.

在本节中，我们将描述如何从数据库引擎生成日志，从而使持久状态、运行时状态和副本状态始终保持一致。 特别是，我们将描述如何在没有昂贵的 2PC 协议的情况下有效地实现一致性。 首先，我们展示了如何避免在崩溃恢复时进行昂贵的重做处理。 接下来，我们解释正常操作以及我们如何维护运行时和副本状态。 最后，我们提供了恢复过程的详细信息。

### 4.1 Solution sketch: Asynchronous Processing

> Since we model the database as a redo log stream (as described in Section 3), we can exploit the fact that the log advances as an ordered sequence of changes. In practice, each log record has an associated Log Sequence Number (LSN) that is a monotonically increasing value generated by the database.

由于我们将数据库建模为重做日志流（如第 3 节所述），我们可以利用日志作为有序更改序列前进的事实。实际上，每条日志记录都有一个关联的日志序列号 (LSN)，它是由数据库生成的单调递增值。

> This lets us simplify a consensus protocol for maintaining state by approaching the problem in an asynchronous fashion instead of using a protocol like 2PC which is chatty and intolerant of failures. At a high level, we maintain points of consistency and durability, and continually advance these points as we receive acknowledgements for outstanding storage requests. Since any individual storage node might have missed one or more log records, they gossip with the other members of their PG, looking for gaps and fill in the holes. The runtime state maintained by the database lets us use single segment reads rather than quorum reads except on recovery when the state is lost and has to be rebuilt.

这让我们可以通过以异步方式解决问题来简化用于维护状态的共识协议，而不是使用像 2PC 这样健谈且不能容忍故障的协议。在高层次上，我们维护一致性和持久性点，并在我们收到对未完成存储请求的确认时不断推进这些点。由于任何单个存储节点都可能遗漏了一条或多条日志记录，因此它们会与 PG 的其他成员闲聊，寻找差距并填补漏洞。数据库维护的运行时状态允许我们使用单段读取而不是仲裁读取，除非在状态丢失且必须重建时进行恢复

> The database may have multiple outstanding isolated transactions, which can complete (reach a finished and durable state) in a different order than initiated. Supposing the database crashes or reboots, the determination of whether to roll back is separate for each of these individual transactions. The logic for tracking partially completed transactions and undoing them is kept in the database engine, just as if it were writing to simple disks. However, upon restart, before the database is allowed to access the storage volume, the storage service does its own recovery which is focused not on user-level transactions, but on making sure that the database sees a uniform view of storage despite its distributed nature.

数据库可能有多个未完成的隔离事务，它们可以以与启动不同的顺序完成（达到完成和持久状态）。假设数据库崩溃或重新启动，对于这些单独事务中的每一个，确定是否回滚是分开的。跟踪部分完成的事务和撤消它们的逻辑保存在数据库引擎中，就像写入简单的磁盘一样。但是，在重新启动时，在允许数据库访问存储卷之前，存储服务会进行自己的恢复，这不是专注于用户级别的事务，而是确保数据库看到统一的存储视图，尽管它具有分布式特性.

> The storage service determines the highest LSN for which it can guarantee availability of all prior log records (this is known as the VCL or Volume Complete LSN). During storage recovery, every log record with an LSN larger than the VCL must be truncated. The database can, however, further constrain a subset of points that are allowable for truncation by tagging log records and identifying them as CPLs or Consistency Point LSNs. We therefore define VDL or the Volume Durable LSN as the highest CPL that is smaller than or equal to VCL and truncate all log records with LSN greater than the VDL. For example, even if we have the complete data up to LSN 1007, the database may have declared that only 900, 1000, and 1100 are CPLs, in which case, we must truncate at 1000. We are complete to 1007, but only durable to 1000.

存储服务确定它可以保证所有先前日志记录的可用性的最高 LSN（这称为 VCL 或 Volume Complete LSN）。在存储恢复期间，每个 LSN 大于 VCL 的日志记录都必须被截断。但是，数据库可以通过标记日志记录并将它们标识为 CPL 或一致性点 LSN 来进一步限制允许截断的点的子集。因此，我们将 VDL 或 Volume Durable LSN 定义为小于或等于 VCL 的最高 CPL，并截断 LSN 大于 VDL 的所有日志记录。例如，即使我们有到 LSN 1007 的完整数据，数据库可能已经声明只有 900、1000 和 1100 是 CPL，在这种情况下，我们必须在 1000 处截断。我们完整到 1007，但只有持久性到 1000。

> Completeness and durability are therefore different and a CPL can be thought of as delineating some limited form of storage system transaction that must be accepted in order. If the client has no use for such distinctions, it can simply mark every log record as a CPL. In practice, the database and storage interact as follows:
> 1. Each database-level transaction is broken up into multiple mini-transactions (MTRs) that are ordered and must be performed atomically.
> 2. Each mini-transaction is composed of multiple contiguous log records (as many as needed).
> 3. The final log record in a mini-transaction is a CPL. 

因此，完整性和持久性是不同的，可以将 CPL 视为描述必须按顺序接受的某种有限形式的存储系统事务。 如果客户端不需要这种区分，它可以简单地将每个日志记录标记为 CPL。 在实践中，数据库和存储交互如下：
1. 每个数据库级事务被分解为多个有序且必须以原子方式执行的小事务 (MTR)。
2. 每个小交易由多个连续的日志记录组成（根据需要多多）。
3. 小型事务中的最终日志记录是 CPL。

> On recovery, the database talks to the storage service to establish the durable point of each PG and uses that to establish the VDL and then issue commands to truncate the log records above VDL.

在恢复时，数据库与存储服务对话以建立每个 PG 的持久点，并使用它来建立 VDL，然后发出命令截断 VDL 之上的日志记录。

### 4.2 NormalOperation
> We now describe the “normal operation” of the database engine and focus in turn on writes, reads, commits, and replicas.

我们现在描述数据库引擎的“正常操作”，并依次关注写入、读取、提交和副本。

#### 4.2.1 Writes

> In Aurora, the database continuously interacts with the storage service and maintains state to establish quorum, advance volume durability, and register transactions as committed. For instance, in the normal/forward path, as the database receives acknowledgements to establish the write quorum for each batch of log records, it advances the current VDL. At any given moment, there can be a large number of concurrent transactions active in the database, each generating their own redo log records. The database allocates a unique ordered LSN for each log record subject to a constraint that no LSN is allocated with a value that is greater than the sum of the current VDL and a constant called the LSN Allocation Limit (LAL) (currently set to 10 million). This limit ensures that the database does not get too far ahead of the storage system and introduces back-pressure that can throttle the incoming writes if the storage or network cannot keep up.

在 Aurora 中，数据库持续与存储服务交互并维护状态以建立仲裁、提高卷持久性并将事务注册为已提交。例如，在正常/转发路径中，当数据库收到确认以建立每批日志记录的写入仲裁时，它会推进当前的 VDL。在任何给定时刻，数据库中都可能有大量并发事务处于活动状态，每个事务都会生成自己的重做日志记录。数据库为每条日志记录分配一个唯一有序的 LSN，受约束的约束是没有 LSN 分配的值大于当前 VDL 和称为 LSN 分配限制 (LAL) 的常数之和（当前设置为 1000 万） ）。此限制可确保数据库不会领先于存储系统太远，并引入背压，如果存储或网络跟不上，则会限制传入的写入。

> Note that each segment of each PG only sees a subset of log records in the volume that affect the pages residing on that segment. Each log record contains a backlink that identifies the previous log record for that PG. These backlinks can be used to track the point of completeness of the log records that have reached each segment to establish a Segment Complete LSN(SCL) that identifies the greatest LSN below which all log records of the PG have been received. The SCL is used by the storage nodes when they gossip with each other in order to find and exchange log records that they are missing.

请注意，每个 PG 的每个段只能看到卷中影响驻留在该段上的页面的日志记录的子集。每个日志记录都包含一个反向链接，用于标识该 PG 的前一个日志记录。这些反向链接可用于跟踪已到达每个段的日志记录的完整性点，以建立一个段完整 LSN（SCL），该 LSN 标识最大 LSN，低于该 LSN 的 PG 的所有日志记录都已收到。存储节点在相互闲聊时使用 SCL，以便查找和交换它们丢失的日志记录。

#### 4.2.2 Commits
> In Aurora, transaction commits are completed asynchronously. When a client commits a transaction, the thread handling the commit request sets the transaction aside by recording its “commit LSN” as part of a separate list of transactions waiting on commit and moves on to perform other work. The equivalent to the WAL protocol is based on completing a commit, if and only if, the latest VDL is greater than or equal to the transaction’s commit LSN. As the VDL advances, the database identifies qualifying transactions that are waiting to be committed and uses a dedicated thread to send commit acknowledgements to waiting clients. Worker threads do not pause for commits, they simply pull other pending requests and continue processing.

在 Aurora 中，事务提交是异步完成的。 当客户端提交事务时，处理提交请求的线程通过将其“提交 LSN”记录为等待提交的单独事务列表的一部分来将事务搁置一旁，并继续执行其他工作。 WAL 协议的等价物基于完成提交，当且仅当最新的 VDL 大于或等于事务的提交 LSN。 随着 VDL 的推进，数据库会识别等待提交的合格事务，并使用专用线程向等待的客户端发送提交确认。 工作线程不会因为提交而暂停，它们只是拉动其他待处理的请求并继续处理。

#### 4.2.3 Reads

> In Aurora, as with most databases, pages are served from the buffer cache and only result in a storage IO request if the page in question is not present in the cache.

在 Aurora 中，与大多数数据库一样，页面由缓冲区缓存提供，只有在缓存中不存在相关页面时才会产生存储 IO 请求。

> If the buffer cache is full, the system finds a victim page to evict from the cache. In a traditional system, if the victim is a “dirty page” then it is flushed to disk before replacement. This is to ensure that a subsequent fetch of the page always results in the latest data. While the Aurora database does not write out pages on eviction (or anywhere else), it enforces a similar guarantee: a page in the buffer cache must always be of the latest version. The guarantee is implemented by evicting a page from the cache only if its “page LSN” (identifying the log record associated with the latest change to the page) is greater than or equal to the VDL. This protocol ensures that: (a) all changes in the page have been hardened in the log, and (b) on a cache miss, it is sufficient to request a version of the page as of the current VDL to get its latest durable version.

如果缓冲区缓存已满，系统会查找要从缓存中逐出的受害者页面。在传统系统中，如果受害者是“脏页”，则在替换之前将其刷新到磁盘。这是为了确保页面的后续提取始终产生最新的数据。虽然 Aurora 数据库不会在逐出（或其他任何地方）时写出页面，但它强制执行类似的保证：缓冲区缓存中的页面必须始终是最新版本。只有当页面的“页面 LSN”（标识与页面的最新更改相关联的日志记录）大于或等于 VDL 时，才通过从缓存中逐出页面来实现这一保证。该协议确保：(a) 页面中的所有更改都已在日志中硬化，并且 (b) 在缓存未命中时，足以请求当前 VDL 的页面版本以获取其最新的持久版本.

> The database does not need to establish consensus using a read quorum under normal circumstances. When reading a page from disk, the database establishes a read-point, representing the VDL at the time the request was issued. The database can then select a storage node that is complete with respect to the read point, knowing that it will therefore receive an up to date version. A page that is returned by the storage node must be consistent with the expected semantics of a mini-transaction (MTR) in the database. Since the database directly manages feeding log records to storage nodes and tracking progress (i.e., the SCL of each segment), it normally knows which segment is capable of satisfying a read (the segments whose SCL is greater than the read-point) and thus can issue a read request directly to a segment that has sufficient data.

数据库在正常情况下不需要使用读仲裁来建立共识。从磁盘读取页面时，数据库会建立一个读取点，表示发出请求时的 VDL。数据库然后可以选择相对于读取点完整的存储节点，知道它将因此接收最新版本。存储节点返回的页面必须与数据库中小事务 (MTR) 的预期语义一致。由于数据库直接管理向存储节点馈送日志记录和跟踪进度（即每个段的 SCL），因此它通常知道哪个段能够满足读取（SCL 大于读取点的段），从而可以直接向有足够数据的段发出读请求

> Given that the database is aware of all outstanding reads, it can compute at any time the Minimum Read Point LSN on a per-PG basis. If there are read replicas the writer gossips with them to establish the per-PG Minimum Read Point LSN across all nodes. This value is called the Protection Group Min Read Point LSN (PGMRPL) and represents the “low water mark” below which all the log records of the PG are unnecessary. In other words, a storage node segment is guaranteed that there will be no read page requests with a read-point that is lower than the PGMRPL. Each storage node is aware of the PGMRPL from the database and can, therefore, advance the materialized pages on disk by coalescing the older log records and then safely garbage collecting them.

鉴于数据库知道所有未完成的读取，它可以随时计算每个 PG 的最小读取点 LSN。如果有只读副本，作者会与它们闲聊以在所有节点上建立每个 PG 的最小读取点 LSN。该值称为保护组最小读取点 LSN (PGMRPL)，代表“低水位线”，低于该值时 PG 的所有日志记录都是不必要的。换句话说，保证一个存储节点段不会有读点低于 PGMRPL 的读页请求。每个存储节点都知道来自数据库的 PGMRPL，因此可以通过合并较旧的日志记录然后安全地对它们进行垃圾回收来推进磁盘上的物化页面。

> The actual concurrency control protocols are executed in the database engine exactly as though the database pages and undo segments are organized in local storage as with traditional MySQL.

实际的并发控制协议在数据库引擎中执行，就好像数据库页面和撤消段在本地存储中组织的方式与传统 MySQL 完全一样。

#### 4.2.4 Replicas
> In Aurora, a single writer and up to 15 read replicas can all mount a single shared storage volume. As a result, read replicas add no additional costs in terms of consumed storage or disk write operations. To minimize lag, the log stream generated by the writer and sent to the storage nodes is also sent to all read replicas. In the reader, the database consumes this log stream by considering each log record in turn. If the log record refers to a page in the reader's buffer cache, it uses the log applicator to apply the specified redo operation to the page in the cache. Otherwise it simply discards the log record. Note that the replicas consume log records asynchronously from the perspective of the writer, which acknowledges user commits independent of the replica. The replica obeys the following two important rules while applying log records: (a) the only log records that will be applied are those whose LSN is less than or equal to the VDL, and (b) the log records that are part of a single mini-transaction are applied atomically in the replica's cache to ensure that the replica sees a consistent view of all database objects. In practice, each replica typically lags behind the writer by a short interval (20 ms or less).

在 Aurora 中，单个写入器和最多 15 个只读副本都可以挂载单个共享存储卷。因此，只读副本不会在消耗的存储或磁盘写入操作方面增加额外的成本。为了最小化延迟，由写入器生成并发送到存储节点的日志流也会发送到所有只读副本。在阅读器中，数据库通过依次考虑每个日志记录来使用此日志流。如果日志记录引用读取器缓冲区缓存中的页面，则它使用日志应用程序将指定的重做操作应用于缓存中的页面。否则它只是丢弃日志记录。请注意，从作者的角度来看，副本异步使用日志记录，这会独立于副本确认用户提交。副本在应用日志记录时遵循以下两个重要规则：(a) 将应用的唯一日志记录是那些 LSN 小于或等于 VDL 的日志记录，以及 (b) 属于单个日志记录的日志记录小事务在副本的缓存中以原子方式应用，以确保副本看到所有数据库对象的一致视图。在实践中，每个副本通常落后于写入者一小段时间（20 毫秒或更短）。

### 4.3 Recovery

> Most traditional databases use a recovery protocol such as ARIES [7] that depends on the presence of a write-ahead log (WAL) that can represent the precise contents of all committed transactions. These systems also periodically checkpoint the database to establish points of durability in a coarse-grained fashion by flushing dirty pages to disk and writing a checkpoint record to the log. On restart, any given page can either miss some committed data or contain uncommitted data. Therefore, on crash recovery the system processes the redo log records since the last checkpoint by using the log applicator to apply each log record to the relevant database page. This process brings the database pages to a consistent state at the point of failure after which the in-flight transactions during the crash can be rolled back by executing the relevant undo log records. Crash recovery can be an expensive operation. Reducing the checkpoint interval helps, but at the expense of interference with foreground transactions. No such tradeoff is required with Aurora.

大多数传统数据库使用诸如 ARIES [7] 之类的恢复协议，该协议依赖于可以表示所有已提交事务的精确内容的预写日志 (WAL) 的存在。这些系统还通过将脏页刷新到磁盘并将检查点记录写入日志来定期检查数据库，以粗粒度的方式建立持久性点。在重新启动时，任何给定的页面可能会丢失一些已提交的数据或包含未提交的数据。因此，在崩溃恢复时，系统通过使用日志应用程序将每个日志记录应用到相关数据库页面来处理自上次检查点以来的重做日志记录。此过程使数据库页面在故障点处于一致状态，此后可以通过执行相关的撤消日志记录来回滚崩溃期间进行中的事务。崩溃恢复可能是一项昂贵的操作。减少检查点间隔会有所帮助，但代价是会干扰前台事务。 Aurora 不需要这样的权衡。

> A great simplifying principle of a traditional database is that the same redo log applicator is used in the forward processing path as well as on recovery where it operates synchronously and in the foreground while the database is offline. We rely on the same principle in Aurora as well, except that the redo log applicator is decoupled from the database and operates on storage nodes, in parallel, and all the time in the background. Once the database starts up it performs volume recovery in collaboration with the storage service and as a result, an Aurora database can recover very quickly (generally under 10 seconds) even if it crashed while processing over 100,000 write statements per second.

传统数据库的一个极大简化原则是在前向处理路径以及恢复时使用相同的重做日志应用程序，当数据库离线时，它在前台同步运行。我们在 Aurora 中也依赖相同的原则，不同之处在于重做日志应用程序与数据库分离并在存储节点上并行运行，并且始终在后台运行。数据库启动后，它会与存储服务协作执行卷恢复，因此，即使 Aurora 数据库在每秒处理超过 100,000 个写入语句时崩溃，它也可以非常快速地恢复（通常在 10 秒内）。

> The database does need to reestablish its runtime state after a crash. In this case, it contacts for each PG, a read quorum of segments which is sufficient to guarantee discovery of any data that could have reached a write quorum. Once the database has established a read quorum for every PG it can recalculate the VDL above which data is truncated by generating a truncation range that annuls every log record after the new VDL, up to and including an end LSN which the database can prove is at least as high as the highest possible outstanding log record that could ever have been seen. The database infers this upper bound because it allocates LSNs, and limits how far allocation can occur above VDL (the 10 million limit described earlier). The truncation ranges are versioned with epoch numbers, and written durably to the storage service so that there is no confusion over the durability of truncations in case recovery is interrupted and restarted.

数据库在崩溃后确实需要重新建立其运行时状态。在这种情况下，它为每个 PG 联系一个段的读取仲裁，这足以保证发现任何可能已达到写入仲裁的数据。一旦数据库为每个 PG 建立了读取仲裁，它就可以重新计算 VDL 以上的数据，通过生成一个截断范围来取消新 VDL 之后的每个日志记录，直到并包括数据库可以证明的结束 LSN至少与可能见过的最高未完成日志记录一样高。数据库推断此上限是因为它分配了 LSN，并限制了分配可以在 VDL 之上发生的程度（前面描述的 1000 万个限制）。截断范围使用纪元编号进行版本控制，并持久写入存储服务，以便在恢复中断和重新启动时不会混淆截断的持久性。

> The database still needs to perform undo recovery to unwind the operations of in-flight transactions at the time of the crash. However, undo recovery can happen when the database is online after the system builds the list of these in-flight transactions from the undo segments.

数据库仍然需要执行undo恢复来解除崩溃时正在进行的事务的操作。但是，在系统从撤消段构建这些正在进行的事务的列表之后，当数据库处于联机状态时，可能会发生撤消恢复。

## 5 PUTTING IT ALL TOGETHER

> In this section, we describe the building blocks of Aurora as shown with a bird’s eye view in Figure 5.

在本节中，我们将描述 Aurora 的构建块，如图 5 中的鸟瞰图所示。

> The database engine is a fork of “community” MySQL/InnoDB and diverges primarily in how InnoDB reads and writes data to disk. In community InnoDB, a write operation results in data being modified in buffer pages, and the associated redo log records written to buffers of the WAL in LSN order. On transaction commit, the WAL protocol requires only that the redo log records of the transaction are durably written to disk. The actual modified buffer pages are also written to disk eventually through a double-write technique to avoid partial page writes. These page writes take place in the background, or during eviction from the cache, or while taking a checkpoint. In addition to the IO Subsystem, InnoDB also includes the transaction subsystem, the lock manager, a B+-Tree implementation and the associated notion of a “mini transaction” (MTR). An MTR is a construct only used inside InnoDB and models groups of operations that must be executed atomically (e.g., split/merge of B+-Tree pages).

数据库引擎是“社区”MySQL/InnoDB 的一个分支，主要区别在于 InnoDB 读取数据和将数据写入磁盘的方式。在社区 InnoDB 中，写操作会导致缓冲区页面中的数据被修改，并且关联的重做日志记录以 LSN 顺序写入 WAL 的缓冲区。在事务提交时，WAL 协议只要求将事务的重做日志记录持久地写入磁盘。实际修改的缓冲区页面也最终通过双写技术写入磁盘，以避免部分页面写入。这些页面写入发生在后台，或者在从缓存中逐出期间，或者在获取检查点时。除了 IO 子系统，InnoDB 还包括事务子系统、锁管理器、B+-Tree 实现和相关的“迷你事务”（MTR）概念。 MTR 是仅在 InnoDB 内部使用的构造，并对必须以原子方式执行的操作组进行建模（例如，B+-Tree 页面的拆分/合并）

> In the Aurora InnoDB variant, the redo log records representing the changes that must be executed atomically in each MTR are organized into batches that are sharded by the PGs each log record belongs to, and these batches are written to the storage service. The final log record of each MTR is tagged as a consistency point. Aurora supports exactly the same isolation levels that are supported by community MySQL in the writer (the standard ANSI levels and Snapshot Isolation or consistent reads). Aurora read replicas get continuous information on transaction starts and commits in the writer and use this information to support snapshot isolation for local transactions that are of course read-only. Note that concurrency control is implemented entirely in the database engine without impacting the storage service. The storage service presents a unified view of the underlying data that is logically identical to what you would get by writing the data to local storage in community InnoDB.

在 Aurora InnoDB 变体中，代表必须在每个 MTR 中原子执行的更改的重做日志记录被组织成批次，这些批次由每个日志记录所属的 PG 分片，并将这些批次写入存储服务。每个 MTR 的最终日志记录都被标记为一致性点。 Aurora 支持与社区 MySQL 完全相同的隔离级别（标准 ANSI 级别和快照隔离或一致性读取）。 Aurora 只读副本在写入器中获取有关事务启动和提交的连续信息，并使用此信息支持本地事务的快照隔离，这些事务当然是只读的。请注意，并发控制完全在数据库引擎中实现，不会影响存储服务。存储服务提供底层数据的统一视图，在逻辑上与将数据写入社区 InnoDB 中的本地存储所获得的视图相同。

> Aurora leverages Amazon Relational Database Service (RDS) for its control plane. RDS includes an agent on the database instance called the Host Manager (HM) that monitors a cluster’s health and determines if it needs to fail over, or if an instance needs to be replaced. Each database instance is part of a cluster that consists of a single writer and zero or more read replicas. The instances of a cluster are in a single geographical region (e.g., us-east-1, us- west-1 etc.), are typically placed in different AZs, and connect to a storage fleet in the same region. For security, we isolate the communication between the database, applications and storage. In practice, each database instance can communicate on three Amazon Virtual Private Cloud (VPC) networks: the customer VPC through which customer applications interact with the engine, the RDS VPC through which the database engine and control plane interact with each other, and the Storage VPC through which the database interacts with storage services.

Aurora 利用 Amazon Relational Database Service (RDS) 作为其控制平面。 RDS 在数据库实例上包含一个称为主机管理器 (HM) 的代理，用于监控集群的运行状况并确定是否需要进行故障转移，或者是否需要更换实例。每个数据库实例都是由单个写入器和零个或多个只读副本组成的集群的一部分。集群的实例位于单个地理区域（例如，us-east-1、us-west-1 等）中，通常放置在不同的可用区中，并连接到同一区域的存储队列。为了安全起见，我们隔离了数据库、应用程序和存储之间的通信。实际上，每个数据库实例都可以在三个 Amazon Virtual Private Cloud (VPC) 网络上进行通信：客户应用程序通过其与引擎交互的客户 VPC、数据库引擎和控制平面通过其相互交互的 RDS VPC 以及存储数据库通过 VPC 与存储服务交互。

> The storage service is deployed on a cluster of EC2 VMs that are provisioned across at least 3 AZs in each region and is collectively responsible for provisioning multiple customer storage volumes, reading and writing data to and from those volumes, and backing up and restoring data from and to those volumes. The storage nodes manipulate local SSDs and interact with database engine instances, other peer storage nodes, and the backup/restore services that continuously backup changed data to S3 and restore data from S3 as needed. The storage control plane uses the Amazon DynamoDB database service for persistent storage of cluster and storage volume configuration, volume metadata, and a detailed description of data backed up to S3. For orchestrating long-running operations, e.g. a database volume restore operation or a repair (re-replication) operation following a storage node failure, the storage control plane uses the Amazon Simple Workflow Service. Maintaining a high level of availability requires pro-active, automated, and early detection of real and potential problems, before end users are impacted. All critical aspects of storage operations are constantly monitored using metric collection services that raise alarms if key performance or availability metrics indicate a cause for concern.

存储服务部署在 EC2 虚拟机集群上，该集群跨每个区域的至少 3 个可用区进行供应，共同负责供应多个客户存储卷、从这些卷读取和写入数据以及备份和恢复数据从和到这些卷。存储节点操作本地 SSD 并与数据库引擎实例、其他对等存储节点以及备份/恢复服务交互，这些服务不断将更改的数据备份到 S3，并根据需要从 S3 恢复数据。存储控制平面使用 Amazon DynamoDB 数据库服务持久存储集群和存储卷配置、卷元数据以及备份到 S3 的数据的详细描述。用于编排长时间运行的操作，例如存储节点故障后的数据库卷还原操作或修复（重新复制）操作，存储控制平面使用 Amazon Simple Workflow Service。保持高水平的可用性需要在最终用户受到影响之前主动、自动化和及早地检测实际和潜在问题。使用指标收集服务持续监控存储操作的所有关键方面，如果关键性能或可用性指标表明存在问题，则会发出警报。

## 6 PERFORMANCE RESULTS
> In this section, we will share our experiences in running Aurora as a production service that was made “Generally Available” in July 2015. We begin with a summary of results running industry standard benchmarks and then present some performance results from our customers.

在本节中，我们将分享我们将 Aurora 作为生产服务运行的经验，该服务于 2015 年 7 月“普遍可用”。我们首先总结运行行业标准基准的结果，然后展示我们客户的一些性能结果。

### 6.1 ResultswithStandardBenchmarks
> Here we present results of different experiments that compare the performance of Aurora and MySQL using industry standard benchmarks such as SysBench and TPC-C variants. We ran MySQL on instances that are attached to an EBS volume with 30K provisioned IOPS. Except when stated otherwise, these are r3.8xlarge EC2 instances with 32 vCPUs and 244GB of memory and features the Intel Xeon E5-2670 v2 (Ivy Bridge) processors. The buffer cache on the r3.8xlarge is set to 170GB.

在这里，我们展示了不同实验的结果，这些实验使用行业标准基准（例如 SysBench 和 TPC-C 变体）来比较 Aurora 和 MySQL 的性能。 我们在附加到具有 30K 预配置 IOPS 的 EBS 卷的实例上运行 MySQL。 除非另有说明，否则这些实例是 r3.8xlarge EC2 实例，具有 32 个 vCPU 和 244GB 内存，并具有 Intel Xeon E5-2670 v2 (Ivy Bridge) 处理器。 r3.8xlarge 上的缓冲区缓存设置为 170GB。

#### 6.1.1 Scaling with instance sizes

> In this experiment, we report that throughput in Aurora can scale linearly with instance sizes, and with the highest instance size can be 5x that of MySQL 5.6 and MySQL 5.7. Note that Aurora is currently based on the MySQL 5.6 code base. We ran the SysBench read-only and write-only benchmarks for a 1GB data set (250 tables) on 5 EC2 instances of the r3 family (large, xlarge, 2xlarge, 4xlarge, 8xlarge). Each instance size has exactly half the vCPUs and memory of the immediately larger instance.

在这个实验中，我们报告说 Aurora 的吞吐量可以随实例大小线性扩展，最高实例大小可以是 MySQL 5.6 和 MySQL 5.7 的 5 倍。 请注意，Aurora 当前基于 MySQL 5.6 代码库。 我们在 r3 系列的 5 个 EC2 实例（large、xlarge、2xlarge、4xlarge、8xlarge）上对 1GB 数据集（250 个表）运行了 SysBench 只读和只写基准测试。 每个实例大小的 vCPU 和内存正好是直接更大的实例的一半。

> The results are shown in Figure 7 and Figure 6, and measure the performance in terms of write and read statements per second respectively. Aurora’s performance doubles for each higher instance size and for the r3.8xlarge achieves 121,000 writes/sec and 600,000 reads/sec which is 5x that of MySQL 5.7 which tops out at 20,000 reads/sec and 125,000 writes/sec.

结果如图 7 和图 6 所示，分别衡量每秒写入和读取语句的性能。 Aurora 的性能随着实例大小的增加而增加一倍，r3.8xlarge 实现了 121,000 次写入/秒和 600,000 次读取/秒，是 MySQL 5.7 的 5 倍，最高为 20,000 次读取/秒和 125,000 次写入/秒。

#### 6.1.2 Throughput with varying data sizes
> In this experiment, we report that throughput in Aurora significantly exceeds that of MySQL even with larger data sizes including workloads with out-of-cache working sets. Table 2 shows that for the SysBench write-only workload, Aurora can be up to 67x faster than MySQL with a database size of 100GB. Even for a database size of 1TB with an out-of-cache workload, Aurora is still 34x faster than MySQL.

在这个实验中，我们报告说 Aurora 的吞吐量显着超过 MySQL，即使数据量更大，包括具有缓存外工作集的工作负载。 表 2 显示，对于 SysBench 只写工作负载，Aurora 的速度比 MySQL 快 67 倍，数据库大小为 100GB。 即使对于具有缓存外工作负载的 1TB 数据库大小，Aurora 仍然比 MySQL 快 34 倍。

#### 6.1.3 Scaling with user connections
> In this experiment, we report that throughput in Aurora can scale with the number of client connections. Table 3 shows the results of running the SysBench OLTP benchmark in terms of writes/sec as the number of connections grows from 50 to 500 to 5000. While Aurora scales from 40,000 writes/sec to 110,000 writes/sec, the throughput in MySQL peaks at around 500 connections and then drops sharply as the number of connections grows to 5000.

在这个实验中，我们报告说 Aurora 的吞吐量可以随着客户端连接的数量而扩展。 表 3 显示了运行 SysBench OLTP 基准测试的结果，连接数从 50 增加到 500 再到 5000。虽然 Aurora 从 40,000 次写入/秒扩展到 110,000 次写入/秒，但 MySQL 中的吞吐量峰值为 大约 500 个连接，然后随着连接数增加到 5000 急剧下降。

#### 6.1.4 Scaling with Replicas
> In this experiment, we report that the lag in an Aurora read replica is significantly lower than that of a MySQL replica even with more intense workloads. Table 4 shows that as the workload varies from 1,000 to 10,000 writes/second, the replica lag in Aurora grows from 2.62 milliseconds to 5.38 milliseconds. In contrast, the replica lag in MySQL grows from under a second to 300 seconds. At 10,000 writes/second Aurora has a replica lag that is several orders of magnitude smaller than that of MySQL. Replica lag is measured in terms of the time it takes for a committed transaction to be visible in the replica.

在此实验中，我们报告说，即使工作负载更密集，Aurora 只读副本的延迟也明显低于 MySQL 副本。 表 4 显示，当工作负载从 1,000 到 10,000 次写入/秒变化时，Aurora 中的副本延迟从 2.62 毫秒增加到 5.38 毫秒。 相比之下，MySQL 中的副本滞后从不到 1 秒增长到 300 秒。 每秒 10,000 次写入时，Aurora 的副本滞后比 MySQL 小几个数量级。 副本滞后是根据提交的事务在副本中可见所需的时间来衡量的。

#### 6.1.5 Throughput with hot row contention
> In this experiment, we report that Aurora performs very well relative to MySQL on workloads with hot row contention, such as those based on the TPC-C benchmark. We ran the Percona TPC-C variant [37] against Amazon Aurora and MySQL 5.6 and 5.7 on an r3.8xlarge where MySQL uses an EBS volume with 30K provisioned IOPS. Table 5 shows that Aurora can sustain between 2.3x to 16.3x the throughput of MySQL 5.7 as the workload varies from 500 connections and a 10GB data size to 5000 connections and a 100GB data size.

在这个实验中，我们报告说 Aurora 在具有热行争用的工作负载上（例如基于 TPC-C 基准的工作负载）相对于 MySQL 的表现非常好。 我们在 r3.8xlarge 上针对 Amazon Aurora 和 MySQL 5.6 和 5.7 运行 Percona TPC-C 变体 [37]，其中 MySQL 使用具有 30K 预置 IOPS 的 EBS 卷。 表 5 显示 Aurora 可以维持 MySQL 5.7 吞吐量的 2.3 到 16.3 倍，因为工作负载从 500 个连接和 10GB 数据大小到 5000 个连接和 100GB 数据大小不等。

### 6.2 ResultswithRealCustomerWorkloads
> In this section, we share results reported by some of our customers who migrated production workloads from MySQL to Aurora.

在本节中，我们分享了一些将生产工作负载从 MySQL 迁移到 Aurora 的客户报告的结果。

#### 6.2.1 Application response time with Aurora
> An internet gaming company migrated their production service from MySQL to Aurora on an r3.4xlarge instance. The average response time that their web transactions experienced prior to the migration was 15 ms. In contrast, after the migration the average response time 5.5 ms, a 3x improvement as shown in Figure 8.

一家互联网游戏公司将他们的生产服务从 MySQL 迁移到 r3.4xlarge 实例上的 Aurora。 他们的 Web 事务在迁移之前经历的平均响应时间为 15 毫秒。 相比之下，迁移后的平均响应时间为 5.5 毫秒，提高了 3 倍，如图 8 所示。

#### 6.2.2 Statement Latencies with Aurora

> An education technology company whose service helps schools manage student laptops migrated their production workload from MySQL to Aurora. The median (P50) and 95th percentile (P99) latencies for select and per-record insert operations before and after the migration (at 14:00 hours) are shown in Figure 9 and Figure 10.

一家帮助学校管理学生笔记本电脑的教育技术公司将他们的生产工作负载从 MySQL 迁移到 Aurora。 图 9 和图 10 显示了迁移前后（14:00 时）选择和每条记录插入操作的中位数 (P50) 和第 95 个百分位 (P99) 延迟。

> Before the migration, the P95 latencies ranged between 40ms to 80ms and were much worse than the P50 latencies of about 1ms. The application was experiencing the kinds of poor outlier performance that we described earlier in this paper. After the migration, however, the P95 latencies for both operations improved dramatically and approximated the P50 latencies.

在迁移之前，P95 的延迟范围在 40 毫秒到 80 毫秒之间，比大约 1 毫秒的 P50 延迟要差得多。 该应用程序遇到了我们在本文前面描述的各种较差的异常值性能。 然而，在迁移之后，两种操作的 P95 延迟都显着改善，并接近 P50 延迟。

#### 6.2.3 Replica Lag with Multiple Replicas
> MySQL replicas often lag significantly behind their writers and can “can cause strange bugs” as reported by Weiner at Pinterest [40]. For the education technology company described earlier, the replica lag often spiked to 12 minutes and impacted application correctness and so the replica was only useful as a stand by. In contrast, after migrating to Aurora, the maximum replica lag across 4 replicas never exceeded 20ms as shown in Figure 11. The improved replica lag provided by Aurora let the company divert a significant portion of their application load to the replicas saving costs and increasing availability.

正如 Weiner 在 Pinterest [40] 所报告的那样，MySQL 副本通常明显落后于它们的作者，并且可能“导致奇怪的错误”。 对于前面描述的教育技术公司，副本延迟通常会飙升至 12 分钟并影响应用程序的正确性，因此副本仅用作备用。 相比之下，迁移到 Aurora 后，4 个副本的最大副本滞后从未超过 20 毫秒，如图 11 所示。 Aurora 提供的改进的副本滞后让公司将其应用程序负载的很大一部分转移到副本上，从而节省了成本并提高了可用性 .

## 7. LESSONS LEARNED
> We have now seen a large variety of applications run by customers ranging from small internet companies all the way to highly sophisticated organizations operating large numbers of Aurora clusters. While many of their use cases are standard, we focus on scenarios and expectations that are common in the cloud and are leading us to new directions.

我们现在已经看到客户运行的各种应用程序，从小型互联网公司一直到运营大量 Aurora 集群的高度复杂的组织。 虽然他们的许多用例都是标准的，但我们专注于云中常见的场景和期望，并引导我们走向新的方向。

### 7.1 Multi-tenancy and database consolidation

> Many of our customers operate Software-as-a-Service (SaaS) businesses, either exclusively or with some residual on-premise customers they are trying to move to their SaaS model. We find that these customers often rely on an application they cannot easily change. Therefore, they typically consolidate their different customers on a single instance by using a schema/database as a unit of tenancy. This idiom reduces costs: they avoid paying for a dedicated instance per customer when it is unlikely that all of their customers active at once. For instance, some of our SaaS customers report ha ving more than 50,000 customers of their own.

我们的许多客户都经营软件即服务 (SaaS) 业务，他们要么独家经营，要么与一些剩余的本地客户合作，他们正试图转向他们的 SaaS 模式。 我们发现这些客户通常依赖于他们无法轻易更改的应用程序。 因此，他们通常使用架构/数据库作为租用单位，将不同的客户整合到一个实例中。 这个习惯用法降低了成本：当他们的所有客户不太可能同时活跃时，他们避免为每个客户支付一个专用实例。 例如，我们的一些 SaaS 客户报告说他们自己拥有超过 50,000 个客户。

> This model is markedly different from well-known multi-tenant applications like Salesforce.com [14] which use a multi-tenant data model and pack the data of multiple customers into unified tables of a single schema with tenancy identified on a per-row basis. As a result, we see many customers with consolidated databases containing a large number of tables. Production instances of over 150,000 tables for small database are quite common. This puts pressure on components that manage metadata like the dictionary cache. More importantly, such customers need (a) to sustain a high level of throughput and many concurrent user connections, (b) a model where data is only provisioned and paid for as it is used since it is hard to anticipate in advance how much storage space is needed, and (c) reduced jitter so that spikes for a single tenant have minimal impact on other tenants. Aurora supports these attributes and fits such SaaS applications very well.

该模型与众所周知的多租户应用程序（如 Salesforce.com [14]）明显不同，后者使用多租户数据模型并将多个客户的数据打包到单个模式的统一表中，并在每行上标识租户 基础。 因此，我们看到许多客户使用包含大量表的统一数据库。 小型数据库超过 150,000 张表的生产实例非常普遍。 这给管理元数据（如字典缓存）的组件带来了压力。 更重要的是，这些客户需要 (a) 维持高水平的吞吐量和许多并发用户连接，(b) 一种模型，其中数据仅在使用时进行配置和付费，因为很难提前预测有多少存储 需要空间，并且 (c) 减少抖动，以便单个租户的峰值对其他租户的影响最小。 Aurora 支持这些属性并且非常适合此类 SaaS 应用程序。

### 7.2 Highly concurrent auto-scaling workloads
> Internet workloads often need to deal with spikes in traffic based on sudden unexpected events. One of our major customers had a special appearance in a highly popular nationally televised show and experienced one such spike that greatly surpassed their normal peak throughput without stressing the database. To support such spikes, it is important for a database to handle many concurrent connections. This approach is feasible in Aurora since the underlying storage system scales so well. We have several customers that run at over 8000 connections per second.

Internet 工作负载通常需要处理基于突发意外事件的流量高峰。 我们的一个主要客户特别出现在一个非常受欢迎的全国电视节目中，并经历了一个这样的高峰，大大超过了他们正常的峰值吞吐量，而不会给数据库带来压力。 为了支持这种峰值，数据库处理许多并发连接很重要。 这种方法在 Aurora 中是可行的，因为底层存储系统可以很好地扩展。 我们有几个客户每秒运行超过 8000 个连接。

### 7.3 Schema evolution
> Modern web application frameworks such as Ruby on Rails deeply integrate object-relational mapping tools. As a result, it is easy for application developers to make many schema changes to their database making it challenging for DBAs to manage how the schema evolves. In Rails applications, these are called “DB Migrations” and we have heard first-hand accounts of DBAs that have to either deal with a “few dozen migrations a week”, or put in place hedging strategies to ensure that future migrations take place without pain. The situation is exacerbated with MySQL offering liberal schema evolution semantics and implementing most changes using a full table copy. Since frequent DDL is a pragmatic reality, we have implemented an efficient online DDL implementation that (a) versions schemas on a per-page basis and decodes individual pages on demand using their schema history, and (b) lazily upgrades individual pages to the latest schema using a modify-on-write primitive.

Ruby on Rails 等现代 Web 应用程序框架深度集成了对象关系映射工具。 因此，应用程序开发人员很容易对其数据库进行许多架构更改，这使得 DBA 难以管理架构的演变方式。 在 Rails 应用程序中，这些被称为“数据库迁移”，我们听说过 DBA 的第一手资料，他们要么必须处理“每周几十次迁移”，要么制定对冲策略以确保未来的迁移不会发生 疼痛。 MySQL 提供自由模式演化语义并使用全表副本实现大多数更改，从而加剧了这种情况。 由于频繁的 DDL 是一个务实的现实，我们已经实现了一个高效的在线 DDL 实现，它 (a) 在每个页面的基础上版本模式并使用它们的模式历史按需解码单个页面，以及 (b) 延迟将单个页面升级到最新 使用写时修改原语的模式。

### 7.4 Availability and Software Upgrades
> Our customers have demanding expectations of cloud-native databases that can conflict with how we operate the fleet and how often we patch servers. Since our customers use Aurora primarily as an OLTP service backing production applications, any disruption can be traumatic. As a result, many of our customers have a very low tolerance to our updates of database software, even if this amounts to a planned downtime of 30 seconds every 6 weeks or so. Therefore, we recently released a new Zero- Downtime Patch (ZDP) feature that allows us to patch a customer while in-flight database connections are unaffected.

我们的客户对云原生数据库有很高的期望，这可能与我们运营机群的方式以及我们修补服务器的频率相冲突。 由于我们的客户主要将 Aurora 用作支持生产应用程序的 OLTP 服务，因此任何中断都可能造成创伤。 因此，我们的许多客户对我们更新数据库软件的容忍度非常低，即使这相当于每 6 周左右的 30 秒计划停机时间。 因此，我们最近发布了新的零停机补丁 (ZDP) 功能，允许我们在不影响动态数据库连接的情况下为客户打补丁。

> As shown in Figure 12, ZDP works by looking for an instant where there are no active transactions, and in that instant spooling the application state to local ephemeral storage, patching the engine and then reloading the application state. In the process, user sessions remain active and oblivious that the engine changed under the covers.

如图 12 所示，ZDP 的工作方式是寻找没有活动事务的时刻，并在该时刻将应用程序状态假脱机到本地临时存储，修补引擎，然后重新加载应用程序状态。 在此过程中，用户会话保持活跃，并且不会注意到引擎在幕后发生了变化。

## 8. RELATED WORK

> In this section, we discuss other contributions and how they relate to the approaches taken in Aurora.

在本节中，我们将讨论其他贡献以及它们与 Aurora 中采用的方法的关系。

> Decoupling storage from compute. Although traditional systems have usually been built as monolithic daemons [27], there has been recent work on databases that decompose the kernel into different components. For instance, Deuteronomy [10] is one such system that separates a Transaction Component (TC) that provides concurrency control and recovery from a Data Component (DC) that provides access methods on top of LLAMA [34], a latch-free log-structured cache and storage manager. Sinfonia [39] and Hyder [38] are systems that abstract transactional access methods over a scale out service and database systems can be implemented using these abstractions. The Yesquel [36] system implements a multi-version distributed balanced tree and separates concurrency control from the query processor. Aurora decouples storage at a level lower than that of Deuteronomy, Hyder, Sinfonia, and Yesquel. In Aurora, query processing, transactions, concurrency, buffer cache, and access methods are decoupled from logging, storage, and recovery that are implemented as a scale out service.

将存储与计算分离。尽管传统系统通常被构建为单体守护进程 [27]，但最近有一些关于将内核分解为不同组件的数据库的工作。例如，Deuteronomy [10] 就是这样一个系统，它将提供并发控制和恢复的事务组件 (TC) 与提供基于 LLAMA [34] 的访问方法的数据组件 (DC) 分开，这是一种无闩锁的日志-结构化缓存和存储管理器。 Sinfonia [39] 和 Hyder [38] 是通过横向扩展服务抽象事务访问方法的系统，并且可以使用这些抽象来实现数据库系统。 Yesquel [36] 系统实现了多版本分布式平衡树，并将并发控制与查询处理器分离。 Aurora 将存储解耦的级别低于 Deuteronomy、Hyder、Sinfonia 和 Yesquel。在 Aurora 中，查询处理、事务、并发、缓冲区缓存和访问方法与作为横向扩展服务实现的日志记录、存储和恢复分离。

>Distributed Systems. The trade-offs between correctness and availability in the face of partitions have long been known with the major result that one-copy serializability is not possible in the face of network partitions [15]. More recently Brewer’s CAP Theorem as proved in [16] stated that a highly available system cannot provide “strong” consistency guarantees in the presence of network partitions. These results and our experience with cloud- scale complex and correlated failures motivated our consistency goals even in the presence of partitions caused by an AZ failure.

分布式系统。面对分区的正确性和可用性之间的权衡早已为人所知，主要结果是，面对网络分区，单副本可串行化是不可能的 [15]。最近在 [16] 中证明的 Brewer CAP 定理指出，在存在网络分区的情况下，高可用系统无法提供“强”一致性保证。这些结果以及我们在云规模复杂和相关故障方面的经验激发了我们的一致性目标，即使存在由 AZ 故障引起的分区也是如此。

>Bailis et al [12] study the problem of providing Highly Available Transactions (HATs) that neither suffer unavailability during partitions nor incur high network latency. They show that Serializability, Snapshot Isolation and Repeatable Read isolation are not HAT-compliant, while most other isolation levels are achievable with high availability. Aurora provides all these isolation levels by making a simplifying assumption that at any time there is only a single writer generating log updates with LSNs allocated from a single ordered domain.

Bailis 等人 [12] 研究了提供高可用事务 (HAT) 的问题，该事务既不会在分区期间遭受不可用，也不会导致高网络延迟。它们表明可串行化、快照隔离和可重复读隔离不符合 HAT，而大多数其他隔离级别都可以通过高可用性实现。 Aurora 通过简化假设提供所有这些隔离级别，即在任何时候都只有一个写入器生成日志更新，并使用从单个有序域分配的 LSN。

>Google’s Spanner [24] provides externally consistent [25] reads and writes, and globally-consistent reads across the database at a timestamp. These features enable Spanner to support consistent backups, consistent distributed query processing [26], and atomic schema updates, all at global scale, and even in the presence of ongoing transactions. As explained by Bailis [12], Spanner is highly specialized for Google’s read-heavy workload and relies on two-phase commit and two-phase locking for read/write transactions.

Google 的 Spanner [24] 提供外部一致的 [25] 读取和写入，以及以时间戳跨数据库的全局一致读取。这些特性使 Spanner 能够支持一致备份、一致分布式查询处理 [26] 和原子模式更新，所有这些都在全球范围内，甚至在存在正在进行的事务的情况下。正如 Bailis [12] 所解释的那样，Spanner 高度专门用于 Google 的读取密集型工作负载，并且依赖于两阶段提交和两阶段锁定来进行读/写事务。

>Concurrency Control. Weaker consistency (PACELC [17]) and isolation models [18][20] are well known in distributed databases and have led to optimistic replication techniques [19] as well as eventually consistent systems [21][22][23]. Other approaches in centralized systems range from classic pessimistic schemes based on locking [28], optimistic schemes like multi-versioned concurrency control in Hekaton [29], sharded approaches such as VoltDB [30] and Timestamp ordering in HyPer [31][32] and Deuteronomy. Aurora’s storage service provides the database engine the abstraction of a local disk that is durably persisted, and allows the engine to determine isolation and concurrency control.

并发控制。较弱的一致性（PACELC [17]）和隔离模型 [18][20] 在分布式数据库中是众所周知的，并导致了乐观复制技术 [19] 以及最终一致的系统 [21][22][23]。中心化系统中的其他方法包括基于锁定的经典悲观方案 [28]、Hekaton [29] 中的多版本并发控制等乐观方案、VoltDB [30] 等分片方法和 HyPer [31] [32] 中的时间戳排序和申命记。 Aurora 的存储服务为数据库引擎提供了持久持久化的本地磁盘的抽象，并允许引擎确定隔离和并发控制。

>Log-structured storage. Log-structured storage systems were introduced by LFS [33] in 1992. More recently Deuteronomy and the associated work in LLAMA [34] and Bw-Tree [35] use log- structured techniques in multiple ways across the storage engine stack and, like Aurora, reduce write amplification by writing deltas instead of whole pages. Both Deuteronomy and Aurora implement pure redo logging, and keep track of the highest stable LSN for acknowledging commits.

日志结构存储。 LFS [33] 于 1992 年引入了日志结构化存储系统。最近，Deuteronomy 以及 LLAMA [34] 和 Bw-Tree [35] 中的相关工作在整个存储引擎堆栈中以多种方式使用日志结构化技术，例如Aurora，通过写入增量而不是整个页面来减少写入放大。 Deuteronomy 和 Aurora 都实现了纯重做日志记录，并跟踪最高稳定的 LSN 以确认提交。

>Recovery. While traditional databases rely on a recovery protocol based on ARIES [5], some recent systems have chosen other paths for performance. For example, Hekaton and VoltDB rebuild their in-memory state after a crash using some form of an update log. Systems like Sinfonia [39] avoid recovery by using techniques like process pairs and state machine replication. Graefe [41] describes a system with per-page log record chains that enables on-demand page-by-page redo that can make recovery fast. Like Aurora, Deuteronomy does not require redo recovery. This is because Deuteronomy delays transactions so that only committed updates are posted to durable storage. As a result, unlike Aurora, the size of transactions can be constrained in Deuteronomy.

恢复。虽然传统数据库依赖于基于 ARIES [5] 的恢复协议，但最近的一些系统选择了其他路径来提高性能。例如，Hekaton 和 VoltDB 在崩溃后使用某种形式的更新日志重建它们的内存状态。像 Sinfonia [39] 这样的系统通过使用进程对和状态机复制等技术来避免恢复。 Graefe [41] 描述了一个具有每页日志记录链的系统，该系统支持按需逐页重做，从而可以快速恢复。与 Aurora 一样，Deuteronomy 不需要重做恢复。这是因为 Deuteronomy 延迟事务，因此只有已提交的更新才会发布到持久存储。因此，与 Aurora 不同的是，Deuteronomy 中可以限制交易的大小。

## 9. CONCLUSION
> We designed Aurora as a high throughput OLTP database that compromises neither availability nor durability in a cloud-scale environment. The big idea was to move away from the monolithic architecture of traditional databases and decouple storage from compute. In particular, we moved the lower quarter of the database kernel to an independent scalable and distributed service that managed logging and storage. With all I/Os written over the network, our fundamental constraint is now the network. As a result we need to focus on techniques that relieve the network and improve throughput. We rely on quorum models that can handle the complex and correlated failures that occur in large-scale cloud environments and avoid outlier performance penalties, log processing to reduce the aggregate I/O burden, and asynchronous consensus to eliminate chatty and expensive multi-phase synchronization protocols, offline crash recovery, and checkpointing in distributed storage. Our approach has led to a simplified architecture with reduced complexity that is easy to scale as well as a foundation for future advances.

我们将 Aurora 设计为一个高吞吐量的 OLTP 数据库，它既不影响云规模环境中的可用性，也不影响持久性。主要想法是摆脱传统数据库的单体架构，将存储与计算分离。特别是，我们将数据库内核的下四分之一移到了一个独立的可扩展和分布式服务，用于管理日志记录和存储。随着所有 I/O 写入网络，我们的基本约束现在是网络。因此，我们需要专注于减轻网络压力和提高吞吐量的技术。我们依靠仲裁模型来处理大规模云环境中发生的复杂且相关的故障并避免异常性能损失，日志处理以减少聚合 I/O 负担，以及异步共识以消除繁琐和昂贵的多阶段同步协议、离线崩溃恢复和分布式存储中的检查点。我们的方法导致了一个简化的架构，降低了复杂性，易于扩展，并为未来的进步奠定了基础。

## 10. ACKNOWLEDGMENTS
> We thank the entire Aurora development team for their efforts on the project including our current members as well as our distinguished alumni (James Corey, Sam McKelvie, Yan Leshinsky, Lon Lundgren, Pradeep Madhavarapu, and Stefano Stefani). We are particularly grateful to our customers who operate production workloads using our service and have been generous in sharing their experiences and expectations with us. We also thank the shepherds for their invaluable comments in shaping this paper.

## 11. REFERENCES

[1] B. Calder, J. Wang, et al. Windows Azure storage: A highly available cloud storage service with strong consistency. In SOSP 2011.

[2] O. Khan, R. Burns, J. Plank, W. Pierce, and C. Huang. Rethinking erasure codes for cloud file systems: Minimizing I/O for recovery and degraded reads. In FAST 2012.

[3] P .A. Bernstein, V . Hadzilacos, and N. Goodman. Concurrency control and recovery in database systems, Chapter 7, Addison Wesley Publishing Company, ISBN 0- 201-10715-5, 1997.

[4] C. Mohan, B. Lindsay, and R. Obermarck. Transaction management in the R* distributed database management system”. ACM TODS, 11(4):378-396, 1986.

[5] C. Mohan and B. Lindsay. Efficient commit protocols for the tree of processes model of distributed transactions. ACM SIGOPS Operating Systems Review, 19(2):40-52, 1985.

[6] D.K. Gifford. Weighted voting for replicated data. In SOSP 1979.

[7] C. Mohan, D.L. Haderle, B. Lindsay, H. Pirahesh, and P.Schwarz. ARIES: A transaction recovery method supporting fine-granularity locking and partial rollbacks using write-ahead logging. ACM TODS, 17 (1): 94–162, 1992

[8] R. van Renesse and F. Schneider. Chain replication for supporting high throughput and availability. In OSDI 2004.

[9] A. Kopytov. Sysbench Manual. Available at http://imysql.com/wp-content/uploads/2014/10/sysbench-manual.pdf

[10] J. Levandoski, D. Lomet, S. Sengupta, R. Stutsman, and R.Wang. High performance transactions in deuteronomy. In CIDR 2015.

[11] P. Bailis, A. Fekete, A. Ghodsi, J.M. Hellerstein, and I.Stoica. Scalable atomic visibility with RAMP Transactions. In SIGMOD 2014

[12] P. Bailis, A. Davidson, A. Fekete, A. Ghodsi, J.M.Hellerstein, and I. Stoica. Highly available transactions: virtues and limitations. In VLDB 2014.

[13] R. Taft, E. Mansour, M. Serafini, J. Duggan, A.J. Elmore, A.Aboulnaga, A. Pavlo, and M. Stonebraker. E-Store: fine-grained elastic partitioning for distributed transaction processing systems. In VLDB 2015.

[14] R. Woollen. The internal design of salesforce.com’s multi-tenant architecture. In SoCC 2010.

[15] S. Davidson, H. Garcia-Molina, and D. Skeen. Consistency in partitioned networks. ACM CSUR, 17(3):341–370, 1985.

[16] S. Gilbert and N. Lynch. Brewer’s conjecture and the feasibility of consistent, available, partition-tolerant web services. SIGACT News, 33(2):51–59, 2002.

[17] D.J. Abadi. Consistency tradeoffs in modern distributed database system design: CAP is only part of the story. IEEE Computer, 45(2), 2012.

[18] A. Adya. Weak consistency: a generalized theory and optimistic implementations for distributed transactions. PhD Thesis, MIT, 1999.

[19] Y. Saito and M. Shapiro. Optimistic replication. ACM Comput. Surv., 37(1), Mar. 2005.

[20] H. Berenson, P. Bernstein, J. Gray, J. Melton, E. O’Neil, and P. O’Neil. A critique of ANSI SQL isolation levels. In SIGMOD 1995.

[21] P. Bailis and A. Ghodsi. Eventual consistency today: limitations, extensions, and beyond. ACM Queue, 11(3), March 2013.

[22] P. Bernstein and S. Das. Rethinking eventual consistency. In SIGMOD, 2013.

[23] B. Cooper et al. PNUTS: Yahoo!’s hosted data serving platform. In VLDB 2008.

[24] J. C. Corbett, J. Dean, et al. Spanner: Google’s globally- distributed database. In OSDI 2012.

[25] David K. Gifford. Information Storage in a Decentralized Computer System. Tech. rep. CSL-81-8. PhD dissertation. Xerox PARC, July 1982.

[26] Jeffrey Dean and Sanjay Ghemawat. MapReduce: a flexible data processing tool”. CACM 53 (1):72-77, 2010.

[27] J. M. Hellerstein, M. Stonebraker, and J. R. Hamilton. Architecture of a database system. Foundations and Trends in Databases. 1(2) pp. 141-259, 2007.

[28] J.Gray,R.A.Lorie,G.R.Putzolu,I.L.Traiger.Granularity of locks in a shared data base. In VLDB 1975.

[29] P-A Larson, et al. High-Performance Concurrency control mechanisms for main-memory databases. PVLDB, 5(4): 298- 309, 2011.

[30] M. Stonebraker and A. Weisberg. The VoltDB main memory DBMS. IEEE Data Eng. Bull., 36(2): 21-27, 2013.

[31] V. Leis, A. Kemper, et al. Exploiting hardware transactional memory in main-memory databases. In ICDE 2014.

[32] H. Mühe, S. Wolf, A. Kemper, and T. Neumann: An evaluation of strict timestamp ordering concurrency control for main-memory database systems. In IMDM 2013.

[33] M. Rosenblum and J. Ousterhout. The design and implementation of a log-structured file system. ACM TOCS 10(1): 26–52, 1992.

[34] J. Levandoski, D. Lomet, S. Sengupta. LLAMA: A cache/storage subsystem for modern hardware. PVLDB 6(10): 877-888, 2013.

[35] J. Levandoski, D. Lomet, and S. Sengupta. The Bw-Tree: A B-tree for new hardware platforms. In ICDE 2013.

[36] M. Aguilera, J. Leners, and M. Walfish. Yesquel: scalable SQL storage for web applications. In SOSP 2015.

[37] Percona Lab. TPC-C Benchmark over MySQL. Available at https://github.com/Percona-Lab/tpcc-mysql

[38] P. Bernstein, C. Reid, and S. Das. Hyder – A transactional record manager for shared flash. In CIDR 2011.

[39] M. Aguilera, A. Merchant, M. Shah, A. Veitch, and C. Karamanolis. Sinfonia: A new paradigm for building scalable distributed systems. ACM Trans. Comput. Syst. 27(3): 2009.

[40] M. Weiner. Sharding Pinterest: How we scaled our MySQL fleet. Pinterest Engineering Blog. Available at: https://engineering.pinterest.com/blog/sharding-pinterest- how-we-scaled-our-mysql-fleet

[41] G. Graefe. Instant recovery for data center savings. ACM SIGMOD Record. 44(2):29-34, 2015.

[42] J. Dean and L. Barroso. The tail at scale. CACM 56(2):74- 80, 2013.