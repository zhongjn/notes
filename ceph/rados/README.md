# RADOS

## 核心优势

机器之间以一种近似 P2P 的方式协作。

- P2P 交换集群信息，管理集群信息的单点压力小。
- P2P 地进行迁移，并行性很高。

## 对象定位

- PG + CRUSH 两层索引（类似哈希）。
- PG 聚合一部分对象（减少元数据量），共有2的幂次个，可翻倍并增量迁移（迁移机制？）。
- CRUSH 负责 PG 摆放，随机器上线下线增量改变（类似一致性哈希）。

## 集群信息

- 通过 Monitor 维护真实集群信息。
- 存储机器（OSD）从 Monitor 获得集群信息时，P2P 地向同伴（共享PG）增量交换集群信息，有点像路由协议。

## 复制策略

- 主从复制。客户端写主，主再写从；从写完时通知主，主确认全部从写完后提交。读主。
  - 序列化点：主。
  - 优点：写从可并行。

- 链式复制。客户端写链头，每个链上机器负责复制到下一个机器，链尾写完提交。读链尾。
  - 序列化点：链尾。
  - 优点：单机器网络开销小，读写负载分离。可以加入 CRAQ 优化，链上所有节点可读。

- 混合复制。主从、链式的混合版本。客户端写主，主写所有从；非尾从写完时通知尾从，尾从收到全部通知后提交。读链尾。
  - 序列化点：链尾的从。
  - 优点：写从可并行，读写负载分离。

## 迁移流程

当一个 PG 摆放的机器集合变化时，可能是因为网络故障、CRUSH 算法作用等，此时就产生了迁移。迁移由新主机器协调，联系所有旧机器，决定 PG 的真实内容。如果 PG 其他相关机器发现了迁移，它们需要向新主机器发送通知，确保新主机器得知新集群信息，得知迁移。

## 一致性

- PG 迁移时的对更新操作的一致性保证。当 PG 迁移时，新主机器必须先联系所有旧机器来获取 PG 的真实内容。在联系旧机器的过程中，旧机器会发现自身的集群信息过期，从而停止服务被迁移 PG 上的操作。这确保了旧机器停止 PG 服务严格发生在新机器开始 PG 服务前。
- 机器发生部分网络错误时对读操作的一致性保证。当一个机器对 Monitor、同伴网络不通，对客户端网络通，此时存在集群信息不更新，返回旧数据（已被新机器修改）的可能。因此，对于一个 PG，机器在与同伴通信时获取一个读 Lease（同伴同意），在这期间才能服务读请求。如果 Monitor 发起了 PG 迁移，指定的新机器在联系当前错误机器（旧机器）时，要么联系成功并撤回 Lease，要么联系失败并等到 Lease 时间结束，确保新机器等待时间有限；注意新机器会与错误机器的同伴通信，避免同伴再向错误机器提供读 Lease。