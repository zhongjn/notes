# LFS

## 日志摆放策略

将磁盘空间分为1MB左右的 Segments，每个 Segment 支持追加写、回收（非常像闪存）。

## Segment 回收策略

在尝试回收一些 Segment 时，需要挑选最合适的 Segment。我们希望回收冷块为主（！）的、活数据量小的 Segment，这样的回收长期收益最高，产出的冷 Segment 将能长期稳定存在。

因此，对于每个活跃 Segment，以回收收益、代价之比为得分，回收收益为可带来的空闲空间、Segment 内块年龄的乘积，代价基于当前 Segment 内活数据量（即向新 Segment 写的代价）。

这样的策略很好的区别了冷热数据，成功达成了 Segment 的冷热划分（活数据含量的双峰分布）。如果 Segment 的冷热和数据的冷热不对应，将会造成很大的写放大，因为包含冷数据的 Segment 将会被反复回收、写出。
