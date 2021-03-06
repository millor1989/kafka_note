### 数据中心

某些部署可能需要管理一个跨多个数据中心的数据管道。推荐方案是，在每个数据中心内部部署一个本地的Kafka集群，每个数据中心内部的应用实例只和它们本地的集群交互，并且在集群间进行镜像。

这种部署模式使得数据中心可以作为独立地个体，用户可以集中地管理和调试数据中心内部的备份。这让每个数据中心都相互独立并能够单独进行操作，即使某个数据中心内部链接不可用；当发生这种情况时，镜像会落后，直到链接恢复才会赶上来。

对于需要全局的所有数据的应用，可以使用镜像提供具有来自所有数据中心本地集群镜像的聚合数据的集群。这些聚合集群是用于需要全部数据集的应用的读取的。

这不是唯一可能的部署模式。还可以通过WAN（Wide area network）读或写一个远程的Kafka集群，但是，很明显会有获取集群的延时。

Kafka本质上会在生产者和消费者上批量处理数据，所以即使通过高延迟的连接也能达到高吞吐量。但是，可能有必要为生产者和消费者以及代理增加TCP socket的buffer大小，使用`socket.send.buffer.bytes`和`scoket.receive.buffer.bytes`配置。设置的恰当方式参考[文档](https://en.wikipedia.org/wiki/Bandwidth-delay_product)

一般不推荐运行一个高延时的跨多个数据中心的Kafka集群。这会导致Kafka写操作和ZooKeeper写操作的高延时，如果位置间的网络不可用，Kafka和ZooKeeper将在所有位置都不可用。