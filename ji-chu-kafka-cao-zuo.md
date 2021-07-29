### 基础Kafka操作

本节描述的所有Kafka工具位于Kafka发行版的`bin/`目录下，无参运行这些工具可以显示这些命令的所有命令行选项。

#### 1、增加和删除主题

可以选择手动增加主题，或者当数据首次发布到一个不存在的主题时自动地创建主题。如果要自动创建主题，调试默认[主题配置](https://kafka.apache.org/documentation/#topicconfigs)。

使用主题工具增加和修改主题：

```bash
  > bin/kafka-topics.sh --bootstrap-server broker_host:port --create --topic my_topic_name \
        --partitions 20 --replication-factor 3 --config x=y
```

**备份因子**控制有多少台服务器来备份写入的每条消息。推荐的备份因子是2或3，以便可以透明地跳转机器而不打断数据消费。

**分区数量**控制主题会被分片到多少个日志中。分区数量的影响：1、每个分区完全适合一台服务器。如果有20个分区，全部数据集（读和写）会由不超过20太服务器处理（不管备份）。最终，分区数量影响了消费者的最大并行度。

每个分片的分区日志会被放到Kafka日志目录下它自己的目录中。目录的名字由主题名称，中间线（-）和分区id组成。由于，以便文件夹名称不能超过255个字符，主题名称会有长度的限制。假设分区数量不超过100,000，那么主题名称不应该超过249个字符，这就给文件夹名称中的中间线和5位数字长的分区id留了足够的空间。

对于像数据保留时长真央的配置，命令行的配置会覆盖默认设置。[主题配置](https://kafka.apache.org/documentation/#topicconfigs)

#### 2、修改主题

可以使用主题工具修改主题的配置或分区。

增加分区：

```bash
  > bin/kafka-topics.sh --bootstrap-server broker_host:port --alter --topic my_topic_name \
        --partitions 40
```

要注意，分区的一个应用场景是给数据分区，增加分区不会改变已经存在的数据的分区，所以如果消费者依赖既存的分区，这个修改会干扰到消费者。即，如果数据按照`hash(key) % number_of_partitions`分区，那么增加分区会把这种分区扰乱，但是Kafka不会尝试自动地重新分发数据。

Kafka目前不支持减少主题分区的个数。

增加备份因子，见[链接](https://kafka.apache.org/documentation/#basic_ops_increase_replication_factor)。

增加配置：

```bash
 > bin/kafka-configs.sh --bootstrap-server broker_host:port --entity-type topics --entity-name my_topic_name --alter --add-config x=y
```

删除配置：

```bash
> bin/kafka-configs.sh --bootstrap-server broker_host:port --entity-type topics --entity-name my_topic_name --alter --delete-config x
```

删除主题：

```bash
  > bin/kafka-topics.sh --bootstrap-server broker_host:port --delete --topic my_topic_name
```

#### 3、优雅的关闭

Kafka集群会自动地侦测到代理关闭或者故障，并且会为那个机器上的分区选举新的leader。当服务器故障或者为了维护或者配置改变而关机时，会发生这种情况。为了后续的访问，Kafka支持一种更加优雅的机制来停止服务器，而不仅是杀死服务器。如果服务器优雅的关闭，会有两点好处：

1. 它会同步它的所有日志到硬盘以避免当它重启的时候（即，验证日志尾中所有消息的校验和）需要进行日志恢复。日志恢复需要时间，所以这能够加速启动过程。
2. 在关闭之前，它会转移将这台服务器作为leader的分区的领导权到其它备份。这会使领导权转换更快，并且将每个分区不可用的时间最小化到几毫秒。

如果服务器是被关闭而不是直接杀死，同步日志的过程会自动的发生。但是，需要一个特殊的配置开启领导权的转移：

```text
      controlled.shutdown.enable=true
```

注意，只有这个代理上的所有分区都有备份（即，备份因子大于1，并且至少一个备份是活跃的）受控制的关闭才会成功。因为，一旦关闭最后的一个备份主题分区将不可用。

#### 4、平衡领导权

当一个代理停止或者故障，这个代理的分区的领导权会转移到其它备份。当这个代理重启，它将会成为它的所有分区的follower，这意味着，它将不会被用于客户端读取和写入。

为了避免这种不平衡，Kafka有一个首选备份（preffered replicas）的概念。如果一个分区的备份列表是1，5，9，那么节点1相对节点5或9的首选leader节点，因为她在备份列表中位置靠前。默认情况下，Kafka集群会尝试恢复已经恢复的备份的领导权。这种行为可以通过如下配置修改：

```reStructuredText
      auto.leader.rebalance.enable=true
```

可以讲这个配置设置为false，这样，就需要手动的恢复备份的领导权，通过如下命令：

```bash
  > bin/kafka-preferred-replica-election.sh --bootstrap-server broker_host:port
```

#### 5、跨机架平衡备份

不同机架相同分区的备份具有机架意识的特性。这将Kafka提供的代理故障的保证扩展到了机架故障，可以避免机架上的所有代理同时故障时的数据丢失。

通过增加一个属性到代理配置可以指定一个代理属于一个特定的机架：

```text
   broker.rack=my-rack-id
```

主题的创建、修改，备份的分发都会遵守机架限制，确保备份分不到尽可能多的机架（分区会分布到`min(#racks,replication-factor)`个不同机架）。

用于分配备份到代理的算法确保每个代理的leaders的数量是一个常量，而不管代理是如何跨机架分布的。这保证了平衡的吞吐量。

但是，如果机架被分配了不同数量的代理，备份的分配就会不均匀。拥有较少代理的机架会有更多的备份，这意味着它们会使用更多的存储并且投入更多的资源到备份。因此，配置每个机架都有相同数量的代理是敏感的。

#### 6、集群间数据镜像

在提到Kafka集群之间的数据备份时使用“镜像”，以避免与在一个集群中节点间发生的备份混淆。Kafka有一个集群间镜像数据的工具。这个工具消费源集群并且生产目标集群。这种镜像的常见应用场景是在另一个数据中心提供一个备份。

可以运行多个这种镜像进程来增加吞吐量和容错（如果一个进程死掉，其它的进程会接管额外的加载任务）。

数据会从源集群的主题读取，并写到目标集群中具有相同名字的主题中。事实上，mirror maker只不过是Kafka生产者和消费者结合到一起。

源集群和目标集群是完全独立的：他们可以有不同数量的分区，并且偏移也不同。因此，镜像集群不是一个容错机制（因为消费者位置不同）；因此，推荐使用普通的集群内备份。但是mirror maker进程会保留并使用消息key来进行分区，所以对于每个key消息的顺序是保留了的。

从输入集群镜像一个主题主题（*my_topic*）的例子：

```bash
  > bin/kafka-mirror-maker.sh
        --consumer.config consumer.properties
        --producer.config producer.properties --whitelist my-topic
```

注意，使用了`--whitelist`选项指定了主题集合。这个选项可以使用Java风格的正则表达式。可以使用`--whitelist 'A|B'`镜像名为*A*和*B*的两个主题。也可以使用`--whitelist '*'`镜像所有的主题。要用单引号包括正则表达式以避免shell将其解释为一个文件路径。为了方便，支持使用`,`代替`|`来指定主题集合。使用配置`auto.create.topics.enable=true`开启合并镜像，可以让备份集群自动创建和备份源集群中新增的主题。

#### 7、检查消费者位置

有时需要查看消费者的位置。有一个显示某一个消费者群组中所有消费者位置和距离日志结尾距离的工具。对消费主题*my-topic*的群组*my-group*运行命令：

```bash
  > bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group my-group

  TOPIC                          PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG        CONSUMER-ID                                       HOST                           CLIENT-ID
  my-topic                       0          2               4               2          consumer-1-029af89c-873c-4751-a720-cefd41a669d6   /127.0.0.1                     consumer-1
  my-topic                       1          2               3               1          consumer-1-029af89c-873c-4751-a720-cefd41a669d6   /127.0.0.1                     consumer-1
  my-topic                       2          2               3               1          consumer-2-42c1abd4-e3b2-425d-a8bb-e1ea49b29bb2   /127.0.0.1                     consumer-2
```

#### 8、管理消费者群组

使用`ConsumerGroupCommand`工具，可以查看、描述、或者删除消费者群组。当为消费群组进行的最后提交的偏移失效后，可以手动或者自动地删除消费者群组。如果群组没有活跃的成员，手动删除才会运行。比如，查看所有主题的消费者群组：

```bash
  > bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --list

  test-consumer-group
```

查看偏移：

```bash
> bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group my-group

  TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID                                    HOST            CLIENT-ID
  topic3          0          241019          395308          154289          consumer2-e76ea8c3-5d30-4299-9005-47eb41f3d3c4 /127.0.0.1      consumer2
  topic2          1          520678          803288          282610          consumer2-e76ea8c3-5d30-4299-9005-47eb41f3d3c4 /127.0.0.1      consumer2
  topic3          1          241018          398817          157799          consumer2-e76ea8c3-5d30-4299-9005-47eb41f3d3c4 /127.0.0.1      consumer2
  topic1          0          854144          855809          1665            consumer1-3fc8d6f1-581a-4472-bdf3-3515b4aee8c1 /127.0.0.1      consumer1
  topic2          0          460537          803290          342753          consumer1-3fc8d6f1-581a-4472-bdf3-3515b4aee8c1 /127.0.0.1      consumer1
  topic3          2          243655          398812          155157          consumer4-117fe4d3-c6c1-4178-8ee9-eb4a3954bee0 /127.0.0.1      consumer4
```

还有一些其它的“describe”选项，可以查看群组的更加详细的信息：

- **--members**：查看消费者群组中的所有活跃成员。

  ```bash
        > bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group my-group --members
  
        CONSUMER-ID                                    HOST            CLIENT-ID       #PARTITIONS
        consumer1-3fc8d6f1-581a-4472-bdf3-3515b4aee8c1 /127.0.0.1      consumer1       2
        consumer4-117fe4d3-c6c1-4178-8ee9-eb4a3954bee0 /127.0.0.1      consumer4       1
        consumer2-e76ea8c3-5d30-4299-9005-47eb41f3d3c4 /127.0.0.1      consumer2       3
        consumer3-ecea43e4-1f01-479f-8349-f9130b75d8ee /127.0.0.1      consumer3       0
  ```

- **--members --verbose**：在“--members”选项提供的信息的基础上，这个选项提供了分配给每个成员的分区的信息

  ```bash
        > bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group my-group --members --verbose
  
        CONSUMER-ID                                    HOST            CLIENT-ID       #PARTITIONS     ASSIGNMENT
        consumer1-3fc8d6f1-581a-4472-bdf3-3515b4aee8c1 /127.0.0.1      consumer1       2               topic1(0), topic2(0)
        consumer4-117fe4d3-c6c1-4178-8ee9-eb4a3954bee0 /127.0.0.1      consumer4       1               topic3(2)
        consumer2-e76ea8c3-5d30-4299-9005-47eb41f3d3c4 /127.0.0.1      consumer2       3               topic2(1), topic3(0,1)
        consumer3-ecea43e4-1f01-479f-8349-f9130b75d8ee /127.0.0.1      consumer3       0               -
  ```

- **--offsets**：默认的“describe”选项，输出与“--describe”选项单独使用相同

- **--state**：这个选项提供了组级别的信息

  ```bash
        > bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group my-group --state
  
        COORDINATOR (ID)          ASSIGNMENT-STRATEGY       STATE                #MEMBERS
        localhost:9092 (0)        range                     Stable               4
  ```

可以使用“**--delete**”选项删除一个或者多个消费者群组：

```bash
  > bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --delete --group my-group --group my-other-group

  Deletion of requested consumer groups ('my-group', 'my-other-group') was successful.
```

可以使用“**--reset-offset**”选项重置消费者群组的偏移。这个选项只支持对一个消费者群组操作。需要定义作用域：“--all-topics”或者“--topic”。必须要选择一种作用域，除非使用了“--from-file”。并且，首先要确保消费者实例是不活跃的。[更多详情](https://cwiki.apache.org/confluence/display/KAFKA/KIP-122%3A+Add+Reset+Consumer+Group+Offsets+tooling)

它有三个执行选项：

- （默认）显示要重置的偏移
- --execute：执行--reset-offset进程
- --export：导出结果到CSV文件

--reset-offset可以选择的场景（必须至少选择一个场景）：

- --to-datetime &lt;String: datetime&gt;：重置偏移至datetime时的偏移。格式’YYYY-MM-DDTHH:mm:SS.sss‘
- --to-earliest：重置偏移到最早的偏移
- --to-latest：重置偏移为最后的偏移
- --shift-by &lt;Long:number-of-offset&gt;：重置偏移，将当前偏移移动’n‘，其中’n‘可以为正也可以为负
- --from-file：重置偏移为CSV文件中定义的值
- --to-current：重置偏移为当前偏移
- --by-duration &lt;String duration&gt;：重置偏移为与当前时间戳相距duration时间段的偏移。格式：’PnDTnHnMnS‘
- --to-offset：重置偏移为指定偏移

要注意，超过偏移的范围会被设置为最后可用的偏移。比如，偏移的结尾是10，移动15个偏移，最终偏移为10.

比如，重置消费者组的偏移为最后的偏移：

```bash
  > bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --reset-offsets --group consumergroup1 --topic topic1 --to-latest

  TOPIC                          PARTITION  NEW-OFFSET
  topic1                         0          0
```

如果使用的是旧版的高层次（high-level）消费者，并把组元数据保存在了ZooKeeper（即，offset.storage=zookeeper），传递`--zookeeper`代替`--bootstrap-server`：

```bash
  > bin/kafka-consumer-groups.sh --zookeeper localhost:2181 --list
```

#### 9、扩展集群

为Kafka集群增加服务器很简单，只用为新的服务器分配一个唯一的代理id并在上面启动Kafka就行了。但是，这些新的服务器不会自动地被分配任何数据分区，除非分区被移动到这些服务器，只有创建了新的主题，这些服务器才会有数据分区。

迁移数据的进程是手动地初始化的，但是是完全自动化的。事实上，Kafka将新的服务器增加为它正在迁移的分区的follower，并允许它完全地备份那个分区的数据。当新的服务器完整地备份了这个分区，并且加入到了in-sync备份中，已经存在的备份中的一个会删除它们分区的数据。

分区重新分配工具可以用来在代理间移动分区。一个理想的分区分布，够确保代理间数据负载和分区大小的均匀。分区重新分配工具不能自动地获取Kafka集群中数据的分布，也不能自动地移动分区并达到均匀地负载分布。因此，管理员必须指出要移动哪个主题或者分区。

分区重新分配工具可以运行在3种互斥的模式：

- **--generate**：这种模式中，指定一个主题集合和代理集合，工具会生成一个候选重新分配任务来将指定主题的分区全部移动到新的代理。这种指定主题集合和目标代理的模式，仅仅是提供了一个生成分区重新分配计划的便捷方式。
- **--execute**：这种模式下，工具基于用户提供的重新分配计划启动分区的重新分配。（使用`--reassignment-json-file`选项）。可以是管理员手动制定的自定义重新分配计划，或者是`--generate`选项生成的计划。
- **--verify**：这种模式下，工具验证最近的`--execute`过程中列出的所有分区的重新分配状态。状态可以是成功完成、失败或者进行中

##### 9.1、自动迁移数据到新的机器

分区重新分配工具可以被用来将某些主题从当前的代理集合移动到新增加的代理。当扩展既存集群的时候是很有用的，因为很容易就能将整个主题移动到新的代理集合，而不是一次移动一个分区。用来进行主题移动时，用户要提供一个要被移动到新的代理集合的主题集合和一个新代理的目标集合。这个工具会将这个主题集合的分区均匀的分发到新的代理集合。在移动过程中，主题备份因子不变。输入主题集合的所有分区都会被高效地从旧的代理移动到新增加的代理。

比如，如下例子会将foo1，foo2的所有分区移动到新的代理集合5，6。移动结束后，foo1和foo2主题的所有分区将会仅仅存在于代理5，6上。

由于，这个工具可以接受json文件格式的输入主题集合，首先把要移动的主题加到创建的json文件如下：

```bash
  > cat topics-to-move.json
  {"topics": [{"topic": "foo1"},
              {"topic": "foo2"}],
  "version":1
  }
```

创建完json文件，使用分区重新分配工具生成一个候选分配计划：

```bash
  > bin/kafka-reassign-partitions.sh --bootstrap-server localhost:9092 --topics-to-move-json-file topics-to-move.json --broker-list "5,6" --generate
  Current partition replica assignment

  {"version":1,
  "partitions":[{"topic":"foo1","partition":2,"replicas":[1,2]},
                {"topic":"foo1","partition":0,"replicas":[3,4]},
                {"topic":"foo2","partition":2,"replicas":[1,2]},
                {"topic":"foo2","partition":0,"replicas":[3,4]},
                {"topic":"foo1","partition":1,"replicas":[2,3]},
                {"topic":"foo2","partition":1,"replicas":[2,3]}]
  }

  Proposed partition reassignment configuration

  {"version":1,
  "partitions":[{"topic":"foo1","partition":2,"replicas":[5,6]},
                {"topic":"foo1","partition":0,"replicas":[5,6]},
                {"topic":"foo2","partition":2,"replicas":[5,6]},
                {"topic":"foo2","partition":0,"replicas":[5,6]},
                {"topic":"foo1","partition":1,"replicas":[5,6]},
                {"topic":"foo2","partition":1,"replicas":[5,6]}]
  }
```

注意，这时只是生成了重新分配计划，分区移动还没有开始。应该把这个分配计划保存，以防万一想要回滚。新的分配计划应该保存在json文件中（例如，expand-cluster-reassignment.json）以作为这个工具使用`--execute`选项的输入，如下：

```bash
  > bin/kafka-reassign-partitions.sh --bootstrap-server localhost:9092 --reassignment-json-file expand-cluster-reassignment.json --execute
  Current partition replica assignment

  {"version":1,
  "partitions":[{"topic":"foo1","partition":2,"replicas":[1,2]},
                {"topic":"foo1","partition":0,"replicas":[3,4]},
                {"topic":"foo2","partition":2,"replicas":[1,2]},
                {"topic":"foo2","partition":0,"replicas":[3,4]},
                {"topic":"foo1","partition":1,"replicas":[2,3]},
                {"topic":"foo2","partition":1,"replicas":[2,3]}]
  }

  Save this to use as the --reassignment-json-file option during rollback
  Successfully started reassignment of partitions
  {"version":1,
  "partitions":[{"topic":"foo1","partition":2,"replicas":[5,6]},
                {"topic":"foo1","partition":0,"replicas":[5,6]},
                {"topic":"foo2","partition":2,"replicas":[5,6]},
                {"topic":"foo2","partition":0,"replicas":[5,6]},
                {"topic":"foo1","partition":1,"replicas":[5,6]},
                {"topic":"foo2","partition":1,"replicas":[5,6]}]
  }
```

最后，可以使用工具用`--verify`选项来验证分区重新分配的状态。注意，此时还需要用到相同的*expand-cluster-reassignment.json*文件：

```bash
  > bin/kafka-reassign-partitions.sh --bootstrap-server localhost:9092 --reassignment-json-file expand-cluster-reassignment.json --verify
  Status of partition reassignment:
  Reassignment of partition [foo1,0] completed successfully
  Reassignment of partition [foo1,1] is in progress
  Reassignment of partition [foo1,2] is in progress
  Reassignment of partition [foo2,0] completed successfully
  Reassignment of partition [foo2,1] completed successfully
  Reassignment of partition [foo2,2] completed successfully
```

##### 9.2、自定义分区分配和迁移

分区重新分配工具还可以选择性地移动一个分区的备份到一个指定的代理集合。这样使用时，它假设用户直到重新分配计划并且不需要用工具来生成一个候选重新分配计划，有效地跳过了`--generate`步骤直接进行`--execute`步骤。

比如，移动foo1主题的分区0到代理5，6，移动foo2主题的分区1到代理2，3：

第一步，手动创建自定义的重新分配计划json文件：

```bash
  > cat custom-reassignment.json
  {"version":1,"partitions":[{"topic":"foo1","partition":0,"replicas":[5,6]},{"topic":"foo2","partition":1,"replicas":[2,3]}]}
```

然后，使用json文件开始执行重新分配进程：

```bash
  > bin/kafka-reassign-partitions.sh --bootstrap-server localhost:9092 --reassignment-json-file custom-reassignment.json --execute
  Current partition replica assignment

  {"version":1,
  "partitions":[{"topic":"foo1","partition":0,"replicas":[1,2]},
                {"topic":"foo2","partition":1,"replicas":[3,4]}]
  }

  Save this to use as the --reassignment-json-file option during rollback
  Successfully started reassignment of partitions
  {"version":1,
  "partitions":[{"topic":"foo1","partition":0,"replicas":[5,6]},
                {"topic":"foo2","partition":1,"replicas":[2,3]}]
  }
```

执行校验：

```bash
  > bin/kafka-reassign-partitions.sh --bootstrap-server localhost:9092 --reassignment-json-file custom-reassignment.json --verify
  Status of partition reassignment:
  Reassignment of partition [foo1,0] completed successfully
  Reassignment of partition [foo2,1] completed successfully
```

#### 10、退役代理

分区重新分配工具没有自动地生成即将退役代理重新分配计划的功能。因此，管理员必须拿出一个移动即将退役代理上所有分区的备份到其它代理的重新分配计划。需要确保退役代理的所有的备份不是被移动到了仅仅某一台代理。为了是这个过程不费力，未来会增加退役代理工具。

#### 11、增加备份因子

增加既存分区的备份因子很简单。只用在自定义的重新分配json文件中指定额外的备份，并用`--execute`选项时使用这个文件，就能增加指定分区的备份因子。

比如，如下例子将主题foo的分区0的备份因子从1增加到了3。增加备份因子之前，分区仅有的备份存在于代理5上。作为增加备份因子的一部分，在代理6和7上增加了备份。

第一步，手动编写自定义的重新分配任务计划json文件：

```bash
  > cat increase-replication-factor.json
  {"version":1,
  "partitions":[{"topic":"foo","partition":0,"replicas":[5,6,7]}]}
```

然后，使用`--execute`执行重新分配进程：

```bash
  > bin/kafka-reassign-partitions.sh --bootstrap-server localhost:9092 --reassignment-json-file increase-replication-factor.json --execute
  Current partition replica assignment

  {"version":1,
  "partitions":[{"topic":"foo","partition":0,"replicas":[5]}]}

  Save this to use as the --reassignment-json-file option during rollback
  Successfully started reassignment of partitions
  {"version":1,
  "partitions":[{"topic":"foo","partition":0,"replicas":[5,6,7]}]}
```

校验：

```bash
  > bin/kafka-reassign-partitions.sh --bootstrap-server localhost:9092 --reassignment-json-file increase-replication-factor.json --verify
  Status of partition reassignment:
  Reassignment of partition [foo,0] completed successfully
```

也可以使用kafka-topic工具校验备份因子的增加：

```bash
  > bin/kafka-topics.sh --bootstrap-server localhost:9092 --topic foo --describe
  Topic:foo	PartitionCount:1	ReplicationFactor:3	Configs:
    Topic: foo	Partition: 0	Leader: 5	Replicas: 5,6,7	Isr: 5,6,7
```

#### 12、限制数据迁移期间的带宽使用

Kafka可以设置一个机器间移动备份时使用的带宽的上限。当集群再平衡、启动新的代理或者增加代理、移除代理时这个设置是很有用的，它限制了数据密集操作对用户的影响。

有两个接口可以用来设置带宽限制。最简单、最安全的是在调用`kafka-reassign-partitions.sh`时应用一个限制，但是`kafka-topics.sh`也可以直接地查看或者修改这个限制。

比如，如果用如下命令执行再平衡，带宽就被限制为50MB/s：

```bash
bin/kafka-reassign-partitions.sh --bootstrap-server localhost:9092 --execute --reassignment-json-file bigger-cluster.json --throttle 50000000
```

执行这个脚本时，就能看到这个限制启用：

```bash
  The throttle limit was set to 50000000 B/s
  Successfully started reassignment of partitions.
```

通过再次运行执行命令，可以修改限制：

```bash
$ bin/kafka-reassign-partitions.sh --bootstrap-server localhost:9092  --execute --reassignment-json-file bigger-cluster.json --throttle 700000000
  There is an existing assignment running.
  The throttle limit was set to 700000000 B/s
```

如果再平衡结束，通过`--verify`选项进行验证可以同时移除带宽限制。在再平衡结束后及时地移除限制是很重要的，如果不移除限制也会导致常规的备份带宽被限制。

当执行了`--verify`选项后，并且重新分配完成后，这个脚本能够确认限制的移除：

```bash
  > bin/kafka-reassign-partitions.sh --bootstrap-server localhost:9092  --verify --reassignment-json-file bigger-cluster.json
  Status of partition reassignment:
  Reassignment of partition [my-topic,1] completed successfully
  Reassignment of partition [mytopic,0] completed successfully
  Throttle was removed.
```

管理员也可以使用`kafka-topics.sh`来验证配置。有两对限制配置用于管理限制进程。第一对是限制的值本身。这个配置是代理级别的，使用动态属性：

```text
    leader.replication.throttled.rate
    follower.replication.throttled.rate
```

第二对是被限制的备份的枚举集合，是针对每个主题进行配置的：

```text
    leader.replication.throttled.replicas
    follower.replication.throttled.replicas
```

这四个配置值会被`kafka-reassign-partitions.sh`自动地应用。

查看这个限制配置：

```bash
  > bin/kafka-configs.sh --describe --bootstrap-server localhost:9092 --entity-type brokers
  Configs for brokers '2' are leader.replication.throttled.rate=700000000,follower.replication.throttled.rate=700000000
  Configs for brokers '1' are leader.replication.throttled.rate=700000000,follower.replication.throttled.rate=700000000
```

这里显示了应用到leader侧和follower侧的限制。默认情况下，两侧限制的吞吐量值是相同的。

查看限制的备份：

```bash
  > bin/kafka-configs.sh --describe --bootstrap-server localhost:9092 --entity-type topics
  Configs for topic 'my-topic' are leader.replication.throttled.replicas=1:102,0:101,
      follower.replication.throttled.replicas=1:101,0:102
```

这里可以看到，leader限制应用到了代理102上的分区1和代理101上的分区0。类似地，follower限制被应用到了代理101上的分区1和代理102上的分区0。

默认情况下`kafka-reassign-partitions.sh`会应用leader限制到再平衡前存在的所有的备份，任何一个都可能是leader。它会把follower限制应用到所有移动的目的备份。所以，如果有一个分区在代理101，102上有备份，被重分配到了代理102、103，对于这个分区leader限制会应用到101，102，follower限制只会应用到103。

如果需要，可以使用`kafka-configs.sh`的`--alter`开关来手动地修改限制配置。

##### 12.1、受限备份的安全使用

使用受限制的备份时需要注意：

*（1）限制的移除*

重新分配完成后应该及时地移除限制（通过运行`kafka-reassign-partitions.sh --verify`）

*（2）确保进展（Ensuring Progress）*

如果限制的设置太低，与写入速率相比，可能会使备份没有进展。当`max(BytesInPerSec) > throttle`时（其中`BytesInPerSec`为指示生产者写入每个代理的吞吐量的指标）会发生这种情况。

在再平衡期间，管理员可以监控备份是否取得了进展，通过使用指标：

```
kafka.server:type=FetcherLagMetrics,name=ConsumerLag,clientId=([-.\w]+),topic=([-.\w]+),partition=([0-9]+)
```

在备份期间，这个延后指标应该不断地减少。如果该指标不减少，管理员应该增加吞吐量限制。

#### 13、设置配额

[如前所述](https://kafka.apache.org/documentation/#design_quotas)，配额覆盖和默认值可以在(user, client-id)，user，client-id级别进行配置。默认情况下，客户端接受一个无限制的配额。可以为每个(user, client-id)，user，client-id组设置自定义的配额。

为(user=user1, client-id=clientA)配置自定义配额：

```bash
  > bin/kafka-configs.sh  --bootstrap-server localhost:9092 --alter --add-config 'producer_byte_rate=1024,consumer_byte_rate=2048,request_percentage=200' --entity-type users --entity-name user1 --entity-type clients --entity-name clientA
  Updated config for entity: user-principal 'user1', client-id 'clientA'.
```

为user=user1配置自定义配额：

```bash
  > bin/kafka-configs.sh  --bootstrap-server localhost:9092 --alter --add-config 'producer_byte_rate=1024,consumer_byte_rate=2048,request_percentage=200' --entity-type users --entity-name user1
  Updated config for entity: user-principal 'user1'.
```

为client-id=clientA配置自定义配额：

```bash
  > bin/kafka-configs.sh  --bootstrap-server localhost:9092 --alter --add-config 'producer_byte_rate=1024,consumer_byte_rate=2048,request_percentage=200' --entity-type clients --entity-name clientA
  Updated config for entity: client-id 'clientA'.
```

通过指定*--entity-default*选项而不是*--entity-name*，可以设置每个(user, client-id)，user，client-id组的默认配额。

为user=userA配置默认client-id配额：

```bash
  > bin/kafka-configs.sh  --bootstrap-server localhost:9092 --alter --add-config 'producer_byte_rate=1024,consumer_byte_rate=2048,request_percentage=200' --entity-type users --entity-name user1 --entity-type clients --entity-default
  Updated config for entity: user-principal 'user1', default client-id.
```

为用户配置默认配额：

```bash
  > bin/kafka-configs.sh  --bootstrap-server localhost:9092 --alter --add-config 'producer_byte_rate=1024,consumer_byte_rate=2048,request_percentage=200' --entity-type users --entity-default
  Updated config for entity: default user-principal.
```

为client-id配置默认配额：

```bash
  > bin/kafka-configs.sh  --bootstrap-server localhost:9092 --alter --add-config 'producer_byte_rate=1024,consumer_byte_rate=2048,request_percentage=200' --entity-type clients --entity-default
  Updated config for entity: default client-id.
```

查看指定(user, client-id)的配额：

```bash
  > bin/kafka-configs.sh  --bootstrap-server localhost:9092 --describe --entity-type users --entity-name user1 --entity-type clients --entity-name clientA
  Configs for user-principal 'user1', client-id 'clientA' are producer_byte_rate=1024,consumer_byte_rate=2048,request_percentage=200
```

查看指定用户的配额：

```bash
  >bin/kafka-configs.sh  --bootstrap-server localhost:9092 --describe --entity-type users --entity-name user1
  Configs for user-principal 'user1' are producer_byte_rate=1024,consumer_byte_rate=2048,request_percentage=200
```

查看指定client-id的配额：

```bash
  > bin/kafka-configs.sh  --bootstrap-server localhost:9092 --describe --entity-type clients --entity-name clientA
  Configs for client-id 'clientA' are producer_byte_rate=1024,consumer_byte_rate=2048,request_percentage=200
```

如果不指定entity name，那么会显示所有指定类型的条目的配额。比如，查看所有用户的配额：

```bash
  > bin/kafka-configs.sh  --bootstrap-server localhost:9092 --describe --entity-type users
  Configs for user-principal 'user1' are producer_byte_rate=1024,consumer_byte_rate=2048,request_percentage=200
  Configs for default user-principal are producer_byte_rate=1024,consumer_byte_rate=2048,request_percentage=200
```

类似地，查看所有(user, client)配额：

```bash
  > bin/kafka-configs.sh  --bootstrap-server localhost:9092 --describe --entity-type users --entity-type clients
  Configs for user-principal 'user1', default client-id are producer_byte_rate=1024,consumer_byte_rate=2048,request_percentage=200
  Configs for user-principal 'user1', client-id 'clientA' are producer_byte_rate=1024,consumer_byte_rate=2048,request_percentage=200
```

通过在代理商设置配额，可以设置所有client-ids的配额。只有在ZooKeeper中没有配置配额覆盖或默认配额的情况下，这些属性才会应用。默认情况下，每个client-id收到的是没有限制的配额。如下设置了每个生产者和消费者client-id默认配额为10MB/sec。

```text
    quota.producer.default=10485760
    quota.consumer.default=10485760
```

注意，这些属性可能会在未来的发行版本中移除。使用`kafka-configs.sh`配置的默认配额比这些属性优先级更高。