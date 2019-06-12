# 03- Storm增强

## 1 源码步骤

```
storm流程：
当我们在linux提交时：storm jar xxx.jar com...Topolo
任务提交到node1节点，获取zookeeper中空闲worker;根据任务需要的信息进行分配在node2上6700上运行；上传任务消息到zookeeper；zookeeper上保存着task信息，spout,bolt,ackbolt;
supervisior :  一直监控着zookeeper的目录，得到回响6700；去下载任务信息启动6700worker;启动worker6700,这是一个独立的进程。
```

```
客户端：
storm  jar  ....jsr com...WordCountTopology客户端1.执行WordCountTopology的main方法。 2.创建一个topologyBuilder,(设置了用户开发的业务逻辑（spout/bolt）)。3.得到一个stormtopology的对象，这个对象是对用户开发的代码进行序列化，用于rpc远程传输。4.提交任务submit()。5.通过rpc框架，将序列化的任务信息上传到nimbus上。
serviceHandler:
submit（）方法 1.检查客户端提交任务是否存在，如果存在就报错。2.检查任务队列中是否有重名的任务。3.以上检查通过后，就会生成一个任务id(wordcount-1-151xxx)。4.初始化集群上所有的配置文件。5.创建本地目录，用来存放stormcode。6.在zookeeper上生成每个任务的监控目录，用来保存任务的执行状态。7.生成任务信息，将任务信息存放到任务队列中。8.serviceHandler中有个TopologyAssign类的静态方法，有个while不停的再消费linkedblocingqueue的任务信息。9.消费到任务信息之后，就会任务进行分配。10.分配任务的时候，要判断是否重新分配任务。新任务直接开始分配。11.分配任务的时候需要考虑是本地模式，还是集群模式。12.本地模式下，会获取本地集群supervisor的信息，在supervisorinfo中就包含了，当前的supervisor能够管理几个worker,有几个worker时可用的。13.找到空闲的worker,并进行分配。14.minbus将任务信息写入到zookeeper。15.supervisor启动时，有一个叫做syncProcessEvent的类。会执行watch的回调。16.在syncProcessEvent类中有个方法方法用来启动worker。17.在本地模式下回调worker.mkworker方法。
```

## 2 消息不丢失机制

###  2.1 ack机制

```
	通过ack机制，spout发出的诶一条消息，都可以确定是被成功处理或者失败处理。从而可以让开发者才取动作。
	采用的算法是：异或算法，相同的消息（数字）都为0.
	当spout触发失败是，不会自动重发失败的tuple。需要spout自己重写获取数据，手动重新发送。
	这个timeout时间可以通过Config.TOPOLOGY_MESSAGE_TIMEOUT_SECS来设定。Timeout的默认时长为30秒
```

### 2.2 开启ack 方法

```
1.spout代码进行修改
	1）重写ack方法和fail方法。
		当spout发送的一条消息，被完整的处理了，storm框架会调用这个ack方法。失败就调用fail方法。
	2）在nexttuple方法中发送数据的时候，需要制定messageid。这个messageid需要保证唯一。 使用UUID
2.bolt代码修改
	1）在execute方法中，处理完所有的业务逻辑之后。需要手动的ack以下  collector.ack(input);如果失败就需要collector.fail(input);
	2）bolt如果产生了新的数据，需要锚点一点。让新的tuple和老的tuple产生关联。
注意： bolt继承BaseBasicBolt 不需要手动的去添加锚点和声明处理成功；
	在继承BaseBasicBolt，想让storm知道我们的数据，没有被完整处理，可以跑出一个异常。failedexcption
```