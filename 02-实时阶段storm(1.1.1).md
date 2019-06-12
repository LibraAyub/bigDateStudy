# 02-实时阶段storm

## 1. storm是什么

```
1）	是一个流式计算框架，数据源源不断的产生，收集。计算。相当于一条一条计算，启动将永不停止
2）	只负责数据的计算，不负责存储。
3）	也会有主从结构。能晶块得到出后的数据。
```

###  1.1 storm架构

```
Nimbus:负责资源分配和任务调度
Supervisor：负责接受 nimbus 分配的任务，启动和停止属于自己管理的 worker 进程。 
Worker：运行具体处理组件逻辑的进程。
Task：worker 中每一个 spout/bolt 的线程称为一个 task. 在 storm0.8 之后，task 不再与物理线程对应，同一个 spout/bolt 的 task 可能会共享一个物理线程，该线程称为 executor。
```

### 1.2 基本架构

```
由多个服务器组成，每个服务器都有一个单独的名字broker;生产者，负责生产数据。消费者负责消费数据。存储数据时将一类数据存放某个topic下。
注意：Kafka的元数据都是存放在zookeeper中。
```

### 1.3  编程模型

```
Topology：Storm 中运行的一个实时应用程序，因为各个组件间的消息流动形成逻辑上的一个拓扑结构。
Spout：在一个 topology 中产生源数据流的组件。通常情况下 spout 会从外部数据源中读取数据，然后转换为 topology 内部的源数据。是一个主动角色。
Bolt：在一个 topology 中接受数据然后执行处理的组件。Bolt 可以执行过滤、函数操作、合并、写数据库等任何操作。Bolt 是一个被动的角色。
Tuple：一次消息传递的基本单元。本来应该是一个 key-value 的 map。
Stream：源源不断传递的 tuple 就组成了 stream。
```

###  1.4 storm的安装

```
nimbus.host:的后面必须加一个空格
nimbus.childopts: 的后面必须加一个空格
```

```
#指定storm使用的zk集群
storm.zookeeper.servers:
     - "mini1"
     - "mini2"
     - "mini3"
#指定storm集群中的nimbus节点所在的服务器
nimbus.host: "mini1"
#指定nimbus启动JVM最大可用内存大小
nimbus.childopts: "-Xmx1024m"
#指定supervisor启动JVM最大可用内存大小
supervisor.childopts: "-Xmx1024m"
#指定supervisor节点上，每个worker启动JVM最大可用内存大小
worker.childopts: "-Xmx768m"
#指定ui启动JVM最大可用内存大小，ui服务一般与nimbus同在一个节点上。
ui.childopts: "-Xmx768m"
#指定supervisor节点上，启动worker时对应的端口号，每个端口对应槽，每个槽位对应一个worker
supervisor.slots.ports:
    - 6700
    - 6701
    - 6702
    - 6703
```

#### 	1.4.1 启动storm

```
到主hosts的storm 文件目录下bin/storm nimbus 
其他节点(2,3)输入    bin/storm supervisor
启动后在主节点输入	bin/storm ui  这样就可以从外部登陆mini1:8080 默认端口号是8080 最好更改为8088
	假如配置的PATH路径就可以不需要必须在安装目录下启动了。
```

### 1.5 ssh 远程拷贝（$PWD）

```
scp -r 1.txt mini2:$PWD
```
## 2  WordCount案例

###  2.1  启动类WordCountTopology

```
1.创建一个job;
	new TopologyBuilder();
2.设置job的详细内容；可以指定执行的步骤。
	.setSpout("ReadFileSpout",new ReadFileSpout(),2);
	.setBolt("SentenceSplitBolt",new SentenceSplitBolt(),4).shuffleGrouping("ReadFileSpout");
	.setBolt("WordCountBolt",new WordCountBolt(),4).shuffleGrouping("SentenceSplitBolt");
3.提交job(本地和集群)
	new LocalCluster();
	.submitTopology("wordcount", config, topologyBuilder.createTopology());
```

### 2.2 Spout（ReadFileSpout）

```
需要继承一个模板：BaseRichSpout
open（）方法是初始化；
nextTuple()方法一直有一个while循环，在调用nextTuple() 一行调用一次。
	collector.emit(Arrays.asList(line));输出
declareOutputFields（）方法声明发出的数据是什么。
	declarer.declare(new Fields("juzi"));
```

### 2.3 Bolt （SplitBolt）

```
需要继承一个模板： BaseRichBolt
这个类主要是切割单词。
prepare（）方法是初始化的方法，可以用来获取数据收集数据。
	this.collector = collector;
execute（）方法，每次调用都会被while方法调用。所以也就是在这里进行单词的切割处理。
	String[] strings = sentence.split(" ");
        for (String word : strings) {
            // todo 输出数据
            collector.emit(new Values(word,1));
        }
declareOutputFields（）用于传递到下一个阶段。
	declarer.declare(new Fields("word","num"));
```

### 2.3 计算阶段（WordCountBolt）

```
需要继承一个模板： BaseRichBolt
这个类主要是用来计算单词的个数计算。
prepare（）方法
	this.collector = collector;
execute（）方法进行个数的计算。
	根据上一个bolt的传递下来的方法（"word","num"） 。
	进行判断字段后计算。
declareOutputFields（） 由于是最后一个计算步骤，所以不需要传递下去。
```
### 2.4 组件设置

```
所有组件（spout/bolt）的数量设置，都是根据单位处理的数量来设置。worker数量的设置，根据所有组件的task数量来设置，如果一个worker只能运行16个task。但是完成一定数额数据流，需要32个task,就只能分出多个worker.
Num workers 是集群分配的基本资源。进程
Num executors 代表的是任务。一般executors的数量和tasks的数量是相等的。
replication:代表副本数。代表nimbus启动的个数

```
## 3 实时交易数据设计

```
	支付系统+kafka+storm/Jstorm集群+redis集群；
	1、支付系统发送mq到kafka集群中，编写storm程序消费kafka的数据并计算实时的订单数量、订单数量；
	2、将计算的实时结果保存在redis中；
	3、外部程序访问redis的数据实时展示结果。
	在实际的需求中，数据有可能有上百条的数据。甚至计算统计。人数，订单数，销售额，支付总额，类，线，店。等。
```

### 3.1 kafka整合storm

```
		<dependency>
            <groupId>org.apache.storm</groupId>
            <artifactId>storm-kafka-client</artifactId>
            <version>1.1.1</version>
        </dependency>
        <dependency>
            <groupId>org.apache.kafka</groupId>
            <artifactId>kafka-clients</artifactId>
            <version>0.10.0.0</version>
        </dependency>
```