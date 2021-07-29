### 持久性

#### 1、文件系统

Kafka严重依赖文件系统来存储和缓存消息。直觉上“disks are slow”，但是合理设计的硬盘持久化框架也可以与网络一样快。

关于硬盘性能的关键事实是，最近十年，硬盘的吞吐量与硬盘寻址延时发生了偏离。结果是，线性写6个7200rpm的SATA RAID-5阵列配置的磁盘簇的性能是大约600MB/秒，但是随机写的性能只有大约100k/秒——相差超过6000倍。这种线性读和写是所有使用模式中最可预测的，并且被操作系统深度地优化过。当今的操作系统提供了预读（read-ahead）迟写（write-behind）的技术，以大块倍数预取（prefetch）数据，并且将较小的逻辑写入分组为较大的物理写入。结果是，顺序硬盘访问在某些情况下可能比随机内存访问更快。

为了弥补性能偏差，当今的操作系统在使用主内存进行硬盘缓存方面变得越来越激进。当内存回收（reclaimed）时，当今的OS很乐意转移所有的空闲内存进行硬盘缓存，而性能损失很小。所有的硬盘读写都会通过这个统一的缓存进行。这个特性不可能在不使用direct I/O的情况下被轻易的关闭，所以，即使一个进程维护了一个进程中的数据缓存，这个数据在OS的pagecache中可能是重复的，即将所有数据存储了两次。

此外，Kafka是基于JVM的，了解Java内存使用的都知道：

1. 对象的内存开销很高，通常是保存的数据的大小的两倍或更多
2. 随着堆中数据的增加，Java垃圾收集会变得越来越慢并且难以捉摸（fiddly）

所以，使用**文件系统并依赖pagecache**比维护一个在内存的缓存或者其它架构更具优势——通过具有自动访问所有空闲内存Kafka至少使可用的缓存加倍了，并且通过**存储一个紧凑的字节结构**而不是单个的对象缓存能力可能再次加倍。这样，在没有GC损失的情况下，在32GB的机器上缓存可能高达28-30GB。另外，即使服务重启，缓存仍然是热的（warm），而服务重启后进程中缓存则需要重建（对于10GB缓存可能消耗10分钟）或者重启后使用的是一个完全冷掉的（cold）缓存（这样初始性能会很糟糕）。这也极大地简化了代码，所有维护缓存和文件系统间连贯性的逻辑都在OS中实现了，并且OS实现倾向于更加高效并且比一次性的进程中尝试更加正确。如果硬盘的使用上支持线性读取，那么预读将在每次读取的硬盘上有效地预先填充有用的数据。

这指明了一个非常简单的设计：不用维护尽可能多的内存并在内存空间耗尽时在慌乱中将它刷出到文件系统。所有的数据被直接写到文件系统中的持久化日志中，不必刷出到硬盘。这意味着，它会被传输到操作系统的pagecache中。

#### 2、恒定时间足够

消息系统使用的持久化数据结构通常是每个消费者一个的队列，带有相关BTree或者其它通用目的的随机访问数据结构，以维护有关消息的元数据。BTrees是最通用的数据结构，使得它可以支持消息系统中多种多样的事务性的和非事务性的语义。但是，BTrees开销是相当高的：BTree操作时间复杂度是O(log N)。一般O(log N)被认为本质上是等价于恒定时间的，但是这对于硬盘操作来说是不真实的。硬盘寻址以10ms的速度弹出，并且每个硬盘每次只能进行一次寻址，所以并行性是受限的。因此，即使是少数几个硬盘寻址也会导致很高的开销。因为存储系统由很快的缓存操作和很慢的物理硬盘操作构成，缓存固定情况下，随着数据增加，观测到的树结构的性能通常是超线性的——即两倍的数据会导致比两倍操作时间更长。

直观上，**持久化队列**可以构建在简单的读和追加文件上，这也是日志记录方案的通常情况。这种结构具有操作时间复杂度为O(1)并且不会阻塞写或者相互阻塞的优势。这具有明显得性能优势，因为性能与数据量大小是完全解耦的——一台服务器可以充分利用一些廉价的、低转速的1+TB SATA硬盘。尽管寻址性能较差，对于大量数据读写这些硬盘具有可接受的性能，价格为1/3，并且容量为3倍。

具有没有性能损失的，对虚拟地无限制的硬盘空间的访问权限意味着可以提供某些消息系统不具有特性。例如，在Kafka中，不用在消息被消费后立即删除消息，可以相对长时间的保留消息。这对与消费者来说具有相当大的灵活性。