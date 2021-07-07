# 基于CRAQ的对象存储

## Object Storage on CRAQ
## High-throughput chain replication for read-mostly workloads

---

Translate by [Jeff Zhao](mailto:zhaozhuo112@gmail.com)

[Original Paper](https://dl.acm.org/doi/10.1145/2882903.2903741)

---

Jeff Terrace and Michael J. Freedman

Princeton University

## 摘要

> Massive storage systems typically replicate and partition data over many potentially-faulty components to provide both reliability and scalability. Yet many commercially deployed systems, especially those designed for interactive use by customers, sacrifice stronger consistency properties in the desire for greater availability and higher throughput.

海量存储系统通常在许多可能出现故障的组件上复制和分区数据，以提供可靠性和可扩展性。 然而，许多商业部署的系统，尤其是那些为客户交互使用而设计的系统，为了获得更高的可用性和更高的吞吐量而牺牲了更强的一致性属性。

> This paper describes the design, implementation, and evaluation of CRAQ, a distributed object-storage system that challenges this inflexible tradeoff. Our basic approach, an improvement on Chain Replication, maintains strong consistency while greatly improving read throughput. By distributing load across all object replicas, CRAQ scales linearly with chain size without increasing consistency coordination. At the same time, it exposes noncommitted operations for weaker consistency guarantees when this suffices for some applications, which is especially useful under periods of high system churn. This paper explores additional design and implementation considerations for geo-replicated CRAQ storage across multiple datacenters to provide locality-optimized operations. We also discuss multi-object atomic updates and multicast optimizations for large-object updates.

本文描述了 CRAQ 的设计、实现和评估，这是一个分布式对象存储系统，挑战了这种不灵活的权衡。 我们的基本方法是对链复制的改进，在保持强一致性的同时大大提高了读取吞吐量。 通过在所有对象副本之间分配负载，CRAQ 随链大小线性扩展，而不会增加一致性协调。 同时，当这对于某些应用程序足够时，它会公开未提交的操作以提供较弱的一致性保证，这在系统高波动时期尤其有用。 本文探讨了跨多个数据中心的地理复制 CRAQ 存储的其他设计和实施注意事项，以提供局部优化操作。 我们还讨论了大对象更新的多对象原子更新和多播优化。

## 1 介绍

> Many online services require object-based storage, where data is presented to applications as entire units. Object stores support two basic primitives: read (or query) operations return the data block stored under an object name, and write (or update) operations change the state of a single object. Such object-based storage is supported by key-value databases (e.g., BerkeleyDB [40] or Apache’s semi-structured CouchDB [13]) to the massively-scalable systems being deployed in commercial datacenters (e.g., Amazon’s Dynamo [15], Facebook’s Cassandra [16], and the popular Memcached [18]). To achieve the requisite reliability, load balancing, and scalability in many of these systems, the object namespace is partitioned over many machines and each data object is replicated several times.

许多在线服务需要基于对象的存储，其中数据作为一个整体呈现给应用程序。 对象存储支持两种基本原语：读取（或查询）操作返回存储在对象名称下的数据块，写入（或更新）操作更改单个对象的状态。 这种基于对象的存储由键值数据库（例如 BerkeleyDB [40] 或 Apache 的半结构化 CouchDB [13]）支持到部署在商业数据中心的大规模可扩展系统（例如，亚马逊的 Dynamo [15]、Facebook 的 Cassandra [16] 和流行的 Memcached [18]）。 为了在许多这些系统中实现必要的可靠性、负载平衡和可伸缩性，对象命名空间被划分到多台机器上，并且每个数据对象被复制多次。

> Object-based systems are more attractive than their filesystem counterparts when applications have certain requirements. Object stores are better suited for flat namespaces, such as in key-value databases, as opposed to hierarchical directory structures. Object stores simplify the process of supporting whole-object modifications. And, they typically only need to reason about the ordering of modifications to a specific object, as opposed to the entire storage system; it is significantly cheaper to provide consistency guarantees per object instead of across all operations and/or objects.

当应用程序有特定要求时，基于对象的系统比文件系统更有吸引力。 对象存储更适合平面命名空间，例如在键值数据库中，而不是分层目录结构。 对象存储简化了支持整个对象修改的过程。 而且，他们通常只需要推理对特定对象的修改顺序，而不是整个存储系统； 为每个对象而不是跨所有操作和/或对象提供一致性保证要便宜得多。

> When building storage systems that underlie their myriad applications, commercial sites place the need for high performance and availability at the forefront. Data is replicated to withstand the failure of individual nodes or even entire datacenters, whether from planned maintenance or unplanned failure. Indeed, the news media is rife with examples of datacenters going offline, taking down entire websites in the process [26]. This strong focus on availability and performance—especially as such properties are being codified in tight SLA requirements [4, 24]— has caused many commercial systems to sacrifice strong consistency semantics due to their perceived costs (as at Google [22], Amazon [15], eBay [46], and Facebook [44], among others).

在构建作为无数应用程序基础的存储系统时，商业站点将高性能和可用性需求放在首位。 数据被复制以承受单个节点甚至整个数据中心的故障，无论是计划内的维护还是计划外的故障。 事实上，新闻媒体上充斥着数据中心离线、在此过程中关闭整个网站的例子 [26]。 这种对可用性和性能的强烈关注——尤其是在严格的 SLA 要求 [4, 24] 中将此类属性编入规范时——已经导致许多商业系统因其感知成本而牺牲了强一致性语义（如谷歌 [22]、 亚马逊 [15]、eBay [46] 和 Facebook [44] 等）。

> Recently, van Renesse and Schneider presented a chain replication method for object storage [47] over fail-stop servers, designed to provide strong consistency yet improve throughput. The basic approach organizes all nodes storing an object in a chain, where the chain tail handles all read requests, and the chain head handles all write requests. Writes propagate down the chain before the client is acknowledged, thus providing a simple ordering of all object operations—and hence strong consistency—at the tail. The lack of any complex or multi-round protocols yields simplicity, good throughput, and easy recovery.

最近，van Renesse 和 Schneider 提出了一种基于故障停止服务器的对象存储链复制方法 [47]，旨在提供强一致性并提高吞吐量。 基本方法将存储一个对象的所有节点组织成一个链，其中链尾处理所有读取请求，链头处理所有写入请求。 写入在客户端被确认之前沿链向下传播，从而在尾部提供所有对象操作的简单排序，从而提供强一致性。 没有任何复杂的或多轮协议产生简单性、良好的吞吐量和容易恢复。

> Unfortunately, the basic chain replication approach has some limitations. All reads for an object must go to the same node, leading to potential hotspots. Multiple chains can be constructed across a cluster of nodes for better load balancing—via consistent hashing [29] or a more centralized directory approach [22]—but these algorithms might still find load imbalances if particular objects are disproportionally popular, a real issue in practice [17]. Perhaps an even more serious issue arises when attempting to build chains across multiple datacenters, as all reads to a chain may then be handled by a potentially-distant node (the chain’s tail).

不幸的是，基本的链式复制方法有一些局限性。 对一个对象的所有读取都必须转到同一个节点，从而导致潜在的热点。 可以跨节点集群构建多个链以实现更好的负载平衡——通过一致的散列 [29] 或更集中的目录方法 [22]——但是如果特定对象不成比例地流行，这些算法可能仍然会发现负载不平衡，这是一个真正的问题 实践 [17]。 在尝试跨多个数据中心构建链时，可能会出现更严重的问题，因为对链的所有读取都可能由潜在的远处节点（链的尾部）处理。

> This paper presents the design, implementation, and evaluation of CRAQ (Chain Replication with Apportioned Queries), an object storage system that, while maintaining the strong consistency properties of chain replication [47], provides lower latency and higher throughput for read operations by supporting apportioned queries: that is, dividing read operations over all nodes in a chain, as opposed to requiring that they all be handled by a single primary node. This paper’s main contributions are the following.

本文介绍了 CRAQ（带分配查询的链复制）的设计、实现和评估，这是一个对象存储系统，在保持链复制 [47] 的强一致性属性的同时，通过支持为读取操作提供更低的延迟和更高的吞吐量 分摊查询：即，将读取操作划分到链中的所有节点，而不是要求它们都由单个主节点处理。 本文的主要贡献如下。

> 1.CRAQ enables any chain node to handle read operations while preserving strong consistency, thus supporting load balancing across all nodes storing an object. Furthermore, when workloads are read mostly—an assumption used in other systems such as the Google File System [22] and Memcached [18]—the performance of CRAQ rivals systems offering only eventual consistency.

1.CRAQ 使任何链节点都可以在保持强一致性的同时处理读操作，从而支持跨存储对象的所有节点的负载均衡。 此外，当大部分工作负载被读取时——谷歌文件系统 [22] 和 Memcached [18] 等其他系统中使用的假设——CRAQ 的性能可与仅提供最终一致性的系统相媲美。

> 2.In addition to strong consistency, CRAQ’s design naturally supports eventual-consistency among read operations for lower-latency reads during write contention and degradation to read-only behavior during transient partitions. CRAQ allows applications to specify the maximum staleness acceptable for read operations.

2. 除了强一致性之外，CRAQ 的设计自然支持读操作之间的最终一致性，以实现写争用期间的低延迟读取和瞬态分区期间降级为只读行为。 CRAQ 允许应用程序指定读取操作可接受的最大失效时间。

> 3.Leveraging these load-balancing properties, we describe a wide-area system design for building CRAQ chains across geographically-diverse clusters that preserves strong locality properties. Specifically, reads can be handled either completely by a local cluster, or at worst, require concise metadata information to be transmitted across the wide-area during times of high write contention. We also present our use of ZooKeeper [48], a PAXOS-like group membership system, to manage these deployments.

3.利用这些负载平衡特性，我们描述了一种广域系统设计，用于跨地理不同的集群构建 CRAQ 链，保留强大的局部性。 具体来说，读取可以完全由本地集群处理，或者在最坏的情况下，需要在高写入争用期间跨广域传输简明的元数据信息。 我们还展示了我们使用 ZooKeeper [48]（一个类似于 PAXOS 的组成员系统）来管理这些部署。

> Finally, we discuss additional extensions to CRAQ, including the integration of mini-transactions for multiobject atomic updates, and the use of multicast to improve write performance for large-object updates. We have not yet finished implementing these optimizations, however.

最后，我们讨论了 CRAQ 的其他扩展，包括集成多对象原子更新的小事务，以及使用多播来提高大对象更新的写入性能。 然而，我们还没有完成这些优化。

> A preliminary performance evaluation of CRAQ demonstrates its high throughput compared to the basic chain replication approach, scaling linearly with the number of chain nodes for read-mostly workloads: approximately a 200% improvement for three-node chains, and 600% for seven-node chains. During high write contention, CRAQ’s read throughput in three-node chains still outperformed chain replication by a factor of two, and read latency remains low. We characterize its performance under varying workloads and under failures. Finally, we evaluate CRAQ’s performance for geo-replicated storage, demonstrating significantly lower latency than that achieved by basic chain replication.

CRAQ 的初步性能评估表明，与基本链复制方法相比，它具有高吞吐量，对于以读取为主的工作负载，随着链节点的数量线性扩展：三节点链提高约 200%，七节点提高约 600% 链。 在高写入争用期间，CRAQ 在三节点链中的读取吞吐量仍然比链复制高出两倍，并且读取延迟仍然很低。 我们描述了其在不同工作负载和故障情况下的性能。 最后，我们评估了 CRAQ 在异地复制存储方面的性能，证明其延迟显着低于基本链复制实现的延迟。

> The remainder of this paper is organized as follows. Section §2 provides a comparison between the basic chain replication and CRAQ protocols, as well as CRAQ’s support for eventual consistency. Section §3 describes scaling out CRAQ to many chains, within and across datacenters, as well as the group membership service that manages chains and nodes. Section §4 touches on extensions such as multi-object updates and leveraging multicast. Section §5 describes our CRAQ implementation, §6 presents our performance evaluation, §7 reviews related work, and §8 concludes.

本文的其余部分安排如下。 第 2 节提供了基本链复制和 CRAQ 协议之间的比较，以及 CRAQ 对最终一致性的支持。 第 3 节描述了将 CRAQ 扩展到数据中心内和跨数据中心的多个链，以及管理链和节点的组成员服务。 第 4 节涉及扩展，例如多对象更新和利用多播。 第 5 节描述了我们的 CRAQ 实施，第 6 节介绍了我们的绩效评估，第 7 节回顾了相关工作，第 8 节总结了。

## 2 系统基本模型

> This section introduces our object-based interface and consistency models, provides a brief overview of the standard Chain Replication model, and then presents stronglyconsistent CRAQ and its weaker variants.

本节介绍我们基于对象的接口和一致性模型，简要概述标准链复制模型，然后介绍强一致性 CRAQ 及其较弱的变体。

### 2.1 Interface and Consistency Model

> An object-based storage system provides two simple primitives for users:

基于对象的存储系统为用户提供了两个简单的原语：

> **write(objID, V)**: The write (update) operation stores the value V associated with object identifier ob jID.

**write(objID, V)**：写入（更新）操作存储与对象标识符 objID 关联的值 V。

> **V ← read(objID)**: The read (query) operation retrieves the value V associated with object id ob jID.

**V ← read(objID)**：读取（查询）操作检索与对象 id objID 关联的值 V。

> We will be discussing two main types of consistency, taken with respect to individual objects.

我们将讨论关于单个对象的两种主要类型的一致性。

> **Strong Consistency** in our system provides the guarantee that all read and write operations to an object are executed in some sequential order, and that a read to an object always sees the latest written value.

**强一致性** 在我们的系统中提供了对一个对象的所有读写操作都以某种顺序执行的保证，并且对一个对象的读取总是看到最新的写入值。

> **Eventual Consistency** in our system implies that writes to an object are still applied in a sequential order on all nodes, but eventually-consistent reads to different nodes can return stale data for some period of inconsistency (i.e., before writes are applied on all nodes). Once all replicas receive the write, however, read operations will never return an older version than this latest committed write. In fact, a client will also see monotonic read consistency (That is, informally, successive reads to an object will return either the same prior value or a more recent one, but never an older value.) if it maintains a session with a particular node (although not across sessions with different nodes).

**最终一致性** 在我们的系统中意味着对对象的写入仍然在所有节点上按顺序应用，但对不同节点的最终一致性读取可能会在一段时间内返回陈旧数据（即，在应用写入之前 在所有节点上）。 但是，一旦所有副本都收到写入，读取操作将永远不会返回比此最新提交的写入更旧的版本。 事实上，客户端也会看到单调读取一致性（即，非正式地，对对象的连续读取将返回相同的先前值或更新的值，但不会返回旧值。）如果它与特定的会话保持会话 节点（虽然不是跨不同节点的会话）。

> We next consider how Chain Replication and CRAQ provide their strong consistency guarantees.

接下来我们考虑 Chain Replication 和 CRAQ 如何提供强一致性保证。

### 2.2 Chain Replication

> Chain Replication (CR) is a method for replicating data across multiple nodes that provides a strongly consistent storage interface. Nodes form a chain of some defined length C. The head of the chain handles all write operations from clients. When a write operation is received by a node, it is propagated to the next node in the chain. Once the write reaches the tail node, it has been applied to all replicas in the chain, and it is considered committed. The tail node handles all read operations, so only values which are committed can be returned by a read.

链复制 (CR) 是一种跨多个节点复制数据的方法，它提供了高度一致的存储接口。 节点形成一个定义长度为 C 的链。链的头部处理来自客户端的所有写操作。 当一个节点收到一个写操作时，它被传播到链中的下一个节点。 一旦写入到达尾节点，它就已被应用到链中的所有副本，并被视为已提交。 尾节点处理所有读取操作，因此读取只能返回提交的值。

> Figure 1 provides an example chain of length four. All read requests arrive and are processed at the tail. Write requests arrive at the head of the chain and propagate their way down to the tail. When the tail commits the write, a reply is sent to the client. The CR paper describes the tail sending a message directly back to the client; because we use TCP, our implementation actually has the head respond after it receives an acknowledgment from the tail, given its pre-existing network connection with the client. This acknowledgment propagation is shown with the dashed line in the figure.

图 1 提供了一个长度为 4 的示例链。 所有读取请求到达并在尾部处理。 写请求到达链的头部并向下传播到尾部。 当尾部提交写入时，会向客户端发送回复。 CR论文描述了tail直接向客户端发送消息； 因为我们使用 TCP，我们的实现实际上在收到来自尾部的确认后，头部响应，考虑到它与客户端的预先存在的网络连接。 这种确认传播在图中用虚线表示。

> The simple topology of CR makes write operations cheaper than in other protocols offering strong consistency. Multiple concurrent writes can be pipelined down the chain, with transmission costs equally spread over all nodes. The simulation results of previous work [47] showed competitive or superior throughput for CR compared to primary/backup replication, while arguing a principle advantage from quicker and easier recovery.

CR 的简单拓扑使得写操作比其他提供强一致性的协议更便宜。 多个并发写入可以通过流水线向下传输，传输成本平均分布在所有节点上。 先前工作 [47] 的模拟结果显示，与主/备份复制相比，CR 具有竞争力或更高的吞吐量，同时认为恢复更快、更容易的主要优势。

> Chain replication achieves strong consistency: As all reads go to the tail, and all writes are committed only when they reach the tail, the chain tail can trivially apply a total ordering over all operations. This does come at a cost, however, as it reduces read throughput to that of a single node, instead of being able to scale out with chain size. But it is necessary, as querying intermediate nodes could otherwise violate the strong consistency guarantee; specifically, concurrent reads to different nodes could see different writes as they are in the process of propagating down the chain.

链式复制实现了强一致性：由于所有读取都到达尾部，并且所有写入仅在到达尾部时才提交，因此链尾部可以轻松地对所有操作应用总排序。 然而，这确实是有代价的，因为它将读取吞吐量降低到单个节点的读取吞吐量，而不是能够随着链的大小扩展。 但这是必要的，因为查询中间节点可能会违反强一致性保证； 具体来说，对不同节点的并发读取可能会看到不同的写入，因为它们在沿链传播的过程中。

> While CR focused on providing a storage service, one could also view its query/update protocols as an interface to replicated state machines (albeit ones that affect distinct object). One can view CRAQ in a similar light, although the remainder of this paper considers the problem only from the perspective of a read/write (also referred to as a get/put or query/update) object storage interface.

虽然 CR 专注于提供存储服务，但人们也可以将其查询/更新协议视为复制状态机（尽管影响不同对象的状态机）的接口。 可以从类似的角度看待 CRAQ，尽管本文的其余部分仅从读/写（也称为获取/放置或查询/更新）对象存储接口的角度考虑问题。

### 2.2 Chain Replication with Apportioned Queries

> Motivated by the popularity of read-mostly workload environments, CRAQ seeks to increase read throughput by allowing any node in the chain to handle read operations while still providing strong consistency guarantees. The main CRAQ extensions are as follows.

受以读取为主的工作负载环境流行的推动，CRAQ 试图通过允许链中的任何节点处理读取操作，同时仍然提供强大的一致性保证来提高读取吞吐量。 主要的 CRAQ 扩展如下。

> 1.A node in CRAQ can store multiple versions of an object, each including a monotonically-increasing version number and an additional attribute whether the version is clean or dirty. All versions are initially marked as clean.

1. CRAQ 中的一个节点可以存储一个对象的多个版本，每个版本包括一个单调递增的版本号和一个附加的属性，无论版本是干净的还是脏的。 所有版本最初都标记为干净。

> 2.When a node receives a new version of an object (via a write being propagated down the chain), the node appends this latest version to its list for the object.

2.当一个节点收到一个对象的新版本时（通过沿着链向下传播的写入），该节点将这个最新版本附加到它的对象列表中。

> If the node is not the tail,it marks the version as dirty, and propagates the write to its successor.

如果节点不是尾部，它将版本标记为脏，并将写入传播到其后继节点。

> Otherwise, if the node is the tail, it marks the version as clean, at which time we call the object version (write) as committed. The tail node can then notify all other nodes of the commit by sending an acknowledgement backwards through the chain.

否则，如果节点是尾部，则将版本标记为干净，此时我们将对象版本（写入）称为已提交。 然后，尾节点可以通过链向后发送确认来通知所有其他节点提交。

> 3.When an acknowledgment message for an object version arrives at a node, the node marks the object version as clean. The node can then delete all prior versions of the object.

3.当对象版本的确认消息到达节点时，节点将对象版本标记为干净。 然后该节点可以删除该对象的所有先前版本。

> 4.When a node receives a read request for an object: If the latest known version of the requested object is clean, the node returns this value. Otherwise, if the latest version number of the object requested is dirty, the node contacts the tail and asks for the tail’s last committed version number (a version query). The node then returns that version of the object; by construction, the node is guaranteed to be storing this version of the object. We note that although the tail could commit a new version between when it replied to the version request and when the intermediate node sends a reply to the client, this does not violate our definition of strong consistency, as read operations are serialized with respect to the tail.

4.当一个节点收到一个对象的读请求时：如果请求对象的最新已知版本是干净的，节点返回这个值。 否则，如果请求的对象的最新版本号是脏的，则节点联系尾部并询问尾部最后提交的版本号（版本查询）。 然后节点返回对象的那个版本； 通过构造，节点保证存储这个版本的对象。 我们注意到，虽然尾部可以在回复版本请求和中间节点向客户端发送回复之间提交新版本，但这并不违反我们对强一致性的定义，因为读取操作是相对于 尾巴。

> Note that an object’s “dirty” or “clean” state at a node can also be determined implicitly, provided a node deletes old versions as soon as it receives a write commitment acknowledgment. Namely, if the node has exactly one version for an object, the object is implicitly in the clean state; otherwise, the object is dirty and the properlyordered version must be retrieved from the chain tail.

请注意，如果节点在收到写入承诺确认后立即删除旧版本，也可以隐式确定节点上对象的“脏”或“干净”状态。 即，如果节点只有一个对象版本，则该对象隐式处于干净状态； 否则，对象是脏的，必须从链尾检索正确排序的版本。

> Figure 2 shows a CRAQ chain in the starting clean state. Each node stores an identical copy of an object, so any read request arriving at any node in the chain will return the same value. All nodes remain in the clean state unless a write operation is received.

图 2 显示了处于起始清洁状态的 CRAQ 链。 每个节点存储一个对象的相同副本，因此到达链中任何节点的任何读取请求都将返回相同的值。 除非收到写操作，否则所有节点都保持清洁状态。

> In Figure 3, we show a write operation in the middle of propagation (shown by the dashed purple line). The head node received the initial message to write a new version (V2) of the object, so the head’s object is dirty. It then propagated the write message down the chain to the second node, which also marked itself as dirty for that object (having multiple versions [V1,V2] for a single object ID K). If a read request is received by one of the clean nodes, they immediately return the old version of the object: This is correct, as the new version has yet to be committed at the tail. If a read request is received by either of the dirty nodes, however, they send a version query to the tail— shown in the figure by the dotted blue arrow—which returns its known version number for the requested object (1). The dirty node then returns the old object value (V1) associated with this specified version number. Therefore, all nodes in the chain will still return the same version of an object, even in the face of multiple outstanding writes being propagated down the chain.

在图 3 中，我们展示了传播中间的写操作（由紫色虚线显示）。头节点收到初始消息写入对象的新版本（V2），因此头的对象是脏的。然后它将写消息沿链传播到第二个节点，第二个节点也将自己标记为该对象的脏（对于单个对象 ID K 具有多个版本 [V1,V2]）。如果其中一个干净节点收到读取请求，它们会立即返回对象的旧版本：这是正确的，因为新版本尚未在尾部提交。但是，如果脏节点中的任何一个收到读取请求，它们都会向尾部发送一个版本查询（如图中蓝色虚线箭头所示），它返回所请求对象的已知版本号 (1)。然后，脏节点返回与此指定版本号相关联的旧对象值 (V1)。因此，链中的所有节点仍将返回对象的相同版本，即使面对沿链传播的多个未完成的写入。

> When the tail receives and accepts the write request, it sends an acknowledgment message containing this write’s version number back up the chain. As each predecessor receives the acknowledgment, it marks the specified version as clean (possibly deleting all older versions). When its latest-known version becomes clean, it can subsequently handle reads locally. This method leverages the fact that writes are all propagated serially, so the tail is always the last chain node to receive a write.

当尾部接收并接受写入请求时，它会向链上发送包含此写入版本号的确认消息。 当每个前任收到确认时，它将指定版本标记为干净（可能删除所有旧版本）。 当它的最新已知版本变得干净时，它可以随后在本地处理读取。 这种方法利用了写入都是串行传播的事实，因此尾部始终是接收写入的最后一个链节点。

> CRAQ’s throughput improvements over CR arise in two different scenarios:

CRAQ 相对于 CR 的吞吐量改进出现在两种不同的情况下：

> • Read-Mostly Workloads have most of the read requests handled solely by the C − 1 non-tail nodes (as clean reads), and thus throughput in these scenarios scales linearly with chain size C.

- 以读取为主的工作负载的大部分读取请求仅由 C-1 个非尾节点（作为干净读取）处理，因此这些场景中的吞吐量与链大小 C 呈线性关系。

> • Write-Heavy Workloads have most read requests to non-tail nodes as dirty, thus require version queries to the tail. We suggest, however, that these version queries are lighter-weight than full reads, allowing the tail to process them at a much higher rate before it becomes saturated. This leads to a total read throughput that is still higher than CR.

- Write-Heavy Workloads 对非尾节点的大多数读取请求是脏的，因此需要对尾节点进行版本查询。 然而，我们建议这些版本查询比完整读取更轻，允许尾部在饱和之前以更高的速度处理它们。 这导致总读取吞吐量仍高于 CR。

> Performance results in §6 support both of these claims, even for small objects. For longer chains that are persistently write-heavy, one could imagine optimizing read throughput by having the tail node only handle version queries, not full read requests, although we do not evaluate this optimization.

§6 中的性能结果支持这两种说法，即使是小物体。 对于持续大量写入的较长链，可以想象通过让尾节点仅处理版本查询而不是完整读取请求来优化读取吞吐量，尽管我们没有评估这种优化。

### 2.4 Consistency Models on CRAQ

> Some applications may be able to function with weaker consistency guarantees, and they may seek to avoid the performance overhead of version queries (which can be significant in wide-area deployments, per §3.3), or they may wish to continue to function at times when the system cannot offer strong consistency (e.g., during partitions). To support such variability in requirements, CRAQ simultaneously supports three different consistency models for reads. A read operation is annotated with which type of consistency is permissive.

某些应用程序可能能够在较弱的一致性保证下运行，并且它们可能会寻求避免版本查询的性能开销（这在广域部署中可能很重要，根据 §3.3），或者它们有时可能希望继续运行 当系统不能提供强一致性时（例如，在分区期间）。 为了支持需求的这种可变性，CRAQ 同时支持三种不同的读取一致性模型。 读取操作被注释为允许哪种类型的一致性。

> • Strong Consistency (the default) is described in the model above (§2.1). All object reads are guaranteed to be consistent with the last committed write.

- 强一致性（默认）在上面的模型（第 2.1 节）中进行了描述。 所有对象读取都保证与上次提交的写入一致。

> • Eventual Consistency allows read operations to a chain node to return the newest object version known to it. Thus, a subsequent read operation to a different node may return an object version older than the one previously returned. This does not, therefore, satisfy monotonic read consistency, although reads to a single chain node do maintain this property locally (i.e., as part of a session).

- 最终一致性允许对链节点的读取操作返回其已知的最新对象版本。 因此，对不同节点的后续读取操作可能会返回比先前返回的对象版本更旧的对象版本。 因此，这并不满足单调读取一致性，尽管对单个链节点的读取确实在本地维护此属性（即作为会话的一部分）。

> • Eventual Consistency with Maximum-Bounded Inconsistency allows read operations to return newly written objects before they commit, but only to a certain point. The limit imposed can be based on time (relative to a node’s local clock) or on absolute version numbers. In this model, a value returned from a read operation is guaranteed to have a maximum inconsistency period (defined over time or versioning). If the chain is still available, this inconsistency is actually in terms of the returned version being newer than the last committed one. If the system is partitioned and the node cannot participate in writes, the version may be older than the current committed one.

- 具有最大边界不一致的最终一致性允许读取操作在提交之前返回新写入的对象，但仅限于某个点。 施加的限制可以基于时间（相对于节点的本地时钟）或绝对版本号。 在此模型中，从读取操作返回的值保证具有最大不一致期（随时间或版本控制定义）。 如果链仍然可用，这种不一致实际上是因为返回的版本比最后提交的版本更新。 如果系统已分区且节点无法参与写入，则版本可能比当前提交的版本旧。

### 2.5 Failure Recovery in CRAQ

> As the basic structure of CRAQ is similar to CR, CRAQ uses the same techniques to recover from failure. Informally, each chain node needs to know its predecessor and successor, as well as the chain head and tail. When a head fails, its immediate successor takes over as the new chain head; likewise, the tail’s predecessor takes over when the tail fails. Nodes joining or failing from within the middle of the chain must insert themselves between two nodes, much like a doubly-linked list. The proofs of correctness for dealing with system failures are similar to CR; we avoid them here due to space limitations. Section §5 describes the details of failure recovery in CRAQ, as well as the integration of our coordination service. In particular, CRAQ’s choice of allowing a node to join anywhere in a chain (as opposed only to at its tail [47]), as well as properly handling failures during recovery, requires some careful consideration.

由于 CRAQ 的基本结构与 CR 相似，因此 CRAQ 使用相同的技术从故障中恢复。 非正式地，每个链节点都需要知道其前驱和后继，以及链头和尾。 当一个头出现故障时，它的直接继任者接替新的链头； 同样，当尾部出现故障时，尾部的前任将接管。 从链中间加入或失败的节点必须将自己插入两个节点之间，就像双向链表一样。 处理系统故障的正确性证明类似于CR； 由于空间限制，我们在这里避免使用它们。 第 5 节描述了 CRAQ 中故障恢复的细节，以及我们协调服务的集成。 特别是，CRAQ 选择允许节点加入链中的任何位置（而不是仅在其尾部 [47]），以及在恢复期间正确处理故障，需要仔细考虑。

## 3 Scaling CRAQ

> In this section, we discuss how applications can specify various chain layout schemes in CRAQ, both within a single datacenter and across multiple datacenters. We then describe how to use a coordination service to store the chain metadata and group membership information.

### 3.1 Chain Placement Strategies

> Applications that use distributed storage services can be diverse in their requirements. Some common situations that occur may include:

使用分布式存储服务的应用程序的需求可能多种多样。 发生的一些常见情况可能包括：

> • Most or all writes to an object might originate in a single datacenter.

- 对对象的大部分或全部写入可能源自单个数据中心。

> • Some objects may be only relevant to a subset of datacenters.

- 某些对象可能仅与数据中心的一个子集相关。

> • Popular objects might need to be heavily replicated while unpopular ones can be scarce.

- 流行的对象可能需要大量复制，而不受欢迎的对象可能很少。

> CRAQ provides flexible chain configuration strategies that satisfy these varying requirements through the use of a two-level naming hierarchy for objects. An object’s identifier consists of both a chain identifier and a key identifier. The chain identifier determines which nodes in CRAQ will store all keys within that chain, while the key identifier provides unique naming per chain. We describe multiple ways of specifying application requirements:

CRAQ 提供了灵活的链配置策略，通过使用对象的两级命名层次结构来满足这些不同的要求。 对象的标识符由链标识符和密钥标识符组成。 链标识符确定 CRAQ 中的哪些节点将存储该链中的所有密钥，而密钥标识符为每个链提供唯一的命名。 我们描述了多种指定应用程序要求的方法： 

> 1.Implicit Datacenters & Global Chain Size:

隐式数据中心和全局链规模：

> {num_datacenters, chain_size}

> In this method, the number of datacenters that will store the chain is defined, but not explicitly which datacenters. To determine exactly which datacenters store the chain, consistent hashing is used with unique datacenter identifiers.

在此方法中，定义了将存储链的数据中心的数量，但没有明确定义哪些数据中心。 为了准确确定哪个数据中心存储链，一致的散列与唯一的数据中心标识符一起使用。

> 2.Explicit Datacenters & Global Chain Size:

显式数据中心和全球链规模

> {chain_size, dc1, dc2, ..., dcN}

> Using this method, every datacenter uses the same chain size to store replicas within the datacenter. The head of the chain is located within datacenter dc1, the tail of the chain is located within datacenter dcN, and the chain is ordered based on the provided list of datacenters. To determine which nodes within a datacenter store objects assigned to the chain, consistent hashing is used on the chain identifier. Each datacenter dci has a node which connects to the tail of datacenter dci−1 and a node which connects to the head of datacenter dci+1, respectively. An additional enhancement is to allow chain_size to be 0 which indicates that the chain should use all nodes within each datacenter.

使用这种方法，每个数据中心都使用相同的链大小在数据中心内存储副本。 链的头部位于数据中心 dc1 内，链的尾部位于数据中心 dcN 内，并根据提供的数据中心列表对链进行排序。 为了确定数据中心内的哪些节点存储分配给链的对象，对链标识符使用一致散列。 每个数据中心 dci 都有一个连接到数据中心 dci-1 尾部的节点和一个连接到数据中心 dci+1 头部的节点。 一个额外的增强是允许 chain_size 为 0，这表明链应该使用每个数据中心内的所有节点。

> 3.Explicit Datacenter Chain Sizes:

明确的数据中心链大小：

> {dc1, chain_size1, ..., dcN, chain_sizeN}

> Here the chain size within each datacenter is specified separately. This allows for non-uniformity in chain load balancing. The chain nodes within each datacenter are chosen in the same manner as the previous method, and chain_sizei can also be set to 0.

这里每个数据中心内的链大小是单独指定的。 这允许链负载平衡中的非均匀性。 每个数据中心内的链节点的选择方式与前面的方法相同，chain_sizei也可以设置为0。

> In methods 2 and 3 above, dc1 can be set as a master datacenter. If a datacenter is the master for a chain, this means that writes to the chain will only be accepted by that datacenter during transient failures. Otherwise, if dc1 is disconnected from the rest of the chain, dc2 could become the new head and take over write operations until dc1 comes back online. When a master is not defined, writes will only continue in a partition if the partition contains a majority of the nodes in the global chain. Otherwise, the partition will become read-only for maximumbounded inconsistent read operations, as defined in Section 2.4.

在上面的方法2和3中，可以将dc1设置为主数据中心。 如果一个数据中心是一个链的主节点，这意味着只有在瞬时故障期间，该数据中心才会接受对链的写入。 否则，如果 dc1 与链的其余部分断开连接，则 dc2 可能成为新的头并接管写入操作，直到 dc1 重新联机。 如果未定义主节点，则只有在分区包含全局链中的大多数节点时，才会在分区中继续写入。 否则，对于最大有界不一致读取操作，分区将变为只读，如第 2.4 节中所定义。

> CRAQ could easily support other more complicated methods of chain configuration. For example, it might be desirable to specify an explicit backup datacenter which only participates in the chain if another datacenter fails. One could also define a set of datacenters (e.g., “East coast”), any one of which could fill a single slot in the ordered list of datacenters of method 2. For brevity, we do not detail more complicated methods.

CRAQ 可以轻松支持其他更复杂的链配置方法。 例如，可能需要指定一个显式备份数据中心，如果另一个数据中心出现故障，该数据中心仅参与链。 还可以定义一组数据中心（例如，“东海岸”），其中任何一个都可以填充方法 2 的有序数据中心列表中的单个插槽。 为简洁起见，我们不详细介绍更复杂的方法。

> There is no limit on the number of key identifiers that can be written to a single chain. This allows for highly flexible configuration of chains based on application needs.

可以写入单个链的密钥标识符的数量没有限制。 这允许根据应用需求高度灵活地配置链。

### 3.2 CRAQ within a Datacenter

> The choice of how to distribute multiple chains across a datacenter was investigated in the original Chain Replication work. In CRAQ’s current implementation, we place chains within a datacenter using consistent hashing [29, 45], mapping potentially many chain identifiers to a single head node. This is similar to a growing number of datacenter-based object stores [15, 16]. An alternative approach, taken by GFS [22] and promoted in CR [47], is to use the membership management service as a directory service in assigning and storing randomized chain membership, i.e., each chain can include some random set of server nodes. This approach improves the potential for parallel system recovery. It comes at the cost, however, of increased centralization and state. CRAQ could easily use this alternative organizational design as well, but it would require storing more metadata information in the coordination service.

在最初的 Chain Replication 工作中研究了如何跨数据中心分布多个链的选择。 在 CRAQ 的当前实现中，我们使用一致散列 [29, 45] 在数据中心内放置链，将潜在的许多链标识符映射到单个头节点。 这类似于越来越多的基于数据中心的对象存储 [15, 16]。 GFS [22] 采用并在 CR [47] 中推广的另一种方法是使用成员资格管理服务作为分配和存储随机链成员资格的目录服务，即每个链可以包括一些随机的服务器节点集。 这种方法提高了并行系统恢复的潜力。 然而，这是以增加中心化和国家为代价的。 CRAQ 也可以轻松地使用这种替代组织设计，但它需要在协调服务中存储更多元数据信息。

### 3.3 CRAQ Across Multiple Datacenters

> CRAQ’s ability to read from any node improves its latency when chains stretch across the wide-area: When clients have flexibility in their choice of node, they can choose one that is nearby (or even lightly loaded). As long as the chain is clean, the node can return its local replica of an object without having to send any wide-area requests. With traditional CR, on the other hand, all reads would need to be handled by the potentially-distant tail node. In fact, various designs may choose head and/or tail nodes in a chain based on their datacenter, as objects may experience significant reference locality. Indeed, the design of PNUTS [12], Yahoo!’s new distributed database, is motivated by the high write locality observed in their datacenters.

当链跨越广域时，CRAQ 从任何节点读取数据的能力改善了它的延迟：当客户在选择节点方面具有灵活性时，他们可以选择附近的节点（甚至负载较轻的节点）。只要链是干净的，节点就可以返回对象的本地副本，而无需发送任何广域请求。另一方面，对于传统的 CR，所有读取都需要由潜在遥远的尾节点处理。事实上，各种设计可能会根据其数据中心选择链中的头和/或尾节点，因为对象可能会经历重要的参考位置。事实上，雅虎新的分布式数据库 PNUTS [12] 的设计灵感来自于在其数据中心观察到的高写入局部性。

> That said, applications might further optimize the selection of wide-area chains to minimize write latency and reduce network costs. Certainly the naive approach of building chains using consistent hashing across the entire global set of nodes leads to randomized chain successors and predecessors, potentially quite distant. Furthermore, an individual chain may cross in and out of a datacenter (or particular cluster within a datacenter) several times. With our chain optimizations, on the other hand, applications can minimize write latency by carefully selecting the order of datacenters that comprise a chain, and we can ensure that a single chain crosses the network boundary of a datacenter only once in each direction.

也就是说，应用程序可能会进一步优化广域链的选择，以最大限度地减少写入延迟并降低网络成本。当然，在整个全局节点集上使用一致散列构建链的幼稚方法会导致随机的链后继和前驱，可能相当遥远。此外，单个链可能会多次进出数据中心（或数据中心内的特定集群）。另一方面，通过我们的链优化，应用程序可以通过仔细选择组成链的数据中心的顺序来最大程度地减少写入延迟，并且我们可以确保单个链在每个方向上仅穿过数据中心的网络边界一次。

> Even with an optimized chain, the latency of write operations over wide-area links will increase as more datacenters are added to the chain. Although this increased latency could be significant in comparison to a primary/backup approach which disseminates writes in parallel, it allows writes to be pipelined down the chain. This vastly improves write throughput over the primary/backup approach.

即使使用优化的链，随着更多数据中心添加到链中，广域链接上的写入操作的延迟也会增加。尽管与并行传播写入的主/备份方法相比，这种增加的延迟可能是显着的，但它允许写入沿链向下传输。这大大提高了主/备份方法的写入吞吐量。

### 3.4 ZooKeeper Coordination Service

> Building a fault-tolerant coordination service for distributed applications is notoriously error prone. An earlier version of CRAQ contained a very simple, centrallycontrolled coordination service that maintained membership management. We subsequently opted to leverage ZooKeeper [48], however, to provide CRAQ with a robust, distributed, high-performance method for tracking group membership and an easy way to store chain metadata. Through the use of Zookeper, CRAQ nodes are guaranteed to receive a notification when nodes are added to or removed from a group. Similarly, a node can be notified when metadata in which it has expressed interest changes.

为分布式应用程序构建容错协调服务是出了名的容易出错。早期版本的 CRAQ 包含一个非常简单的、集中控制的协调服务，用于维护成员资格管理。然而，我们随后选择利用 ZooKeeper [48]，为 CRAQ 提供一种强大的、分布式的、高性能的方法来跟踪组成员身份以及一种存储链元数据的简单方法。通过使用 Zookeper，CRAQ 节点可以保证在节点添加到组或从组中删除时收到通知。类似地，当节点表示感兴趣的元数据发生变化时，可以通知节点。

> ZooKeeper provides clients with a hierarchical namespace similar to a filesystem. The filesystem is stored in memory and backed up to a log at each ZooKeeper instance, and the filesystem state is replicated across multiple ZooKeeper nodes for reliability and scalability. To reach agreement, ZooKeeper nodes use an atomic broadcast protocol similar to two-phase-commit. Optimized for read-mostly, small-sized workloads, ZooKeeper provides good performance in the face of many readers since it can serve the majority of requests from memory.

ZooKeeper 为客户端提供类似于文件系统的分层命名空间。文件系统存储在内存中并备份到每个 ZooKeeper 实例的日志中，并且文件系统状态在多个 ZooKeeper 节点之间复制以提高可靠性和可扩展性。为了达成一致，ZooKeeper 节点使用类似于两阶段提交的原子广播协议。针对以读取为主的小型工作负载进行了优化，ZooKeeper 在面对众多读取器时提供了良好的性能，因为它可以处理来自内存的大部分请求。

> Similar to traditional filesystem namespaces, ZooKeeper clients can list the contents of a directory, read the value associated with a file, write a value to a file, and receive a notification when a file or directory is modified or deleted. ZooKeeper’s primitive operations allow clients to implement many higher-level semantics such as group membership, leader election, event notification, locking, and queuing.

与传统的文件系统命名空间类似，ZooKeeper 客户端可以列出目录的内容，读取与文件关联的值，将值写入文件，并在文件或目录被修改或删除时收到通知。 ZooKeeper 的原始操作允许客户端实现许多更高级别的语义，例如组成员资格、领导者选举、事件通知、锁定和排队。

> Membership management and chain metadata across multiple datacenters does introduce some challenges. In fact, ZooKeeper is not optimized for running in a multidatacenter environment: Placing multiple ZooKeeper nodes within a single datacenter improves Zookeeper read scalability within that datacenter, but at the cost of wide-area performance. Since the vanilla implementation has no knowledge of datacenter topology or notion of hierarchy, coordination messages between Zookeeper nodes are transmitted over the wide-area network multiple times. Still, our current implementation ensures that CRAQ nodes always receive notifications from local Zookeeper nodes, and they are further notified only about chains and node lists that are relevant to them. We expand on our coordination through Zookeper in §5.1.

跨多个数据中心的成员资格管理和链元数据确实带来了一些挑战。事实上，ZooKeeper 并未针对在多数据中心环境中运行进行优化：在单个数据中心内放置多个 ZooKeeper 节点可提高该数据中心内的 Zookeeper 读取可扩展性，但以广域性能为代价。由于 vanilla 实现不了解数据中心拓扑或层次结构的概念，因此 Zookeeper 节点之间的协调消息通过广域网多次传输。尽管如此，我们当前的实现确保 CRAQ 节点始终接收来自本地 Zookeeper 节点的通知，并且它们仅被进一步通知与它们相关的链和节点列表。我们在 §5.1 中通过 Zookeper 扩展了我们的协调。

> To remove the redundancy of cross-datacenter ZooKeeper traffic, one could build a hierarchy of Zookeeper instances: Each datacenter could contain its own local ZooKeeper instance (of multiple nodes), as well as having a representative that participates in the global ZooKeeper instance (perhaps selected through leader election among the local instance). Separate functionality could then coordinate the sharing of data between the two. An alternative design would be to modify ZooKeeper itself to make nodes aware of network topology, as CRAQ currently is. We have yet to fully investigate either approach and leave this to future work.

为了消除跨数据中心 ZooKeeper 流量的冗余，可以构建 Zookeeper 实例的层次结构：每个数据中心可以包含自己的本地 ZooKeeper 实例（多个节点），以及有一个代表参与全局 ZooKeeper 实例（可能通过本地实例中的领导者选举来选择）。然后，单独的功能可以协调两者之间的数据共享。另一种设计是修改 ZooKeeper 本身，使节点了解网络拓扑，就像 CRAQ 当前那样。我们尚未对这两种方法进行全面调查，并将其留给未来的工作。

## 4 Extensions

> This section discusses some additional extensions to CRAQ, including its facility with mini-transactions and the use of multicast to optimize writes. We are currently in the process of implementing these extensions.

本节讨论 CRAQ 的一些附加扩展，包括它的小型事务工具和使用多播来优化写入。 我们目前正在实施这些扩展。

### 4.1 Mini-Transactions on CRAQ

> The whole-object read/write interface of an object store may be limiting for some applications. For example, a BitTorrent tracker or other directory service would want to support list addition or deletion. An analytics service may wish to store counters. Or applications may wish to provide conditional access to certain objects. None of these are easy to provide only armed with a pure objectstore interface as described so far, but CRAQ provides key extensions that support transactional operations.

对象存储的整体对象读/写接口可能会限制某些应用程序。 例如，BitTorrent 跟踪器或其他目录服务可能希望支持列表添加或删除。 分析服务可能希望存储计数器。 或者应用程序可能希望提供对某些对象的有条件访问。 到目前为止，仅使用纯对象存储接口来提供这些都不容易，但是 CRAQ 提供了支持事务操作的关键扩展。

#### 4.1.1 Single-Key Operations

> Several single-key operations are trivial to implement, which CRAQ already supports:
> • Prepend/Append: Adds data to the beginning or end of an object’s current value.
> • Increment/Decrement: Adds or subtracts to a key’s object, interpreted as an integer value.
> • Test-and-Set: Only update a key’s object if its current version number equals the version number specified in the operation.

几个单键操作实现起来很简单，CRAQ 已经支持

- Prepend/Append: 将数据添加到对象当前值的开头或结尾。
- Increment/Decrement：增加或减少键的对象，解释为整数值。
- Test-and-Set：仅当当前版本号等于操作中指定的版本号时才更新密钥的对象。

> For Prepend/Append and Increment/Decrement operations, the head of the chain storing the key’s object can simply apply the operation to the latest version of the object, even if the latest version is dirty, and then propagate a full replacement write down the chain. Furthermore, if these operations are frequent, the head can buffer the requests and batch the updates. These enhancements would be much more expensive using a traditional two-phasecommitprotocol.

对于 Prepend/Append 和 Increment/Decrement 操作，存储键对象的链头可以简单地将操作应用到对象的最新版本，即使最新版本是脏的，然后传播完整替换记下链。此外，如果这些操作很频繁，头部可以缓冲请求并批量更新。使用传统的两阶段提交协议，这些增强功能的成本会高得多。

> For the test-and-set operation, the head of the chain checks if its most recent committed version number equals the version number specified in the operation. If there are no outstanding uncommitted versions of the object, the head accepts the operation and propagates an update down the chain. If there are outstanding writes, we simply reject the test-and-set operation, and clients are careful to back off their request rate if continuously rejected. Alternatively, the head could “lock” the object by disallowing writes until the object is clean and recheck the latest committed version number, but since it is very rare that an uncommitted write is aborted and because locking the object would significantly impact performance, we chose not to implement this alternative.

对于 test-and-set 操作，链的头部检查其最近提交的版本号是否等于操作中指定的版本号。如果对象没有未提交的未提交版本，则头部接受操作并沿链向下传播更新。如果有未完成的写入，我们简单地拒绝测试和设置操作，如果连续拒绝，客户端会小心地降低他们的请求率。或者，头部可以通过禁止写入来“锁定”对象，直到对象干净并重新检查最新提交的版本号，但由于未提交的写入很少被中止，并且因为锁定对象会显着影响性能，我们选择不实施此替代方案。

> The test-and-set operation could also be designed to accept a value rather than a version number, but this introduces additional complexity when there are outstanding uncommitted versions. If the head compares against the most recent committed version of the object (by contacting the tail), any writes that are currently in progress would not be accounted for. If instead the head compares against the most recent uncommitted version, this violates consistency guarantees. To achieve consistency, the head would need to temporarily lock the object by disallowing (or temporarily delaying) writes until the object is clean. This does not violate consistency guarantees and ensures that no updates are lost, but could significantly impact write performance.

test-and-set 操作也可以设计为接受一个值而不是一个版本号，但是当存在未提交的未提交版本时，这会引入额外的复杂性。如果头部与对象的最新提交版本进行比较（通过联系尾部），则不会考虑当前正在进行的任何写入。相反，如果头部与最新的未提交版本进行比较，这违反了一致性保证。为了实现一致性，头部需要通过禁止（或暂时延迟）写入来暂时锁定对象，直到对象干净为止。这不会违反一致性保证并确保不会丢失更新，但可能会显着影响写入性能。

#### 4.1.2 Single-Chain Operations

> Sinfonia’s recently proposed “mini-transactions” provide an attractive lightweight method [2] of performing transactions on multiple keys within a single chain. A minitransaction is defined by a compare, read, and write set; Sinfonia exposes a linear address space across many memory nodes. A compare set tests the values of the specified address location and, if they match the provided values, executes the read and write operations. Typically designed for settings with low write contention, Sinfonia’s mini-transactions use an optimistic two-phase commit protocol. The prepare message attempts to grab a lock on each specified memory address (either because different addresses were specified, or the same address space is being implemented on multiple nodes for fault tolerance). If all addresses can be locked, the protocol commits; otherwise, the participant releases all locks and retries later.

Sinfonia 最近提出的“迷你交易”提供了一种有吸引力的轻量级方法 [2]，可以在单个链中的多个密钥上执行交易。小事务由比较、读取和写入集定义； Sinfonia 在许多内存节点上公开了一个线性地址空间。比较集测试指定地址位置的值，如果它们与提供的值匹配，则执行读取和写入操作。 Sinfonia 的小型事务通常专为低写入争用的设置而设计，使用乐观的两阶段提交协议。准备消息试图在每个指定的内存地址上获取一个锁（要么是因为指定了不同的地址，要么是为了容错在多个节点上实现了相同的地址空间）。如果所有地址都可以锁定，则协议提交；否则，参与者将释放所有锁定并稍后重试。

> CRAQ’s chain topology has some special benefits for supporting similar mini-transactions, as applications can designate multiple objects be stored on the same chain— i.e., those that appear regularly together in multi-object mini-transactions—in such a way that preserves locality. Objects sharing the same chainid will be assigned the same node as their chain head, reducing the two-phase commit to a single interaction because only one head node is involved. CRAQ is unique in that mini-transactions that only involve a single chain can be accepted using only the single head to mediate access, as it controls write access to all of a chain’s keys, as opposed to all chain nodes. The only trade-off is that write throughput may be affected if the head needs to wait for keys in the transaction to become clean (as described in §4.1.1). That said, this problem is only worse in Sinfonia as it needs to wait (by exponentially backing off the mini-transaction request) for unlocked keys across multiple nodes. Recovery from failure is similarly easier in CRAQ as well.

CRAQ 的链拓扑在支持类似的小交易方面有一些特殊的好处，因为应用程序可以指定多个对象存储在同一链上——即那些在多对象小交易中经常一起出现的对象——以这种方式保持局部性.共享相同链 ID 的对象将被分配与其链头相同的节点，将两阶段提交减少到单个交互，因为只涉及一个头节点。 CRAQ 的独特之处在于，仅涉及单个链的小交易可以仅使用单个头来调解访问，因为它控制对所有链的密钥的写入访问，而不是所有链节点。唯一的权衡是，如果头需要等待事务中的密钥变干净（如第 4.1.1 节所述），则写入吞吐量可能会受到影响。也就是说，这个问题在 Sinfonia 中只会更糟，因为它需要等待（通过指数回退小交易请求）跨多个节点解锁密钥。在 CRAQ 中从故障中恢复也同样容易。

#### 4.1.3 Multi-Chain Operations

> Even when multiple chains are involved in multi-object updates, the optimistic two-phase protocol need only be implemented with the chain heads, not all involved nodes. The chain heads can lock any keys involved in the minitransaction until it is fully committed.

即使在多对象更新中涉及多个链时，乐观两阶段协议也只需要用链头来实现，而不是所有涉及的节点。 链头可以锁定小交易中涉及的任何密钥，直到它完全提交。

> Of course, application writers should be careful with the use of extensive locking and mini-transactions: They reduce the write throughput of CRAQ as writes to the same object can no longer be pipelined, one of the very benefits of chain replication.

当然，应用程序编写者在使用广泛的锁定和小事务时应该小心：它们会降低 CRAQ 的写入吞吐量，因为对同一对象的写入不能再流水线化，这是链式复制的一大好处。

### 4.2 Lowering Write Latency with Multicast

> CRAQ can take advantage of multicast protocols [41] to improve write performance, especially for large updates or long chains. Since chain membership is stable between node membership changes, a multicast group can be created for each chain. Within a datacenter, this would probably take the form of a network-layer multicast protocol, while application-layer multicast protocols may be bettersuited for wide-area chains. No ordering or reliability guarantees are required from these multicast protocols.

CRAQ 可以利用多播协议 [41] 来提高写入性能，特别是对于大型更新或长链。由于链成员在节点成员变化之间是稳定的，因此可以为每个链创建一个多播组。在数据中心内，这可能采用网络层多播协议的形式，而应用层多播协议可能更适合广域链。这些多播协议不需要排序或可靠性保证。

> Then, instead of propagating a full write serially down a chain, which adds latency proportional to the chain length, the actual value can be multicast to the entire chain. Then, only a small metadata message needs to be propagated down the chain to ensure that all replicas have received a write before the tail. If a node does not receive the multicast for any reason, the node can fetch the object from its predecessor after receiving the write commit message and before further propagating the commit message.

然后，不是在链上串行传播完整写入，这会增加与链长度成比例的延迟，而是可以将实际值多播到整个链。然后，只需要沿着链向下传播一条小的元数据消息，以确保所有副本在尾部之前都收到了写入。如果节点由于任何原因没有接收到多播，则该节点可以在收到写入提交消息后和进一步传播提交消息之前从其前任获取对象。

> Additionally, when the tail receives a propagated write request, a multicast acknowledgment message can be sent to the multicast group instead of propagating it backwards along the chain. This reduces both the amount of time it takes for a node’s object to re-enter the clean state after a write, as well as the client’s perceived write delay. Again, no ordering or reliability guarantees are required when multicasting acknowledgments—if a node in the chain does not receive an acknowledgement, it will reenter the clean state when the next read operation requires it to query the tail.

此外，当尾部接收到传播的写入请求时，可以将多播确认消息发送到多播组，而不是沿链向后传播。这既减少了节点对象在写入后重新进入干净状态所需的时间，也减少了客户端感知的写入延迟。同样，在多播确认时不需要排序或可靠性保证——如果链中的节点没有收到确认，当下一个读取操作要求它查询尾部时，它将重新进入干净状态。

## 5 Management and Implementation

> Our prototype implementation of Chain Replication and CRAQ is written in approximately 3,000 lines of C++ using the Tame extensions [31] to the SFS asynchronous I/O and RPC libraries [38]. All network functionality between CRAQ nodes is exposed via Sun RPC interfaces.

我们的链复制和 CRAQ 原型实现是用大约 3,000 行 C++ 编写的，使用 Tame 扩展 [31] 到 SFS 异步 I/O 和 RPC 库 [38]。 CRAQ 节点之间的所有网络功能都通过 Sun RPC 接口公开。

### 5.1 Integrating ZooKeeper

> As described in §3.4, CRAQ needs the functionality of a group membership service. We use a ZooKeeper file structure to maintain node list membership within each datacenter. When a client creates a file in ZooKeeper, it can be marked as ephemeral. Ephemeral files are automatically deleted if the client that created the file disconnects from ZooKeeper. During initialization, a CRAQ node creates an ephemeral file in /nodes/dc_name/node_id, where dc_name is the unique name of its datacenter (as specified by an administrator) and node_id is a node identifier unique to the node’s datacenter. The content of the file contains the node’s IP address and port number.

如第 3.4 节所述，CRAQ 需要群组成员服务的功能。我们使用 ZooKeeper 文件结构来维护每个数据中心内的节点列表成员资格。当客户端在 ZooKeeper 中创建文件时，可以将其标记为临时文件。如果创建临时文件的客户端与 ZooKeeper 断开连接，则会自动删除临时文件。在初始化期间，CRAQ 节点在 /nodes/dc_name/node_id 中创建一个临时文件，其中 dc_name 是其数据中心的唯一名称（由管理员指定），node_id 是节点数据中心唯一的节点标识符。该文件的内容包含节点的 IP 地址和端口号。

> CRAQ nodes can query /nodes/dc_name to determine the membership list for its datacenter, but instead of having to periodically check the list for changes, ZooKeeper provides processes with the ability to create a watch on a file. A CRAQ node, after creating an ephemeral file to notify other nodes it has joined the system, creates a watch on the children list of /nodes/dc_name, thereby guaranteeing that it receives a notification when a node is added or removed.

CRAQ 节点可以查询 /nodes/dc_name 以确定其数据中心的成员资格列表，但不必定期检查列表的更改，ZooKeeper 为进程提供了在文件上创建监视的能力。一个 CRAQ 节点在创建一个临时文件通知其他节点它已经加入系统后，在 /nodes/dc_name 的孩子列表上创建一个监视，从而保证它在添加或删除节点时收到通知。

> When a CRAQ node receives a request to create a new chain, a file is created in /chains/chain_id, where chain_id is a 160-bit unique identifier for the chain. The chain’s placement strategy (defined in §3.1) determines the contents of the file, but it only includes this chain configuration information, not the list of a chain’s current nodes. Any node participating in the chain will query the chain file and place a watch on it as to be notified if the chain metadata changes.

当 CRAQ 节点收到创建新链的请求时，会在 /chains/chain_id 中创建一个文件，其中 chain_id 是该链的 160 位唯一标识符。链的放置策略（在 §3.1 中定义）决定了文件的内容，但它只包含这个链配置信息，而不是链当前节点的列表。任何参与链的节点都将查询链文件并对其进行监视，以便在链元数据发生更改时收到通知。

> Although this approach requires that nodes keep track of the CRAQ node list of entire datacenters, we chose this method over the alternative approach in which nodes register their membership for each chain they belong to (i.e., chain metadata explicitly names the chain’s current members). We make the assumption that the number of chains will generally be at least an order of magnitude larger than the number of nodes in the system, or that chain dynamism may be significantly greater than nodes joining or leaving the system (recall that CRAQ is designed for managed datacenter, not peer-to-peer, settings). Deployments where the alternate assumptions hold can take the other approach of tracking per-chain memberships explicitly in the coordination service. If necessary, the current approach’s scalability can also be improved by having each node track only a subset of datacenter nodes: We can partition node lists into separate directories within /nodes/dc_name/ according to node_id prefixes, with nodes monitoring just their own and nearby prefixes.

虽然这种方法要求节点跟踪整个数据中心的 CRAQ 节点列表，但我们选择了这种方法，而不是节点为其所属的每个链注册其成员资格的替代方法（即，链元数据明确命名链的当前成员） .我们假设链的数量通常至少比系统中的节点数量大一个数量级，或者链的动态性可能明显大于加入或离开系统的节点（回想一下 CRAQ 是为管理的数据中心，而不是对等设置）。替代假设成立的部署可以采用另一种方法在协调服务中明确跟踪每个链的成员资格。如有必要，还可以通过让每个节点仅跟踪数据中心节点的子集来提高当前方法的可扩展性：我们可以根据 node_id 前缀将节点列表划分到 /nodes/dc_name/ 中的单独目录中，节点仅监控自己和附近的节点前缀。

> It is worth noting that we were able to integrate ZooKeeper’s asynchronous API functions into our codebase by building tame-style wrapper functions. This allowed us to twait on our ZooKeeper wrapper functions which vastly reduced code complexity.

值得注意的是，我们能够通过构建驯服风格的包装函数将 ZooKeeper 的异步 API 函数集成到我们的代码库中。这允许我们等待我们的 ZooKeeper 包装函数，这大大降低了代码复杂性。

### 5.2 Chain Node Functionality

> Our chainnode program implements most of CRAQ’s functionality. Since much of the functionality of Chain Replication and CRAQ is similar, this program operates as either a Chain Replication node or a CRAQ node based on a run-time configuration setting.

我们的链节点程序实现了 CRAQ 的大部分功能。由于 Chain Replication 和 CRAQ 的大部分功能相似，因此该程序根据运行时配置设置作为 Chain Replication 节点或 CRAQ 节点运行。

> Nodes generate a random identifier when joining the system, and the nodes within each datacenter organize themselves into a one-hop DHT [29, 45] using these identifiers. A node’s chain predecessor and successor are defined as its predecessor and successor in the DHT ring. Chains are also named by 160-bit identifiers. For a chain Ci, the DHT successor node for Ci is selected as the chain’s first node in that datacenter. In turn, this node’s S DHT successors complete the datacenter subchain, where S is specified in chain metadata. If this datacenter is the chain’s first (resp. last), than this first (resp. last) node is the chain’s ultimate head (resp. tail).

节点在加入系统时生成一个随机标识符，每个数据中心内的节点使用这些标识符将自己组织成一个单跳 DHT [29, 45]。一个节点的链前驱和后继被定义为其在 DHT 环中的前驱和后继。链也由 160 位标识符命名。对于链 Ci，Ci 的 DHT 后继节点被选为该数据中心链的第一个节点。反过来，该节点的 S DHT 后继者完成数据中心子链，其中 S 在链元数据中指定。如果这个数据中心是链的第一个（相应的最后一个），那么这个第一个（相应的最后一个）节点就是链的最终头（相应的尾）。

> All RPC-based communication between nodes, or between nodes and clients, is currently over TCP connections (with Nagle’s algorithm turned off). Each node maintains a pool of connected TCP connections with its chain’s predecessor, successor, and tail. Requests are pipelined and round-robin’ed across these connections. All objects are currently stored only in memory, although our storage abstraction is well-suited to use an in-process key-value store such as BerkeleyDB [40], which we are in the process of integrating.

节点之间或节点与客户端之间的所有基于 RPC 的通信目前都通过 TCP 连接（关闭 Nagle 的算法）。每个节点都维护一个与其链的前驱、后继和尾部连接的 TCP 连接池。请求在这些连接中被流水线化和循环。所有对象目前仅存储在内存中，尽管我们的存储抽象非常适合使用进程内键值存储，例如我们正在集成的 BerkeleyDB [40]。

> For chains that span across multiple datacenters, the last node of one datacenter maintains a connection to the first node of its successor datacenter. Any node that maintains a connection to a node outside of its datacenter must also place a watch on the node list of the external datacenter. Note, though, that when the node list changes in an external datacenter, nodes subscribing to changes will receive notification from their local ZooKeeper instance only, avoiding additional cross-datacenter traffic.

对于跨越多个数据中心的链，一个数据中心的最后一个节点与其后继数据中心的第一个节点保持连接。任何与其数据中心外的节点保持连接的节点还必须对外部数据中心的节点列表进行监视。但是请注意，当外部数据中心中的节点列表发生更改时，订阅更改的节点将仅收到来自其本地 ZooKeeper 实例的通知，从而避免了额外的跨数据中心流量。

### 5.3 Handling Memberships Changes

> For normal write propagation, CRAQ nodes follow the protocol in §2.3. A second type of propagation, called back-propagation, is sometimes necessary during recovery, however: It helps maintain consistency in response to node additions and failures. For example, if a new node joins CRAQ as the head of an existing chain (given its position in the DHT), the previous head of the chain needs to propagate its state backwards. But the system needs to also be robust to subsequent failures during recovery, which can cascade the need for backwards propagation farther down the chain (e.g., if the now-second chain node fails before completing its back-propagation to the now-head). The original Chain Replication paper did not consider such recovery issues, perhaps because it only described a more centrally-controlled and statically-configured version of chain membership, where new nodes are always added to a chain’s tail.

对于正常的写入传播，CRAQ 节点遵循 §2.3 中的协议。第二种类型的传播称为反向传播，在恢复期间有时是必需的，但是：它有助于在响应节点添加和故障时保持一致性。例如，如果一个新节点作为现有链的头加入 CRAQ（给定它在 DHT 中的位置），则链的前一个头需要向后传播其状态。但是系统还需要对恢复期间的后续故障具有鲁棒性，这可以将反向传播的需求级联到更远的链（例如，如果现在第二个链节点在完成其向现在头的反向传播之前失败）。最初的 Chain Replication 论文没有考虑这样的恢复问题，也许是因为它只描述了一个更集中控制和静态配置的链成员版本，其中新节点总是添加到链的尾部。

> Because of these possible failure conditions, when a new node joins the system, the new node receives propagation messages both from its predecessor and backpropagation from its successor in order to ensure its correctness. A new node refuses client read requests for a particular object until it reaches agreement with its successor. In both methods of propagation, nodes may use set reconciliation algorithms to ensure that only needed objects are actually propagated during recovery.

由于这些可能的故障条件，当新节点加入系统时，新节点会从其前任和后继的反向传播接收传播消息，以确保其正确性。新节点拒绝客户端对特定对象的读取请求，直到它与其后继节点达成一致。在这两种传播方法中，节点可以使用集合协调算法来确保在恢复期间仅实际传播需要的对象。

> Back-propagation messages always contain a node’s full state about an object. This means that rather than just sending the latest version, the latest clean version is sent along with all outstanding (newer) dirty versions. This is necessary to enable new nodes just joining the system to respond to future acknowledgment messages. Forward propagation supports both methods. For normal writes propagating down the chain, only the latest version is sent, but when recovering from failure or adding new nodes, full state objects are transmitted.

反向传播消息总是包含一个节点关于一个对象的完整状态。这意味着不是只发送最新版本，而是将最新的干净版本与所有未完成的（较新的）脏版本一起发送。这对于使刚加入系统的新节点能够响应未来的确认消息是必要的。前向传播支持这两种方法。对于沿链传播的正常​​写入，仅发送最新版本，但在从故障中恢复或添加新节点时，将发送完整状态对象。

> Let us now consider the following cases from node N’s point of view, where LC is the length of a chain C for which N is responsible.

现在让我们从节点 N 的角度考虑以下情况，其中 LC 是 N 负责的链 C 的长度。

> Node Additions. A new node, A,is added to the system.

节点添加。系统中添加了一个新节点 A。

> • If A is N’s successor, N propagates all objects in C to A. If A had been in the system before, N can perform object set reconciliation first to identity the specified object versions required to reach consistency with the rest of the chain.
> • If A is N’s predecessor:
> – N back-propagates all objects in C to A for which N is not the head.
> – A takes over as the tail of C if N was the previous tail.
>  – N becomes the tail of C if N’s successor was previously the tail.
>  – A becomes the new head for C if N was previously the head and A’s identifier falls between C and N’s identifier in the DHT.
>  • If A is within LC predecessors of N:
>  – If N was the tail for C, it relinquishes tail duties and stops participating in the chain. N can now mark its local copies of C’s objects as deletable, although it only recovers this space lazily to support faster state reconciliation if it later rejoins the chain C.
>  – If N’s successor was the tail for C, N assumes tail duties.
>  • If none of the above hold, no action is necessary. NodeDeletions. Anode,D,isremovedfromthesystem.
>  • If D was N’s successor, N propagates all objects in C to N’s new successor (again, minimizing transfer to only unknown, fresh object versions). N has to propagate its objects even if that node already belongs to the chain, as D could have failed before it propagated outstanding writes.
• If D was N’s predecessor:
– N back-propagates all needed objects to N’s new predecessor for which it is not the head. N needs to back-propagate its keys because D could have failed before sending an outstanding acknowledgment to its predecessor, or before finishing its own back-propagation.
– If D was the head for C, N assumes head duties.
– If N was the tail for C, it relinquishes tail duties and propagates all objects in C to N’s new successor.
• If D was within LC predecessors of N and N was the tail for C, N relinquishes tail duties and propagates all objects in C to N’s new successor.
• If none of the above hold, no action is necessary.

## 6 Evaluation

> This section evaluates the performance of our Chain Replication (CR) and CRAQ implementations. At a high level, we are interested in quantifying the read throughput benefits from CRAQ’s ability to apportion reads. On the flip side, version queries still need to be dispatched to the tail for dirty objects, so we are also interested in evaluating asymptotic behavior as the workload mixture changes. We also briefly evaluate CRAQ’s optimizations for wide-area deployment.

本节评估我们的链复制 (CR) 和 CRAQ 实施的性能。在较高的层次上，我们有兴趣量化 CRAQ 分配读取能力带来的读取吞吐量优势。另一方面，对于脏对象，仍然需要将版本查询分派到尾部，因此我们也有兴趣评估随着工作负载混合变化的渐进行为。我们还简要评估了 CRAQ 对广域部署的优化。

> All evaluations were performed on Emulab, a controlled network testbed. Experiments were run using the pc3000-type machines, which have 3GHz processors and 2GB of RAM. Nodes were connected on a 100MBit network. For the following tests, unless otherwise specified, we used a chain size of three nodes storing a single object connected together without any added synthetic latency. This setup seeks to better isolate the performance characteristics of single chains. All graphed data points are the median values unless noted; when present, error bars correspond to the 99th percentile values.

所有评估均在 Emulab（受控网络测试平台）上进行。使用具有 3GHz 处理器和 2GB RAM 的 pc3000 型机器运行实验。节点连接在 100MBit 网络上。对于以下测试，除非另有说明，否则我们使用三个节点的链大小存储连接在一起的单个对象，而没有任何额外的合成延迟。此设置旨在更好地隔离单链的性能特征。除非另有说明，所有绘制的数据点均为中值；当存在时，误差线对应于第 99 个百分位值。

> To determine maximal read-only throughput in both systems, we first vary the number of clients in Figure 4, which shows the aggregate read throughput for CR and CRAQ. Since CR has to read from a single node, throughput stays constant. CRAQ is able to read from all three nodes in the chain, so CRAQ throughput increases to three times that of CR. Clients in these experiments maintained a maximum window of outstanding requests (50), so the system never entered a potential livelock scenario.

为了确定两个系统中的最大只读吞吐量，我们首先改变图 4 中的客户端数量，该图显示了 CR 和 CRAQ 的总读取吞吐量。由于 CR 必须从单个节点读取，因此吞吐量保持不变。 CRAQ 能够读取链中所有三个节点，因此 CRAQ 吞吐量增加到 CR 的三倍。这些实验中的客户端保持了最大的未完成请求窗口 (50)，因此系统从未进入潜在的活锁场景。

> Figure 5 shows throughput for read, write, and testand-set operations. Here, we varied CRAQ chains from three to seven nodes, while maintaining read-only, writeonly, and transaction-only workloads. We see that read throughput scaled linearly with the number of chain nodes as expected. Write throughput decreased as chain length increased, but only slightly. Only one test-and-set operation can be outstanding at a time, so throughput is much lower than for writes. Test-and-set throughput also decreases as chain length increases because the latency for a single operation increases with chain length.

图 5 显示了读取、写入和测试与设置操作的吞吐量。在这里，我们将 CRAQ 链从三个节点改为七个节点，同时保持只读、只写和仅事务工作负载。我们看到读取吞吐量与预期的链节点数量成线性关系。写入吞吐量随着链长度的增加而下降，但幅度很小。一次只能执行一个测试并设置操作，因此吞吐量远低于写入。测试和设置吞吐量也会随着链长度的增加而降低，因为单个操作的延迟会随着链长度的增加而增加。

> To see how CRAQ performs during a mixed read/write workload, we set ten clients to continuously read a 500byte object from the chain while a single client varied its write rate to the same object. Figure 6 shows the aggregate read throughput as a function of write rate. Note that Chain Replication is not effected by writes, as all read requests are handled by the tail. Although throughput for CRAQ starts out at approximately three times the rate of CR (a median of 59,882 reads/s vs. 20,552 reads/s), as expected, this rate gradually decreases and flattens out to around twice the rate (39,873 reads/s vs. 20,430 reads/s). As writes saturate the chain, non-tail nodes are always dirty, requiring them always to first perform version requests to the tail. CRAQ still enjoys a performance benefit when this happens, however, as the tail’s saturation point for its combined read and version requests is still higher than that for read requests alone.

为了了解 CRAQ 在混合读/写工作负载中的表现，我们设置了 10 个客户端，从链中连续读取 500 字节的对象，而单个客户端则改变对同一对象的写入速率。图 6 显示了作为写入速率函数的总读取吞吐量。请注意，链复制不受写入影响，因为所有读取请求都由尾部处理。尽管 CRAQ 的吞吐量开始时大约是 CR 速率的三倍（中位数为 59,882 次读取/秒与 20,552 次读取/秒），但正如预期的那样，该速率逐渐下降并趋于平缓至速率（39,873 次读取/秒）的两倍左右与 20,430 次读取/秒）。当写入使链饱和时，非尾节点总是脏的，要求它们总是首先对尾执行版本请求。但是，当发生这种情况时，CRAQ 仍然享有性能优势，因为它的组合读取和版本请求的尾部饱和点仍然高于单独读取请求的饱和点。

> Figure 7 repeats the same experiment, but using a 5 KB object instead of a 500 byte one. This value was chosen as a common size for objects such as small Web images, while 500 bytes might be better suited for smaller database entries (e.g., blog comments, social-network status information, etc.). Again, CRAQ’s performance in read-only settings significantly outperforms that of CR with a chain size of three (6,808 vs. 2,275 reads/s), while it preserves good behavior even under high write rates (4,416 vs. 2,259 reads/s). This graph also includes CRAQ performance with seven-node chains. In both scenarios, even as the tail becomes saturated with requests, its ability to answer small version queries at a much higher rate than sending larger read replies allows aggregate read throughput to remain significantly higher than in CR.

图 7 重复了相同的实验，但使用了 5 KB 的对象而不是 500 字节的对象。选择此值作为对象（例如小型 Web 图像）的通用大小，而 500 字节可能更适合较小的数据库条目（例如，博客评论、社交网络状态信息等）。同样，CRAQ 在只读设置中的性能显着优于链大小为 3 的 CR（6,808 对 2,275 次读取/秒），同时即使在高写入速率（4,416 对 2,259 次读取/秒）下也能保持良好的行为。该图还包括七节点链的 CRAQ 性能。在这两种情况下，即使尾部被请求饱和，它以比发送更大的读取回复高得多的速率回答小版本查询的能力允许聚合读取吞吐量保持显着高于 CR 中的吞吐量。

> Figure 8 isolates the mix of dirty and clean reads that comprise Figure 6. As writes increase, the number of clean requests drops to 25.4% of its original value, since only the tail is clean as writes saturate the chain. The tail cannot maintain its own maximal read-only throughput (i.e., 33.3% of the total), as it now also handles version queries from other chain nodes. On the other hand, the number of dirty requests would approach two-thirds of the original clean read rate if total throughput remained constant, but since dirty requests are slower, the number of dirty requests flattens out at 42.3%. These two rates reconstruct the total observed read rate, which converges to 67.7% of read-only throughput during high write contention on the chain.

图 8 隔离了构成图 6 的脏读和干净读取的混合。随着写入的增加，干净请求的数量下降到其原始值的 25.4%，因为只有尾部是干净的，因为写入使链饱和。尾部无法维持自己的最大只读吞吐量（即总数的 33.3%），因为它现在还处理来自其他链节点的版本查询。另一方面，如果总吞吐量保持不变，脏请求的数量将接近原始干净读取率的三分之二，但由于脏请求较慢，脏请求的数量趋于平缓，为 42.3%。这两个速率重构了观察到的总读取速率，在链上的高写入争用期间收敛到只读吞吐量的 67.7%。

> The table in Figure 9 shows the latency in milliseconds of clean reads, dirty reads, writes to a 3-node chain, and writes to a 6-node chain, all within a single datacenter. Latencies are shown for objects of 500 bytes and 5 KB both when the operation is the only outstanding request (No Load) and when we saturate the CRAQ nodes with many requests (High Load). As expected, latencies are higher under heavy load, and latencies increase with key size. Dirty reads are always slower than clean reads because of the extra round-trip-time incurred, and write latency increases roughly linearly with chain size.

图 9 中的表格显示了干净读取、脏读取、写入 3 节点链和写入 6 节点链的延迟（以毫秒为单位），所有这些都在单个数据中心内。当操作是唯一未完成的请求（无负载）以及当我们用许多请求（高负载）使 CRAQ 节点饱和时，都会显示 500 字节和 5 KB 对象的延迟。正如预期的那样，重负载下的延迟会更高，并且延迟会随着密钥大小的增加而增加。脏读总是比干净读慢，因为会产生额外的往返时间，写延迟随着链的大小大致线性增加。

> Figure 10 demonstrates CRAQ’s ability to recover from failure. We show the loss in read-only throughput over time for chains of lengths 3, 5, and 7. Fifteen seconds into each test, one of the nodes in the chain was killed. After a few seconds, the time it takes for the node to time out and be considered dead by ZooKeeper, a new node joins the chain and throughput resumes to its original value. The horizontal lines drawn on the graph correspond to the maximum throughput for chains of lengths 1 through 7. This helps illustrate that the loss in throughput during the failure is roughly equal to 1/C, where C is the length of the chain.

图 10 展示了 CRAQ 从故障中恢复的能力。我们展示了长度为 3、5 和 7 的链的只读吞吐量随时间的损失。每次测试 15 秒后，链中的一个节点被杀死。几秒钟后，节点超时并被 ZooKeeper 视为死亡所需的时间，一个新节点加入链，吞吐量恢复到其原始值。图中绘制的水平线对应于长度为 1 到 7 的链的最大吞吐量。这有助于说明故障期间的吞吐量损失大致等于 1/C，其中 C 是链的长度。

> To measure the effect of failure on the latency of read and write operations, Figures 11 and 12 show the latency of these operations during the failure of a chain of length three. Clients that receive an error when trying to read an object choose a new random replica to read from, so failures have a low impact on reads. Writes, however, cannot be committed during the period between when a replica fails and when it is removed from the chain due to timeouts. This causes write latency to increase to the time it takes to complete failure detection. We note that this is the same situation as in any other primary/backup replication strategy which requires all live replicas to participate in commits. Additionally, clients can optionally configure a write request to return as soon as the head of the chain accepts and propagates the request down to the chain instead of waiting for it to commit. This reduces latency for clients that don’t require strong consistency.

为了测量故障对读写操作延迟的影响，图 11 和 12 显示了在长度为 3 的链发生故障期间这些操作的延迟。尝试读取对象时收到错误的客户端会选择一个新的随机副本进行读取，因此失败对读取的影响很小。但是，在副本失败和由于超时而从链中删除之间的时间段内无法提交写入。这会导致写入延迟增加到完成故障检测所需的时间。我们注意到，这与任何其他主/备份复制策略中的情况相同，它要求所有活动副本都参与提交。此外，客户端可以选择配置写入请求，使其在链头接受请求并将其向下传播到链时立即返回，而不是等待它提交。这减少了不需要强一致性的客户端的延迟。

> Finally, Figure 13 demonstrates CRAQ’s utility in wide-area deployments across datacenters. In this experiment, a chain was constructed over three nodes that each have 80ms of round-trip latency to one another (approximately the round-trip-time between U.S. coastal areas), as controlled using Emulab’s synthetic delay. The read client was not local to the chain tail (which otherwise could have just resulted in local-area performance as before).

最后，图 13 展示了 CRAQ 在跨数据中心的广域部署中的效用。在这个实验中，在三个节点上构建了一条链，每个节点之间都有 80 毫秒的往返延迟（大约是美国沿海地区之间的往返时间），使用 Emulab 的合成延迟进行控制。读取客户端不是链尾本地的（否则可能会像以前一样导致本地性能）。

> The figure evaluates read latency as the workload mixture changes; mean latency is now shown with standard deviation as error bars (as opposed to median and 99th percentile elsewhere). Since the tail is not local, CR’s latency remains constantly high, as it always incurs a wide-area read request. CRAQ, on the other hand, incurs almost no latency when no writes are occurring, as the read request can be satisfied locally. As the write rate increases, however, CRAQ reads are increasingly dirty, so the average latency rises. Once the write rate reaches about 15 writes/s, the latency involved in propagating write messages down the wide-area chain causes the client’s local node to be dirty 100% of the time, leading to a widearea version query. (CRAQ’s maximum latency is everso-slightly less than CR given that only metadata is transferred over the wide area, a difference that would only increase with larger objects, especially in slow-start scenarios.) Although this convergence to a 100% dirty state occurs at a much lower write rate than before, we note that careful chain placement allows any clients in the tail’s datacenter to enjoy local-area performance. Further, clients in non-tail datacenters that can be satisfied with a degree of maximum-bounded inconsistency (per §2.4) can also avoid wide-area requests.

该图评估了工作负载混合变化时的读取延迟；平均延迟现在以标准偏差显示为误差条（与其他地方的中位数和第 99 个百分位数相反）。由于尾部不是本地的，CR 的延迟一直很高，因为它总是会引发广域读取请求。另一方面，CRAQ 在没有写入发生时几乎不会产生延迟，因为可以在本地满足读取请求。然而，随着写入速率的增加，CRAQ 读取越来越脏，因此平均延迟增加。一旦写入速率达到大约 15 次写入/秒，沿广域链传播写入消息所涉及的延迟会导致客户端的本地节点 100% 的时间都是脏的，从而导致广域版本查询。 （CRAQ 的最大延迟总是略小于 CR，因为只有元数据在大范围内传输，这种差异只会随着更大的对象而增加，尤其是在慢启动的情况下。）虽然这种收敛到 100% 脏状态发生了以比以前低得多的写入速率，我们注意到谨慎的链放置允许尾部数据中心的任何客户端享受本地性能。此外，可以满足一定程度的最大有界不一致（根据 §2.4）的非尾部数据中心中的客户端也可以避免广域请求。

## 7 Related Work

> **Strong consistency in distributed systems.** Strong consistency among distributed servers can be provided through the use of primary/backup storage [3] and twophase commit protocols [43]. Early work in this area did not provide for availability in the face of failures (e.g., of the transaction manager), which led to the introduction of view change protocols (e.g., through leader consensus [33]) to assist with recovery. There has been a large body of subsequent work in this area; recent examples include both Chain Replication and the ring-based protocol of Guerraoui et al. [25], which uses a two-phase write protocol and delays reads during uncommitted writes. Rather than replicate content everywhere, one can explore other trade-offs between overlapping read and write sets in strongly-consistent quorum systems [23, 28]. Agreement protocols have also been extended to malicious settings, both for state machine replication [10, 34] and quorum systems [1, 37]. These protocols provide linearizability across all operations to the system. This paper does not consider Byzantine faults—and largely restricts its consideration of operations affecting single objects— although it is interesting future work to extend chain replication to malicious settings.

**分布式系统中的强一致性。**分布式服务器之间的强一致性可以通过使用主/备份存储 [3] 和两阶段提交协议 [43] 来提供。该领域的早期工作并没有提供面对故障（例如事务管理器）的可用性，这导致引入了视图更改协议（例如，通过领导者共识 [33]）以协助恢复。在这方面有大量后续工作；最近的例子包括链复制和 Guerraoui 等人的基于环的协议。 [25]，它使用两阶段写入协议并在未提交的写入期间延迟读取。与其在任何地方复制内容，不如探索强一致性仲裁系统中重叠读取和写入集之间的其他权衡 [23, 28]。协议协议也已扩展到恶意设置，包括状态机复制 [10, 34] 和仲裁系统 [1, 37]。这些协议为系统的所有操作提供了线性化能力。本文没有考虑拜占庭故障——并且在很大程度上限制了它对影响单个对象的操作的考虑——尽管将链复制扩展到恶意设置是有趣的未来工作。

> There have been many examples of distributed filesystems that provide strong consistency guarantees, such as the early primary/backup-based Harp filesystem [35]. More recently, Boxwood [36] explores exporting various higher-layer data abstractions, such as a B-tree, while offering strict consistency. Sinfonia [2] provides lightweight “mini-transactions” to allow for atomic updates to exposed memory regions in storage nodes, an optimized two-phase commit protocol well-suited for settings with low write contention. CRAQ’s use of optimistic locking for multi-chain multi-object updates was heavily influenced by Sinfonia.

已经有很多分布式文件系统提供强一致性保证的例子，例如早期的基于主/备份的 Harp 文件系统 [35]。 最近，Boxwood [36] 探索导出各种高层数据抽象，例如 B 树，同时提供严格的一致性。 Sinfonia [2] 提供轻量级的“迷你事务”以允许对存储节点中暴露的内存区域进行原子更新，这是一种优化的两阶段提交协议，非常适合低写入争用的设置。 CRAQ 对多链多对象更新的乐观锁定的使用深受 Sinfonia 的影响。

> CRAQ and Chain Replication [47] are both examples of object-based storage systems that expose wholeobject writes (updates) and expose a flat object namespace. This interface is similar to that provided by keyvalue databases [40], treating each object as a row in these databases. As such, CRAQ and Chain Replication focus on strong consistency in the ordering of operations to each object, but does not generally describe ordering of operations to different objects. (Our extensions in §4.1 for multi-object updates are an obvious exception.) As such, they can be viewed in light of casual consistency taken to the extreme, where only operations to the same object are causally related. Causal consistency was studied both for optimistic concurrency control in databases [7] and for ordered messaging layers for distributed systems [8]. Yahoo!’s new data hosting service, PNUTs [12], also provides per-object write serialization (which they call perrecord timeline consistency). Within a single datacenter, they achieve consistency through a messaging service with totally-ordered delivery; to provide consistency across datacenters, all updates are sent to a local record master, who then delivers updates in committed order to replicas in other datacenters.

CRAQ 和链复制 [47] 都是基于对象的存储系统的示例，它们公开整个对象写入（更新）并公开平面对象命名空间。该接口类似于键值数据库 [40] 提供的接口，将每个对象视为这些数据库中的一行。因此，CRAQ 和 Chain Replication 侧重于对每个对象的操作顺序的强一致性，但通常不描述对不同对象的操作顺序。 （我们在 §4.1 中针对多对象更新的扩展是一个明显的例外。）因此，可以根据极端的偶然一致性来看待它们，其中只有对同一对象的操作是因果相关的。因果一致性被研究用于数据库中的乐观并发控制 [7] 和分布式系统的有序消息传递层 [8]。 Yahoo! 的新数据托管服务 PNUTs [12] 还提供按对象写入序列化（他们称之为按记录时间线一致性）。在单个数据中心内，它们通过具有完全有序交付的消息传递服务实现一致性；为了提供跨数据中心的一致性，所有更新都发送到本地记录主，然后该主按提交的顺序将更新传送到其他数据中心的副本。

> The chain self-organization techniques we use are based on those developed by the DHT community [29, 45]. Focusing on peer-to-peer settings, CFS provides a read-only filesystem on top of a DHT [14]; Carbonite explores how to improve reliability while minimizing replica maintenance under transient failures [11]. Strongly-consistent mutable data is considered by OceanStore [32] (using BFT replication at core nodes) and Etna [39] (using Paxos to partition the DHT into smaller replica groups and quorum protocols for consistency). CRAQ’s wide-area solution is more datacenterfocused and hence topology-aware than these systems. Coral [20] and Canon [21] both considered hierarchical DHT designs.

我们使用的链自组织技术基于 DHT 社区开发的技术 [29, 45]。 CFS 专注于点对点设置，在 DHT 之上提供了一个只读文件系统 [14]； Carbonite 探索了如何在提高可靠性的同时最大限度地减少瞬态故障下的副本维护 [11]。 OceanStore [32]（在核心节点使用 BFT 复制）和 Etna [39]（使用 Paxos 将 DHT 划分为更小的副本组和仲裁协议以保持一致性）考虑了强一致性可变数据。 CRAQ 的广域解决方案更侧重于数据中心，因此比这些系统具有拓扑感知能力。 Coral [20] 和 Canon [21] 都考虑了分层 DHT 设计。

> **Weakening Consistency for Availability. ** TACT [49] considers the trade-off between consistency and availability, arguing that weaker consistency can be supported when system constraints are not as tight. eBay uses a similar approach: messaging and storage are eventuallyconsistent while an auction is still far from over, but use strong consistency—even at the cost of availability—right before an auction closes [46].

**弱化可用性的一致性。 ** TACT [49] 考虑了一致性和可用性之间的权衡，认为当系统约束不那么严格时可以支持较弱的一致性。 eBay 使用类似的方法：当拍卖还远未结束时，消息和存储最终是一致的，但在拍卖结束之前使用强一致性——即使以可用性为代价——[46]。

> A number of filesystems and object stores have traded consistency for scalability or operation under partitions. The Google File System (GFS) [22] is a cluster-based object store, similar in setting to CRAQ. However, GFS sacrifices strong consistency: concurrent writes in GFS are not serialized and read operations are not synchronized with writes. Filesystems designed with weaker consistency semantics include Sprite [6], Coda [30], Ficus [27], and Bayou [42], the latter using epidemic protocols to perform data reconciliation. A similar gossip-style antientropy protocol is used in Amazon’s Dynamo object service [15], to support “always-on” writes and continued operation when partitioned. Facebook’s new Cassandra storage system [16] also offers only eventual consistency. The common use of memcached [18] with a relational database does not offer any consistency guarantees and instead relies on correct programmer practice; maintaining even loose cache coherence across multiple datacenters has been problematic [44].

许多文件系统和对象存储已经用一致性来换取分区下的可扩展性或操作。 Google 文件系统 (GFS) [22] 是一个基于集群的对象存储，类似于 CRAQ 的设置。但是，GFS 牺牲了强一致性：GFS 中的并发写入没有序列化，读取操作与写入不同步。使用较弱一致性语义设计的文件系统包括 Sprite [6]、Coda [30]、Ficus [27] 和 Bayou [42]，后者使用流行病协议来执行数据协调。亚马逊的 Dynamo 对象服务 [15] 中使用了类似的八卦式反熵协议，以支持“永远在线”的写入和分区时的持续操作。 Facebook 的新 Cassandra 存储系统 [16] 也仅提供最终一致性。 memcached [18] 与关系数据库的共同使用不提供任何一致性保证，而是依赖于正确的程序员实践；跨多个数据中心保持松散的缓存一致性一直存在问题[44]。

> CRAQ’s strong consistency protocols do not support writes under partitioned operation, although partitioned chain segments can fall back to read-only operation. This trade-off between consistency, availability, and partition tolerance was considered by BASE [19] and Brewer’s CAP conjecture [9].

CRAQ 的强一致性协议不支持分区操作下的写入，尽管分区链段可以回退到只读操作。 BASE [19] 和 Brewer 的 CAP 猜想 [9] 考虑了一致性、可用性和分区容错性之间的这种权衡。

## 8 Conclusions

> This paper presented the design and implementation of CRAQ, a successor to the chain replication approach for strong consistency. CRAQ focuses on scaling out read throughput for object storage, especially for read-mostly workloads. It does so by supporting apportioned queries: that is, dividing read operations over all nodes of a chain, as opposed to requiring that they all be handled by a single primary node. While seemingly simple, CRAQ demonstrates performance results with significant scalability improvements: proportional to the chain length with little write contention—i.e., 200% higher throughput with three-node chains, 600% with seven-node chains—and,somewhat surprisingly, still noteworthy throughput improvements when object updates are common.

本文介绍了 CRAQ 的设计和实现，CRAQ 是强一致性链复制方法的继承者。 CRAQ 专注于扩展对象存储的读取吞吐量，特别是对于以读取为主的工作负载。 它通过支持分配查询来实现这一点：也就是说，将读取操作划分到链的所有节点上，而不是要求它们都由单个主节点处理。 虽然看似简单，但 CRAQ 展示了具有显着可扩展性改进的性能结果：与链长度成正比，写入争用很少——即，三节点链的吞吐量提高 200%，七节点链的吞吐量提高 600%——而且，有点令人惊讶，仍然值得注意 对象更新很常见时的吞吐量改进。

> Beyond this basic approach to improving chain replication, this paper focuses on realistic settings and requirements for a chain replication substrate to be useful across a variety of higher-level applications. Along with our continued development of CRAQ for multi-site deployments and multi-object updates, we are working to integrate CRAQ into several other systems we are building that require reliable object storage. These include a DNS service supporting dynamic service migration, rendezvous servers for a peer-assisted CDN [5], and a largescale virtual world environment. It remains as interesting future work to explore these applications’ facilities in using both CRAQ’s basic object storage, wide-area optimizations, and higher-level primitives for single-key and multi-object updates.

除了这种改进链复制的基本方法之外，本文还重点介绍了链复制基板的实际设置和要求，以便在各种更高级别的应用程序中发挥作用。 随着我们持续开发用于多站点部署和多对象更新的 CRAQ，我们正在努力将 CRAQ 集成到我们正在构建的需要可靠对象存储的其他几个系统中。 其中包括支持动态服务迁移的 DNS 服务、用于对等辅助 CDN [5] 的集合服务器以及大型虚拟世界环境。 在使用 CRAQ 的基本对象存储、广域优化和用于单键和多对象更新的更高级别原语方面，探索这些应用程序的功能仍然是有趣的未来工作。

## Acknowledgments

The authors would like to thank Wyatt Lloyd, Muneeb Ali, Siddhartha Sen, and our shepherd Alec Wolman for helpful comments on earlier drafts of this paper. We also thank the Flux Research Group at Utah for providing access to the Emulab testbed. This work was partially funded under NSF NeTS-ANET Grant #0831374.

## 参考文献

[1] M. Abd-El-Malek, G. Ganger, G. Goodson, M. Reiter, and J. Wylie. Fault-scalable Byzantine fault-tolerant services. In Proc. Symposium on Operating Systems Principles (SOSP), Oct. 2005.

[2] M. K. Aguilera, A. Merchant, M. Shah, A. Veitch, and C. Karamanolis. Sinfonia: a new paradigm for building scalable distributed systems. In Proc. Symposium on Operating Systems Principles (SOSP), Oct. 2007.

[3] P. Alsberg and J. Day. A principle for resilient sharing of distributed resources. In Proc. Intl. Conference on Software Engineering, Oct. 1976.

[4] Amazon. S3 Service Level Agreement. http://aws. amazon.com/s3sla/, 2009.

[5] C. Aperjis, M. J. Freedman, and R. Johari. Peer-assisted content distribution with prices. In Proc. SIGCOMM Conference on Emerging Networking Experiments and Technologies (CoNEXT), Dec. 2008.

[6] M. Baker and J. Ousterhout. Availability in the Sprite distributed file system. Operating Systems Review, 25(2), Apr. 1991.

[7] P. A. Bernstein and N. Goodman. Timestamp-based algorithms for concurrency control in distributed database systems. In Proc. Very Large Data Bases (VLDB), Oct. 1980.

[8] K. P. Birman. The process group approach to reliable distributed computing. Communications of the ACM, 36(12), 1993.

[9] E. Brewer. Towards robust distributed systems. Principles of Distributed Computing (PODC) Keynote, July 2000.

[10] M. Castro and B. Liskov. Practical Byzantine fault tolerance. In Proc. Operating Systems Design and Implementation (OSDI), Feb. 1999.

[11] B.-G. Chun, F. Dabek, A. Haeberlen, E. Sit, H. Weatherspoon, F. Kaashoek, J. Kubiatowicz, and R. Morris. Efficient replica maintenance for distributed storage systems. In Proc. Networked Systems Design and Implementation (NSDI), May 2006.

[12] B. F. Cooper, R. Ramakrishnan, U. Srivastava, A. Silberstein, P. Bohannon, H.-A. Jacobsen, N. Puz, D. Weaver, and R. Yerneni. PNUTS: Yahoo!’s Hosted Data Serving Platform. In Proc. Very Large Data Bases (VLDB), Aug. 2008.

[13] CouchDB. http://couchdb.apache.org/, 2009.

[14] F. Dabek, M. F. Kaashoek, D. Karger, R. Morris, and I. Stoica. Wide-area cooperative storage with CFS. In Proc. Symposium on Operating Systems Principles (SOSP), Oct. 2001.

[15] G. DeCandia, D. Hastorun, M. Jampani, G. Kakulapati, A. Lak-shman, A. Pilchin, S. Sivasubramanian, P. Vosshall, and W. Vogels. Dynamo: Amazon’s highly available key-value store. In Proc. Symposium on Operating Systems Principles (SOSP), Oct. 2007.

[16] Facebook. Cassandra: A structured storage system on a P2P network. http://code.google.com/p/ the-cassandra-project/, 2009.

[17] Facebook. Infrastructure team. Personal Comm., 2008.

[18] B. Fitzpatrick. Memcached: a distributed memory object caching system. http://www.danga.com/ memcached/, 2009.

[19] A. Fox, S. D. Gribble, Y. Chawathe, E. A. Brewer, and P. Gauthier. Cluster-based scalable network services. In Proc. Symposium on Operating Systems Principles (SOSP), Oct. 1997.

[20] M. J. Freedman, E. Freudenthal, and D. Mazières. Democratizing content publication with Coral. In Proc. Networked Systems Design and Implementation (NSDI), Mar. 2004.

[21] P. Ganesan, K. Gummadi, and H. Garcia-Molina. Canon in G Major: Designing DHTs with hierarchical structure. In Proc. Intl. Conference on Distributed Computing Systems (ICDCS), Mar. 2004.

[22] S. Ghemawat, H. Gobioff, and S.-T. Leung. The google file system. In Proc. Symposium on Operating Systems Principles (SOSP), Oct. 2003.

[23] D. K. Gifford. Weighted voting for replicated data. In Proc. Symposium on Operating Systems Principles (SOSP), Dec. 1979.

[24] Google. Google Apps Service Level Agreement. http://www.google.com/apps/intl/en/ terms/sla.html, 2009.

[25] R. Guerraoui, D. Kostic, R. R. Levy, and V. Quéma. A high throughput atomic storage algorithm. In Proc. Intl.Conference on Distributed Computing Systems (ICDCS), June 2007.

[26] D. Hakala. Top 8 datacenter disasters of 2007. IT Management, Jan. 28 2008.

[27] J.HeidemannandG.Popek.Filesystemdevelopmentwith stackable layers. ACM Trans. Computer Systems, 12(1), Feb. 1994.

[28] M. Herlihy. A quorum-consensus replication method for abstract data types. ACM Trans. Computer Systems, 4(1), Feb. 1986.

[29] D. Karger, E. Lehman, F. Leighton, M. Levine, D. Lewin, and R. Panigrahy. Consistent hashing and random trees: Distributed caching protocols for relieving hot spots on the World Wide Web. In Proc. Symposium on the Theory of Computing (STOC), May 1997.

[30] J. Kistler and M. Satyanarayanan. Disconnected operation in the Coda file system. ACM Trans. Computer Systems, 10(3), Feb. 1992.

[31] M. Krohn, E. Kohler, and M. F. Kaashoek. Events can make sense. In Proc. USENIX Annual Technical Conference, June 2007.

[32] J. Kubiatowicz, D. Bindel, Y. Chen, S. Czerwinski, P. Eaton, D. Geels, R. Gummadi, S. Rhea, H. Weatherspoon, W. Weimer, C. Wells, and B. Zhao. OceanStore: An architecture for global-scale persistent storage. In Proc. Architectural Support for Programming Languages and Operating Systems (ASPLOS), Nov 2000.

[33] L. Lamport. The part-time parliament. ACM Trans. Computer Systems, 16(2), 1998.

[34] L. Lamport, R. Shostak, and M. Pease. The Byzantine generals problem. ACM Trans. Programming Language Systems, 4(3), 1982.

[35] B. Liskov, S. Ghemawat, R. Gruber, P. Johnson, L. Shrira, and M. Williams. Replication in the harp file system. In Proc. Symposium on Operating Systems Principles (SOSP), Aug. 1991.

[36] J. MacCormick, N. Murphy, M. Najork, C. A. Thekkath, and L. Zhou. Boxwood: Abstractions as the foundation for storage infrastructure. In Proc. Operating Systems Design and Implementation (OSDI), Dec. 2004.

[37] D. Malkhi and M. Reiter. Byzantine quorum systems. In Proc. Symposium on the Theory of Computing (STOC), May 1997.

[38] D. Mazières, M. Kaminsky, M. F. Kaashoek, and E. Witchel. Separating key management from file system security. In Proc. Symposium on Operating Systems Principles (SOSP), Dec 1999.

[39] A. Muthitacharoen, S. Gilbert, and R. Morris. Etna: a fault-tolerant algorithm for atomic mutable DHT data. Technical Report MIT-LCS-TR-993, MIT, June 2005.

[40] Oracle. BerkeleyDB v4.7, 2009.

[41] C. Patridge, T. Mendez, and W. Milliken. Host anycasting service. RFC 1546, Network Working Group, Nov. 1993.

[42] K. Petersen, M. Spreitzer, D. Terry, M. Theimer, , and A. Demers. Flexible update propagation for weakly consistent replication. In Proc. Symposium on Operating Systems Principles (SOSP), Oct. 1997.

[43] D. Skeen. A formal model of crash recovery in a distributed system. IEEE Trans. Software Engineering, 9(3), May 1983.

[44] J. Sobel. Scaling out. Engineering at Facebook blog, Aug. 20 2008.

[45] I. Stoica, R. Morris, D. Liben-Nowell, D. Karger, M. F. Kaashoek, F. Dabek, and H. Balakrishnan. Chord: A scalable peer-to-peer lookup protocol for Internet applications. IEEE/ACM Trans. Networking, 11, 2002.

[46] F. Travostino and R. Shoup. eBay’s scalability odyssey: Growing and evolving a large ecommerce site. In Proc. Large-Scale Distributed Systems and Middleware (LADIS), Sept. 2008.

[47] R. van Renesse and F. B. Schneider. Chain replication for supporting high throughput and availability. In Proc. Operating Systems Design and Implementation (OSDI), Dec. 2004.

[48] Yahoo! Hadoop Team. Zookeeper. http://hadoop. apache.org/zookeeper/, 2009.

[49] H. Yu and A. Vahdat. The cost and limits of availability for replicated services. In Proc. Symposium on Operating Systems Principles (SOSP), Oct. 2001.
