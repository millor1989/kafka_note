### 第三章、Kafka生产者

除了 Kafka 的内置生产者客户端，Kafka 还提供了二进制连接协议，即可以直接向 Kafka 网络端口发送适当的字节序列。所以，通过支持 Kafka 二进制连接协议，可以实现第三方的生产者客户端。

#### 3.1、概览

不同使用场景，生产者 API 的使用和配置也会不同。某些场景要求低延时、高吞吐量、消息不能丢失或者重复，而另一些场景可以允许少量的消息丢失或重复，延迟也可以高一些。

![Kafka 生产者组件](/assets/1627536367743.png)

如图展示了消息的发送过程。

首先，创建 `ProducerRecord` 对象，它包含了目标主题和要发送的内容，还可以指定键或者分区。发送 `ProducerRecord` 对象时，生产者要先把键和值对象序列化为字节数组以在网络上传输。

序列化后的数据通过网络传给分区器。如果之前在 `ProducerRecord` 对象里指定了分区，那么分区器直接使用指定的分区。如果没有指定分区，分区器会根据 `ProducerRecord` 对象的键来选择分区。生产者发送的消息被添加到一个记录批次中，批次中的消息被发送到相同的主题和分区。有一个独立的线程负责将记录批次发送到对应的 broker 上。

服务器收到消息后会返回一个响应，消息成功写入 Kafka，就返回一个 `RecordMetaData` 对象，它包含了主题和分区信息，以及记录在分区里的偏移量。如果写入失败，则返回一个错误。生产者在收到错误之后会尝试重新发送消息，重试几次仍然失败则返回错误信息。

#### 3.2、创建 Kafka 生产者

创建 Kafka 生产者的三个必要属性：

- bootstrap.servers

  指定 broker 的地址清单，地址格式为 `host:port`。清单里不需要包含所有 broker 地址，生产者可以从给定的 broker 中查找其他 broker 的信息。建议至少提供两个 broker 信息，一旦其中一个宕机，生产者还能连接到集群。

- key.serializer

  key 序列化器，必须被设置为一个实现了 `org.apache.kafka.common.serialization.Serializer` 接口的类。broker 希望接收到的消息的键和值都是字节数组。生产者接口可以使用参数化类型，因此可以把 Java 对象作为键和值发送给 broker，生产者使用 key 序列化器将键对象转化为字节数组。Kafka 客户端提供了 `ByteArraySerializer` 、`StringSerializer`、`IntegerSerializer`，如果使用常见的集中 Java 对象类型，则不用实现自定义的序列化器。

- value.serializer

  value 序列化器。与 key 序列化器类似，将值序列化为字节数组。

一个创建生产者对象的例子：

```java
	private Properties kafkaProps = new Properties();
	kafkaProps.put("bootstrap.servers", "broker1:9092,broker2:9092");
	kafkaProps.put("key.serializer", 
                   "org.apache.kafka.common.serialization.StringSerializer");
	kafkaProps.put("value.serializer", 
                   "org.apache.kafka.common.serialization.StringSerializer");

	producer = new KafkaProducer<String, String>(kafkaProps);
```

此处仅设置了 Kafka 生产者的必要属性，还有一系列的其他属性用来控制生产者的行为。

生产者发送消息的三种方式：

1. ##### 发送并忘记（fire-and-forget）

   只管将消息发送给服务器，不关心它是否正常到达。大多数情况下，消息会正常达到服务器，因为 Kafka 是高可用的，并且生产者会自动尝试重发。但，这种方式有时候会丢失一些消息。

2. ##### 同步发送

   使用 `send()` 方法发送消息，返回一个 `Future` 对象，调用 `get()` 方法进行等待可以知道消息是否发送成功。

3. ##### 异步发送

   调用 `send(rec, Callback)` 方法，并指定一个回调函数，服务器在返回响应时，调用该函数。

本章的例子都是使用单线程，实际上生产者是可以使用多线程来发送消息的。需要高吞吐量的情况下，可以增加生产者数量，也可以生产者数量不变的情况下增加线程数量。

#### 3.3、发送消息到 Kafka

最简单的消息发送：

```java
	ProducerRecord<String, String> record = 
        new ProducerRecord<>("CustomerCountry", "Precision Products", "France");
	try {
		producer.send(record);
	} catch (Exception e) {
        e.printStackTrace();
    }
```

 `producer.send(record)` 返回一个包含 `RecordMetadata` 的 `Future` 对象，但是本例中忽略了返回值——不关心发送是否成功。即使用了发送并忘记的发送方式。

##### 3.3.1、同步发送消息

```
	ProducerRecord<String, String> record = 
        new ProducerRecord<>("CustomerCountry", "Precision Products", "France");
	try {
		producer.send(record).get();
	} catch (Exception e) {
        e.printStackTrace();
    }
```

本例中通过调用 `producer.send(record)` 返回的 `Future` 对象的 `get()` 方法等待 Kafka 响应。如果服务器返回错误，`get()` 方法会跑出异常。如果发送成功，将会得到一个 `RecordMetadata` 对象，通过它可以获取消息的偏移量。

`KafkaProducer` 一般会发生两类错误，一类是**可重试错误**，可以通过重发消息来解决，比如连接错误，可以通过重建连接来解决，“无主（no leader）”错误可以通过重新为分区选举 leader 来解决。`KafkaProducer` 可以被配置为自动重试，如果多次重试后仍无法解决问题，应用程序会收到一个重试异常。另一类错误无法通过重试解决，比如“消息太大”异常——`KafkaProducer` 不会进行任何重试，直接抛出异常。

##### 3.3.2、异步发送消息

通过对回调的支持，在异步发送消息的同时可以对异常情况进行处理：

```java
import org.apache.kafka.clients.producer.Callback

private class DemoProducerCallback implements Callback {
	@Override
    public void onCompletion(RecordMetadata recordMetadata, Exception e) {
        if (e != null) {
            e.printStackTrace();
        }
    }
}

ProducerRecord<String, String> record =
    new ProducerRecord<>("CustomerCountry", "Biomedical Materials", "USA");
producer.send(record, new DemoProducerCallback());
```

如果 Kafka 返回一个错误，回调函数的 `onCompletion` 方法会抛出一个非空（non null）异常。本例只是简单地打印了异常，生产环境应该对异常进行处理。

#### 3.4、生产者配置

除了之前介绍的必要的生产者配置参数，还有其它控制生产者的参数：

1. acks

   指定当有多少个分区副本收到消息后，生产者才会认为消息写入是成功的。这个参数对消息丢失可能性有重要影响，该参数的选项有：

   - acks=0，生产者成功写入消息之前不等待任何来自服务器的响应。生产者无法得知服务器是否收到消息，但是生产者也不需等待服务器的响应，此时她可以以网络能够支持的最大速度发送消息，从而达到很高的吞吐量。
   - acks=1，只要集群的 leader 节点收到消息，生产者就会受到来自服务器的成功响应。如果消息无法达到 leader 节点（leader 崩溃，新的 leader 还没选出），生产者会收到一个错误响应，为避免数据丢失，生产者会重发消息。但是，如果没有收到消息的节点成为新的 leader，消息还是会丢失。