#### 1、获取Kafka

[下载](https://www.apache.org/dyn/closer.cgi?path=/kafka/2.6.0/kafka_2.13-2.6.0.tgz)最新的Kafka版本解压：

```bash
$ tar -xzf kafka_2.13-2.6.0.tgz
$ cd kafka_2.13-2.6.0
```

#### 2、启动Kafka环境

注意：本地环境必须有Java 8+

按顺序运行如下命令以用正确的顺序启动所有的服务：

```bash
# Start the ZooKeeper service
# Note: Soon, ZooKeeper will no longer be required by Apache Kafka.
$ bin/zookeeper-server-start.sh config/zookeeper.properties
```

打开另一个终端会话并运行：

```bash
# Start the Kafka broker service
$ bin/kafka-server-start.sh config/server.properties
```

一旦所有的服务成功启动，就拥有的一个运行的基础的Kafka环境并且可以使用。

#### 3、创建一个主题来保存事件

Kafka是一个分布式的事件流平台，可以用来跨许多机器读、写、处理事件。

事件例子是付款交易、移动电话位置更新、航运订单、IoT设备或医疗设施的传感器测量值、等等。这些事件被组织并保存在主题中。主题类似于文件系统的文件夹，事件是文件家中的文件。

写事件之前，必须创建主题。打开会话终端运行：

```bash
$ bin/kafka-topics.sh --create --topic quickstart-events --bootstrap-server localhost:9092
```

所有的Kafka命令行工具都有额外的选项：没有任何参数的`kafka-topics.sh`命令会展示使用信息。例如，它也能显示新主题的详细信息比如分区数量：

```bash
$ bin/kafka-topics.sh --describe --topic quickstart-events --bootstrap-server localhost:9092
Topic:quickstart-events  PartitionCount:1    ReplicationFactor:1 Configs:
    Topic: quickstart-events Partition: 0    Leader: 0   Replicas: 0 Isr: 0
```

#### 4、往主题中写入事件

Kafka客户端通过网络和Kafka代理通信来写（读）事件。代理收到事件后，会把它以持久的并且是容错的方式根据需要的保存时间（可以是永远）进行保存。

运行控制台生产者客户端写一些事件到主题。默认情况下，输入的每行都会被作为独立的事件写入主题：

```bash
$ bin/kafka-console-producer.sh --topic quickstart-events --bootstrap-server localhost:9092
This is my first event
This is my second event
```

可以随时使用`Ctrl-C`来停止生产者客户端。

#### 5、读取事件

打开新的会话终端运行控制台消费者客户端读取事件：

```bash
$ bin/kafka-console-consumer.sh --topic quickstart-events --from-beginning --bootstrap-server localhost:9092
This is my first event
This is my second event
```

可以随时使用`Ctrl-C`来停止消费者客户端。

事件是持久的存储在Kafka中的，可以随时被任意数量的消费者读取。

#### 6、使用Kafka Connect导入/导出数据为事件的流

既存系统（关系型数据库或者传统的消息系统）中可能有很多数据，并且很多应用在使用这些系统。Kafka Connect可以连续的从外部系统往Kafka摄取数据，反之亦然。因此将既存系统与Kafka集成很简单。为了使这个过程更简单，Kafka社区提供了数百种这种可用的连接器。

#### 7、使用Kafka Streams处理事件

如果数据作为事件存储在Kafka中，可以使用Kafka Streams的Java/Scala客户端库对数据进行处理。可以用来实现关键任务的实时应用和微服务，输入、输出数据保存在Kafka主题中。，Kafka Streams结合了容易在客户端侧编写和部署标准的Java和Scala应用和Kafka的服务器侧集群技术的益处，使得应用高可扩展、弹性、容错、并且是分布式的。这个库支持exactly-once处理，有状态的操作和聚合、windowing、连接、基于事件时间的处理、等等。

Kafka Streams的小尝试，实现`WordCount`算法：

```java
KStream<String, String> textLines = builder.stream("quickstart-events");

KTable<String, Long> wordCounts = textLines
            .flatMapValues(line -> Arrays.asList(line.toLowerCase().split(" ")))
            .groupBy((keyIgnored, word) -> word)
            .count();

wordCounts.toStream().to("output-topic", Produced.with(Serdes.String(), Serdes.Long()));
```

#### 8、关闭Kafka环境

1. 使用`Ctrl-C`停止生产者和消费者客户端。
2. 使用`Ctrl-C`停止Kafka代理。
3. 最后，使用`Ctrl-C`停止ZooKeeper服务器。

如果要删除本地Kafka环境的所有数据，包括创建的事件，运行如下命令：

```bash
$ rm -rf /tmp/kafka-logs /tmp/zookeeper
```



