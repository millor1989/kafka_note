### Kafka配置

#### 1、重要的客户端配置

最重要的生产者配置是：

* acks
* compression
* batch size

最重要的消费者配置是fetch size。  
所有配置的[文档](https://kafka.apache.org/documentation/#configuration)

#### 2、生产环境服务器配置

生产环境服务器配置的例子：

```bash
  # ZooKeeper
  zookeeper.connect=[list of ZooKeeper servers]

  # Log configuration
  num.partitions=8
  default.replication.factor=3
  log.dir=[List of directories. Kafka should have its own dedicated disk(s) or SSD(s).]

  # Other configurations
  broker.id=[An integer. Start with 0 and increment by 1 for each new broker.]
  listeners=[list of listeners]
  auto.create.topics.enable=false
  min.insync.replicas=2
  queued.max.requests=[number of concurrent requests]
```

客户端配置不同应用场景之间差异很大。

