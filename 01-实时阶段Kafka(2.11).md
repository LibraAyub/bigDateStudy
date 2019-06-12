# 01-实时阶段Kafka

## 1. kafka是什么，为什么快

```
1）	Apache Kafka 是一个消息队列（生产者消费者模式）
2）	Apache Kafka 目标：构建企业中统一的、高通量、低延时的消息平台。
3）	大多的是消息队列（消息中间件）都是基于JMS标准实现的，Apache Kafka 类似于JMS的实现。

为什么快？
Broker:Kafka 的设计是把所有的 Message 都要写入速度低容量大的硬盘，以此来换取更强的存储能力。
	首先，Kafka 重度依赖底层操作系统提供的PageCache 功能。当上层有写操作时，操作系统只是将数据写入 PageCache，同时标记 Page 属性为 Dirty。
	当读操作发生时，先从 PageCache 中查找。使用 PageCache 功能同时可以避免在 JVM 内部缓存数据，JVM 为我们提供了强大的 GC 能力，同时也引入了一些问题不适用与 Kafka 的设计。PageCache 还只是第一步，Kafka 为了进一步的优化性能还采用了 Sendfile 技术。N i/o
```

###  1.1 kafka是什么

```
1）	作为缓冲，来异构、解耦系统。削峰，并行。
	用户注册需要完成多个步骤，每个步骤执行都需要很长时间。代表用户等待时间是所有步骤的累计时间。
	为了减少用户等待的时间，使用并行执行执行，有多少个步骤，就开启多少个线程来执行。代表用户等待时间是所有步骤中耗时最长的那个步骤时间。
	有了新得问题：开启多线程执行每个步骤，如果以一个步骤执行异常，或者严重超时，用户等待的时间就不可控了。
	通过消息队列来保证。
		注册时，立即返回成功。
		发送注册成功的消息到消息平台。
		对注册信息感兴趣的程序，可以消息消息。
```

### 1.2 基本架构

```
由多个服务器组成，每个服务器都有一个单独的名字broker;生产者，负责生产数据。消费者负责消费数据。存储数据时将一类数据存放某个topic下。
注意：Kafka的元数据都是存放在zookeeper中。
```

### 1.3  安装、启动

```
需要依赖JDK,Zookeeper  
	修改配置文件server.properties
	1）	Borker.id=0（1）（2）
	2）	数据存放的目录，注意目录如果不存在，需要新建下。
	3）	zookeeper的地址信息

启动：需要指定配置文件（server.properties）；到bin路径下；
	./kafka-server-start.sh /usr/local/src/kafka/kafka-2.11/config/server.properties
或者使用后台启动：
	nohup /usr/local/src/kafka/kafka-2.11/bin/kafka-server-start.sh	/usr/local/src/kafka/kafka-2.11/config/server.properties >/dev/null 2>&1 &
	
linux命令：
	/dev/null  黑洞    
	grep -v "#"  排除注释；-v 是排除
	scp ： 的时候路径指定$PWD 就可以拷贝到其他的相同路径。
	nohup & :后台启动
```

### 1.4 操作topic

```
查看：bin/kafka-topics.sh --list --zookeeper zk01:2181

创建：
bin/kafka-topics.sh --create --zookeeper mini1:2181,mini2:2181,mini3:2181 --replication-factor 3 --partitions 2 --topic WordCount

启动生产者：
	bin/kafka-console-producer.sh --broker-list mini1:9092,mini2:2181,mini3:2181 --topic WordCount
	
启动消费者：
	bin/kafka-console-consumer.sh --zookeeper mini1:2181 --from-beginning --topic order
	
删除 topic（需要设置server.properties 中设置delete.topic.enable=true）
	kafka-topics.sh --delete --zookeeper mini1:2181 --topic system_log
```

## 2 Kafka原理

###  2.1Kafka分片与副本

```
	当数据量非常大的时候，一个服务器存放不了，就将数据分成两个或者多个部分，存放在多台服务器上。每个服务器上的数据，叫做一个分片。
	当数据只保存一份的时候，有丢失的风险。为了更好的容错和容灾，将数据拷贝几份，保存到不同的机器上。一般都是3个。只有一个leader,同时只有leader做读写操作。
bin/kafka-topics.sh --create --zookeeper mini1:2181 --replication-factor 1 --partitions 1 --topic order   这是创建topic
	--replication-factor 1  :代表是副本数为1；--partitions 1 ：分片为1.
```

###  2.2 消息不丢失机制

#### 	2.2.1生产者消息不丢失

```
（1）生产分为同步模式和异步模式；同步：一个消息发送出去后，一直等待结果回应；生产者重试3次，如果还没有响应，就报错。异步：生产消息后，存入一个缓冲池，在producer端，达到一定的数量阈值（2W）或者时间阈值（500ms）之后发送.
(2)消息确认分为三个状态：0,1，-1
	1） 0：生产者只负责发送数据；
	2） 1：某个partition的leader收到数据给出响应；1对1
	3）-1：partiton的所有副本都收到数据后给出响应。
在同步模式下
a)	生产者等待10S，如果broker没有给出ack响应，就认为失败。
b)	生产者重试3次，如果还没有响应，就报错。
在异步模式下
a)	先将数据保存在生产者端的buffer中。Buffer大小是2万条。
b)	满足数据阈值或者数量阈值其中的一个条件就可以发送数据。
c)	发送一批数据的大小是500条。

如果broker迟迟不给ack，而buffer又满了。
开发者可以设置是否直接清空buffer中的数据。
```

#### 	2.2.2 Borker消息不丢失

```
broker端的消息不丢失，其实就是用partition副本机制来保证；
Producer ack -1 能够保证所有的副本都同步好了数据。其中一台机器挂了，并不影响数据的完整性。
```

#### 	2.2.3 消费者端消息不丢失

```
	只要记录offset值，消费者端不会存在消息不丢失的可能。只会重复消费。需要借助外部的力量。partition的offset值可以存放在 zookeeper保证数据一致，就不会丢失。在partition中有segment段 里面有log和index文件，log文件存放的是消息本身。index文件存放的是消息offset值。
	解决kafka重复消费的问题只有是关闭自动提交，改手动提交。
```

### 2.3 文件存储机制

```
	同一个topic下有多个不同partition,每个partition为一个目录，partition命名规则为topic名称+有序序号，第一个partition序号从0开始，序号最大为partitions数量-1.partition默认保留7天的数据。
	当一个partition中有10T数据，会切片成多个1G的数据文件中，同时kafka是一个消息中间件，所以，只是临时存储。满了需要删除过期的数据。这样便于操作。
	segment端中有两个核心的文件一个是.log,一个是.index. 当log文件等于1G时，新的会写入到下一个segment中。
	segment段差不多会存储70万条数据。全局第一个segment从0开始。索引文件中元数据指向对应数据文件中 message 的物理偏移地址。
```

###  2.4 文件查询机制

```
00000000.index的文件名代表最开始的文件，起始偏移量offset为0
00367869.index的消息量起始偏移量为367869+1
以起始偏移量命名并排序这些文件，只要根据offset “二分法” 就可以快速定位到具体文件。当offset为368776时，定位到文件为368769.index和对呀的log文件。这样能快速查找。
```

### 2.5 生产者数据分发策略

```
默认情况使用DefaultPartitioner.class类
1） 用户指定了partition,生产就不会调用DefaultPartitioner.class类
2）用户指定key,使用hash算法。如果key一直不变，同一个key算出来的hash值是个固定值。这样无意义。
Utils.toPositive(Utils.murmur2(keyBytes)) % numPartitions
3）当用户即没有指定partion也没有key.就使用轮询发送数据。
public ProducerRecord(String topic, V value) {
    this(topic, null, null, null, value, null);
}

Producer 消息发送的应答机制
设置发送数据是否需要服务端的反馈,有三个值 0,1,-1
0: producer 不会等待 broker 发送 ack
1: 当 leader 接收到消息之后发送 ack
-1: 当所有的 follower 都同步消息成功后发送 ack
request.required.acks=0
```

### 2.6 消费者的负载均衡

```
问题;生产者的速度很快，但是消费者跟不上。造成数据大量滞后和延时。
解决：多几个消费，共同来消费数据。

问题：消费组中消费者的数量和partition数量一致，但是消费者的处理速度还是跟不上。
解决：要么修改topic的partition数量，要么减少消费者处理时间，提高处理速度。

一个partition只能被一个组中的成员消费。
所以如果消费组中有多于partition数量的消费者，那么一定会有消费者无法消费数据。
1）多出来的消费者是处于空闲状态。在真实解决方案：要么修改topic的partition数量要么减少消费者处理时间，提高处理速度。
```
## 3.一键启动

```
1)写一个 slave文件 内容：
	mini1
	mini2
	mini3
	
2)在写一个.sh文件startkafka.sh文件，读取slave每行内容，然后通过ssh远程调用profile文件
内容：
	cat /usr/local/src/kafka/onekey/slave | while read line
	do
	{
      echo $line
      ssh $line "source /etc/profile;nohup /usr/local/src/kafka/kafka-2.11/bin/kafka-server-start.sh	/usr/local/src/kafka/kafka-2.11/config/server.properties >/dev/null 2>&1 &"
	}$
	wait
	done
	
```