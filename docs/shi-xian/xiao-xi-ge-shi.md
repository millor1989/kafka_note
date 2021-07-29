### 消息格式

消息（即，记录）总是成批写入的。一个消息的批次的技术术语是一个记录批次，一个记录批次包含一个或者多个记录。在简并情况下（degenerate case），会有一个记录批次只有一条记录。记录批次和记录有自己的headers。

#### 1、记录批次

RecordBatch的硬盘上格式：

```java
		baseOffset: int64
		batchLength: int32
		partitionLeaderEpoch: int32
		magic: int8 (current magic value is 2)
		crc: int32
		attributes: int16
			bit 0~2:
				0: no compression
				1: gzip
				2: snappy
				3: lz4
				4: zstd
			bit 3: timestampType
			bit 4: isTransactional (0 means not transactional)
			bit 5: isControlBatch (0 means not a control batch)
			bit 6~15: unused
		lastOffsetDelta: int32
		firstTimestamp: int64
		maxTimestamp: int64
		producerId: int64
		producerEpoch: int16
		baseSequence: int32
		records: [Record]
```

注意，当启用了压缩时，压缩的记录数据是紧接着记录数的计数进行序列化的。

CRC覆盖了从属性到批次结尾的数据（即，CRC后的所有字节）。它位于magic字节后，这意味着客户端必须在决定如何解释批次长度和magic字节之间的字节前解析magic字节。partitionLeaderEpoch不包括在CRC运算中，以避免当这个属性被分配给代理接收的每个批次时需要重新运算这个CRC。CRC-32C多项式被用于这个运算。

压缩时：不同于老的消息格式，当日志被清理掉，magic v2和以上的版本会保留原始批次中的第一个和最后一个偏移/序号。这是需要的，为了能够在日志重新加载的时候可以恢复生产者的状态。比如，如果不保留最后的序号，那么在一个分区leader故障后，生产者可能会遇到OutOfSequence错误。基本的序号必须保留以用于重复检查（代理通过验证从相同生产者进来的Produce请求的第一个和最后一个序号，来检查进来的Produce请求是否重复）。结果是，当批次中所有的记录被清空当时批次仍然保留着以保留生产者的最后一个序号时，日志中可能有空的批次。一个古怪的地方是，压缩时firstTimestamp不会保留，所以如果批次中的第一条记录被压缩掉后它会改变。

##### 1.1、控制批次

控制批次只包含一条叫作控制记录的记录。控制记录不应该被传递到应用。而是被消费者用来过滤掉事务消息。

控制记录的key服从如下schema：

```java
       version: int16 (current version is 0)
       type: int16 (0 indicates an abort marker, 1 indicates a commit)
```

控制记录的值的schema依赖于类型。值对客户端是不透明的。

#### 2、记录

记录级别的header是Kafka 0.11.0引入的。具有header的记录的硬盘上格式如下：

```java
		length: varint
		attributes: int8
			bit 0~7: unused
		timestampDelta: varint
		offsetDelta: varint
		keyLength: varint
		key: byte[]
		valueLen: varint
		value: byte[]
		Headers => [Header]
```

##### 2.1、记录header

```java
		headerKeyLength: varint
		headerKey: String
		headerValueLength: varint
		Value: byte[]
```

使用与Protobuf相同的varint编码。记录中headers的数量也被编码为一个varint。

#### 3、旧的消息格式

Kafka 0.11之前，消息是在**消息集合（message set）**中进行传输和保存的。在消息集合中每条消息都有自己的元数据（metadata）。注意，尽管消息集合是由数组表示的，但是它们不像协议中的其它数组元素，前面有一个int32的数组大小。

**消息集合：**

```java
    MessageSet (Version: 0) => [offset message_size message]
        offset => INT64
        message_size => INT32
        message => crc magic_byte attributes key value
            crc => INT32
            magic_byte => INT8
            attributes => INT8
                bit 0~2:
                    0: no compression
                    1: gzip
                    2: snappy
                bit 3~7: unused
            key => BYTES
            value => BYTES
```

```java
    MessageSet (Version: 1) => [offset message_size message]
        offset => INT64
        message_size => INT32
        message => crc magic_byte attributes timestamp key value
            crc => INT32
            magic_byte => INT8
            attributes => INT8
                bit 0~2:
                    0: no compression
                    1: gzip
                    2: snappy
                    3: lz4
                bit 3: timestampType
                    0: create time
                    1: log append time
                bit 4~7: unused
            timestamp => INT64
            key => BYTES
            value => BYTES
```

在Kafka 0.10之前的版本中，只支持消息格式版本0。在版本0.10，引入了支持时间戳的消息格式版本1。

- 与版本2以上类似，`attributes`的最低的比特表示压缩类型
- 版本1中，生产者应该总是设置`timestampType`比特为0。如果主题被配置为使用log append time（通过代理级别配置`log.message.timestamp.type = LogAppendTime`或者主题级别配置`message.timestamp.type = LogAppendTime`）代理会覆盖这个时间戳类型消息集合中的时间戳。
- `attributes`最高的比特位必须设置为0

在消息格式版本0和1中，Kafka支持递归消息以开启压缩。这种情况下，消息的`attributes`必须设置为某一种压缩类型，并且value字段必须包含一条用那个类型进行压缩的消息集合。通常把嵌套的消息称为“内部消息”，把包括消息称为“外部消息”。注意，外部消息的key应该是null，并且它的偏移是最后一条内部消息的偏移。

当接收递归的版本0消息时，代理将它们解压并且每个消息会被分配一个偏移量。在版本1，为了避免服务器侧的重复压缩，只有包裹消息会被分配偏移量。内部消息会有相对偏移量。可以通过外部消息的偏移（对应于分配给最后一条内部消息的偏移）来计算内部消息的绝对偏移。

crc字段包含了后续消息字节（即，从magic byte到value的字节）的CRC32（不是CRC-32C）。