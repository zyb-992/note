# Kafka配置

> 路径：/usr/local/kafka/kafka_2.11-2.4.0/config

## 服务端参数配置

```shell
# 
zookeeper.connect

# 
listeners

# 
broker.id

# 
log.dir

# 
log.dirs

# 
message.max.bytes

```



## 生产者必要参数配置

```shell
# 指定生产者客户端连接kafka集群所需的broker清单
# 内容格式:host1:port1,host2:port2 
# 可以设置一个或者多个地址，中间用逗号隔开
bootstrap.servers

# 指定分区中必须要有多少个副本收到这条消息，生产者才认为这条消息被成功写入
# acks是字符串类型
# 1 默认值即为1 发送消息后 只要分区的leader副本成功写入消息 那生产者就会收到服务端响应
# 0 发送消息后不等待任何服务端响应，若写入Kafka过程中出现异常，没有收到该消息，那么消息就会丢失
# -1 / all 消息发送后需要等待ISR中所有副本成功写入消息后才能够收到服务端的成功响应
acks

# 用来限制生产者客户端能发送的消息的最大值
# 默认值为1048576B（1B=1024bytes）
# 配置该参数要和Broker端的message.max.bytes联系起来
max.request.size

# 配置生产者重试次数 即消息写入发生异常时 重新发送消息
# 当重试次数达到设定的该次数 那么生产者就会放弃重试并且抛出异常
retries

# 设定两次重试之间的时间间隔 避免无效的频繁重试
retry.backoff.ms

# 用来指定消息的压缩方式
# 默认情况 消息不会被压缩
# 可以配置为 gzip, snappy,lz4
# 对消息压缩可以极大减少网络传输、降低网络IO
compression.type

# 指定多久后关闭限制的连接，默认值9分钟
connections.max.idle.ms

# 指定生产者发送ProducerBatch之前等待更多消息(ProduceRecord)加入ProducerBatch的时间
# ProducerBatch在被填满或者linger.ms超过时被发送出去
# 增大该参数会增加消息延迟 但同时能提升一定吞吐量
linger.ms

# 设置Socket接收消息缓冲区(SO_RECBUF)的大小
# 默认为32768B
receive.buffer.bytes

# 设置Socket发送消息缓冲区(SO_SNDBUF)的大小，默认值为131072B
send.buffer.bytes

# 配置Producer等待请求响应的最长时间，默认值为30000ms
request.timeout.ms

# 

```



### 消费者必要参数配置

```shell
# 指定连接kafka集群所需的broker地址清单
bootstrap.servers

# 消费者隶属的消费组的名称 默认为""
group.id
```

