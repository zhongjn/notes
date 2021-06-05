# Windows Azure Storage

分为 Stream、Partition 两层。Stream 层类似 GFS，提供可追加写的流；Partition 层类似 BigTable，在流之上提供高级语义，如表、队列、对象等。

## 1 组织层级

1. Region，代表一个大的地区，如亚洲、北美、欧洲等。包含若干个 Storage Stamp。
2. Storage Stamp，代表一个集群。包含若干个机柜（rack），机柜之间资源独立，尽量保证不同时发生错误。
3. 机柜。包含若干个服务器。

## 1 Stream Layer

### 1.1 API

Stream 提供了原子追加写、随机读、序列读的 API。（注意，读的模式存在限制，但恰能满足上层需求）

### 1.2 组织

一个 Storage Stamp 中运行了一个 Stream Layer 集群。每一个 Stream Layer 集群能够服务很多条 Stream。每个 Stream Layer 集群都由一个 Stream Manager（使用 Paxos 防单点错误）、若干个 Extent Node 组成。它们对应了 GFS 中的 Master 和 Chunk Server。

一个 Stream 被切分为许多 Extent（考虑 GFS 中的 Chunk），长度可以在1GB左右。

Extent 可以由 Stream Manager 密封，意味着 Extent 再也不会被修改，且读操作将看到一致的结果；Stream 中只有最后一个 Extent 可以在不密封的状态。Extent 存储在哪些机器上、长度、是否密封等元数据由 Stream Manager 存储。

每个未密封的 Extent 存储在3台 Extent Node 机器上，其中1台是主机器，2台是从机器，机器分配永远不会改变。而密封的 Extent 则可以在任意机器上摆放，对于热数据通常要求3副本，冷数据则可以用纠删码编码。

在 Extent Node 上，每个 Extent 以一个 NTFS 文件的形式存在，每个 Extent 又分为许多 Block，Block 元数据存储在 Extent Node 本地。

### 1.3 追加写

#### 1.3.1 语义

客户端可以对于一个 Stream 进行追加写，写入一个或多个 Block。追加写提供 All-or-nothing 原子性、至少一次的语义。

#### 1.3.2 流程

客户端发起追加写时，首先向 Stream Manager 查询该 Stream 的元信息，即当前应该写入哪个 Extent、该 Extent 的元信息。这些元信息可以被客户端缓存以减小 Stream Manager 压力；缓存不一致是不重要的，我们将在 Extent Node 回应错误时进行恰当的重新查询。

在查询到 应该写入的 Extent 的元信息后，客户端联系主节点进行追加写。主节点将协调剩下追加写的过程。主节点对于追加写操作进行序列化（排出全序）。对于当前追加写，在自身、2从节点上进行写入落盘，3机器全部落盘成功后向客户端回复成功（提交）；提交消息的返回对于但客户端是顺序的，因此客户端可以看到单调递增的提交长度；提交长度内的读是一致的。

如果3副本中出现任意一个写入失败，则认为当前追加写失败，并立刻密封（！）当前 Extent。为了满足密封后的 Extent 读一致的要求，我们需要为此 Extent 决定正确的内容。我们选择剩余可达副本中最小的写入长度作为提交长度，这确保向客户端应答提交的数据不丢失。

注1：这里有个 tricky 的情况，即对于主副本不可达的从副本，对于客户端可能仍然可达，导致数据长度小于提交长度，出现读不一致。我们需要考虑 Partiton Layer 的两种读行为：第一种是随机读已提交的位置，客户端不认为最后的不一致区域已提交所以不会读；第二种是序列读，我们限制它只能在已密封的 Extent 上进行。

注2：当前 Extent 因为机器错误密封后，下一个 Extent 将会被开启，指定不同的3副本机器，也就取缔了出现错误的机器。

### 1.4 随机读、序列读

比较简单，此处省略。追加写中的注1中记录了一个 tricky 的情况。

### 1.5 其他

- 每台机器上使用独立的硬盘来做预写日志，避免磁头竞争，稳定写入、提交延迟。（这是机械硬盘，对于 SSD 还需要这样吗？）

- 读请求指定一个期限，副本认为不能满足时立刻拒绝，让客户端转向其他副本。

- 对于冷的 Extent，不再进行3副本存储（消耗空间过大），转而使用辅以纠删码的单副本存储，空间消耗仅为 Extent 大小的1.5倍左右。

## 2 Partition Layer

### 2.1 API

向用户提供四种数据组织：Blob、Table、Queue。所有数据位于统一命名空间下，由 AccountName、PartitionName、ObjectName 三个键一并作为主键。AccountName 为用户账户，PartitionName 为分区名（由具体数据组织要求，相同则一定在同一台机器上），ObjectName 为可选的数据标识符。通常，如果上层数据组织要求跨对象事务（如 Table 中的一些行、Queue 中的单个队列），则必须要把这些数据放在同一个 PartitionName 下。

### 2.2 数据组织

基于 Stream 原子追加写的 LSM 树。与 BigTable 中较为类似，此处省略。

### 2.3 分区策略

常规的基于 Range 的划分。注意一个 PartitionName 只能出现在一个 Range 中。

## 3 个人想法

1. 引入不可变，降低系统复杂度。（Stream Layer）

2. 考虑使用方的使用特征，避免提供不必要的特性，降低系统复杂度。（Partition Layer 对 Stream Layer 的读特征）
