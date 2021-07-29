### Java 版本

支持Java 8和Java 11。如果开启了TLS Java 11性能会显著地更好，所以强烈推荐（还有一些其它的性能提升：G1GC，CRC32C，压缩字符串，Thread-Local Handshakes等等）。从安全角度，推荐最新发布的补丁版本，因为更老的版本有未修复的安全漏洞。使用基于OpenJDK的Java实现（包括Oracle JDK）运行Kafka的一般参数是：

```bash
  -Xmx6g -Xms6g -XX:MetaspaceSize=96m -XX:+UseG1GC
  -XX:MaxGCPauseMillis=20 -XX:InitiatingHeapOccupancyPercent=35 -XX:G1HeapRegionSize=16M
  -XX:MinMetaspaceFreeRatio=50 -XX:MaxMetaspaceFreeRatio=80 -XX:+ExplicitGCInvokesConcurrent
```

如下，为LinkedIn的最繁忙的集群（高峰）中使用上述Java参数的一个的统计数据，作为参考：

* 60代理
* 50,000分区（备份因子2）
* 800,000消息/每秒的输入
* 300MB/sec输入限制，1GB/sec以上的输出限制

集群中的所有代理都有一个90%大约21ms的GC暂停时间（由于GC线程运行导致的应用线程暂停），少于每秒1个的young GC。

