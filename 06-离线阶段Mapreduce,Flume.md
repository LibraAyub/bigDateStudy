# 06-离线阶段Mapreduce,Flume

## 1.reducetask

```
reduce个数跟最终输出文件个数,文件被分区几个部分有关系对等.
默认只有一个reducetask  part-r-0000
手动去改变个数.
job.setNumReducetask
如何分区呢?
	这和Hash取值有关.
```
### 1.1 map阶段

```
逐个遍历待处理目录下的文件
	split size=block size==128MB
切片的个数就决定了本次mr程序启动多少个maptask
在读取一行的时候. 获取到了K V ;每解析出来的一个
<k,v>，调用一次 map 方法。得到context, 写入内存.当到一定的内存(内存溢出)会保存到磁盘中.分区排序.键相等的键
值对会调用一次 reduce 方法。
```

### 1.2 ruduce阶段

```
  磁盘分区完后,  reduce拉取属于自己的分区(默认分区规则,key的hash取余)的数据.按照key的字典序进行排序,按照key是否相同作为一组来调用重写的ruduce方法.
  k:就是这一组共同的key V:这一组所有的V组装的一个迭代器<hadoop,[1,1,1]>,然后通过context(上下文)进行输出.
```

### 1.3 compareTo

```
compareTo 方法用于将当前对象与方法的参数进行比较。
	如果指定的数与参数相等返回 0。
	如果指定的数小于参数返回 -1。
	如果指定的数大于参数返回 1。
例如：o1.compareTo(o2);
返回正数的话，当前对象（调用 compareTo 方法的对象 o1）要排在比较对象（compareTo 传参对象 o2）后面，返回负数的话，放在前面。
```

​	在整个 MapReduce 程序的开发过程中，我们最大的工作量是覆盖 map 函数和覆盖 reduce 函数。 

## 2.MapReduce序列化(Writable)

```
指吧结构对象转换为字节流
反序列化:是序列化的逆过程
```

### 2.1 hadoop的序列化机制

```
Writable,比较一个bean对象的时候还需要实现comparable接口,因为mapreduce框架中shuffle过程一定会对key进行排序.
	MR 程序在处理数据的过程中会对数据排序(map 输出的 kv 对传输到 reduce之前，会排序)，排序的依据是 map 输出的 key。所以，我们如果要实现自己需要的排序规则，则可以考虑将排序因素放到 key 中，让 key 实现接口:WritableComparable，然后重写 key 的 compareTo 方法
```
### 2.2  Mapreduce 的分区—Partitioner

```
Mapreduce 中会将 map 输出的 kv 对，按照相同 key 分组，然后分发给不同的 reducetask。
默认的分发规则为：根据 key 的 hashcode%reducetask 数来分发
所以：如果要按照我们自己的需求进行分组，则需要改写数据分发（分组）组件 Partitioner，自定义一个 CustomPartitioner 继承抽象类：Partitioner，然后在job 对象中，设置自定义 partitioner：job.setPartitionerClass(CustomPartitioner.class)
```

### 2.3 Mapreduce 的 combiner

```
	每一个 map 都可能会产生大量的本地输出，Combiner 的作用就是对 map 端的输出先做一次合并，以减少在 map 和 reduce 节点之间的数据传输量，以提高网络 IO 性能，是 MapReduce 的一种优化手段之一
	具体实现步骤：
1、自定义一个 combiner 继承 Reducer，重写 reduce 方法
2、在 job 中设置： job.setCombinerClass(CustomCombiner.class)
```

## 3. Apache Flume(1.6)

```
1.6后可以支持断点续传
```

### 3.1运行机制

```
核心角色是:agent(数据传递员) 是java的一个进程.events是flume内部数据传输的最基本单元.
```

```
	多级agent: 一个agent传递给另一个agent
```

### 3.2 Flume安装

```
	进入flume目录,修改conf下flume-env.sh,在里面配置java_HOME即可.
运行:
	bin/flume-ng agent -c conf -f conf/netcat-logger.conf -n a1 -Dflume.root.logger=INFO,console
-c conf 指定 flume 自身的配置文件所在目录
-f conf/netcat-logger.con 指定我们所描述的采集方案
-n a1 指定我们这个 agent 的名字 一定要和脚本的名字一样
```
### 3.3 spooldir 监控目录

```
采集源，即 source——监控文件目录 : spooldir
 下沉目标，即 sink——HDFS 文件系统 : hdfs sink
 source 和 sink 之间的传递通道——channel，可用 file channel 也可以用内存 channel, 优先选择file channell

注意: 当这个文件中已经有这个文件,这个flume将报错,并停止运行

```

### 3.4 采集文件  监控文件

```
 采集源，即 source——监控文件内容更新 : exec ‘tail -F file’
 下沉目标，即 sink——HDFS 文件系统 : hdfs sink
 Source 和 sink 之间的传递通道——channel，可用 file channel 也可以用内存 channel
# 监控test文件
a1.sources.r1.type = exec
a1.sources.r1.command = tail -F /root/logs/test.log
a1.sources.r1.channels = c1
```

### 3.5 负载均衡 avro

```
负载均衡的实现需要在flume的上一级实现,需要Load balancing Sink Processor能够实现 load balance 功能，我们在启动的时候,我们需要先启动后面的flume,这是为了保证数据的完整性.
a1.sinkgroups = g1
a1.sinkgroups.g1.sinks = k1 k2 k3
a1.sinkgroups.g1.processor.type = load_balance
a1.sinkgroups.g1.processor.backoff = true #如果开启，则将失败的 sink 放入黑名单
a1.sinkgroups.g1.processor.selector = round_robin # 另外还支持 random
a1.sinkgroups.g1.processor.selector.maxTimeOut=10000 #在黑名单放置的超时时间，超时结
束时，若仍然无法接收，则超时时间呈指数增长
```

```
Failover Sink Processor :这是和负载均衡不一样的. 这是采用优先级别sink防止出现错误的情况,当一个机器出错的.后面一个优先级的上.负载均衡是采用轮询的方式下沉.只有一个能工作.出错了后面顶上来.
a1.sinkgroups = g1
a1.sinkgroups.g1.sinks = k1 k2 k3
a1.sinkgroups.g1.processor.type = failover
a1.sinkgroups.g1.processor.priority.k1 = 5 #优先级值, 绝对值越大表示优先级越高
a1.sinkgroups.g1.processor.priority.k2 = 7
a1.sinkgroups.g1.processor.priority.k3 = 6
a1.sinkgroups.g1.processor.maxpenalty = 20000 #失败的 Sink 的最大回退期（millis）
```