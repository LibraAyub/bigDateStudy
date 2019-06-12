# 04-实时日志监控告警

## 1.技术

```
Flume+Kafka+Storm+Redis+Mysql+java web
```

## 2. 开发步骤

```
1）创建数据库的表架构、初始化业务系统名称、业务系统需要监控的字段、业务系统的开发人员（手机号、邮箱地址）
2）编写Flume脚本去收集数据
3）创建Kafka的topic，触发规则，规则在mysql里去提取。
4）编写Storm程序。判断，保存到数据库
```

## 3.数据库表结构

```
用户表；应用程序表；规则表；结果表。详细略
```

## 4. Flume+Kafka整合

###  4.1 整合前思考

```
1)文件由谁生成：log4j生成，生成一个error.log文件。一直监控这个文件，采集完重命名另外一个文件。又生成这个文件。所以会一直监控这个文件 采用：
tail -F /export/data/flume/click_log/error.log
	-F 文件不见会重新生成一个。
2）flume如何采集：log文件；channels使用内存；sink下沉到kafka.
3) kafka topic是什么：system_log
```

### 4.2 创建 修改配置

#### 		4.2.1创建topic

```
需要先启动zookeeper;然后启动Kafka.创建log_monitor  的topic用于存储error信息。
	kafka-topics.sh --create --zookeeper mini1:2181 --topic log_monitor --partitions 6 --replication-factor 2
```

####  		4.2.2配置Flume配置文件（下沉到kafka）

```
a1.sources = r1
a1.channels = c1
a1.sinks = k1

a1.sources.r1.type = exec
a1.sources.r1.command = tail -F /export/data/flume/click_log/error.log
a1.sources.r1.channels = c1

a1.channels.c1.type=memory
a1.channels.c1.capacity=10000
a1.channels.c1.transactionCapacity=100

a1.sinks.k1.type = org.apache.flume.sink.kafka.KafkaSink
a1.sinks.k1.topic = system_log
a1.sinks.k1.brokerList = mini1:9092
a1.sinks.k1.requiredAcks = 1
a1.sinks.k1.batchSize = 20
a1.sinks.k1.channel = c1
```
#### 	4.2.3 后台启动flume

```
./start.sh 一定要写在一行 内容：
source /etc/profile;nohup flume-ng agent -n a1 -c /usr/local/src/flume/apacheflume/myconfig/app_interceptor conf -f /usr/local/src/flume/apacheflume/myconfig/app_interceptor/app_interceptor.conf >/dev/null 2>&1 &

-n:代表source的名字；
-c:指定配置文件的路径；
-f:指定配置文件。
```

#### 	4.2.4 启动kafka Consumer

```
kafka-console-consumer.sh --zookeeper mini1:2181 --from-beginning –topic log_monitor
```

### 4.3 Flume 拦截器的使用

```
根据不同的消息发出不同的告警。所以需要我们在配置文件中进行修改，首先，上传一个jar包。到lib目录，启动集群，kafka,运行jar包，创建topic, 监控文件。
```

#### 	4.3.1修改配置文件（添加拦截器）

```
a1.sources = r1
a1.channels = c1
a1.sinks = k1

a1.sources.r1.type = exec
a1.sources.r1.command = tail -F /export/data/flume/click_log/error.log
a1.sources.r1.channels = c1
a1.sources.r1.interceptors = i1
a1.sources.r1.interceptors.i1.type = cn.itcast.realtime.flume.AppInterceptor$AppInterceptorBuilder
a1.sources.r1.interceptors.i1.appId = 1

a1.channels.c1.type=memory
a1.channels.c1.capacity=10000
a1.channels.c1.transactionCapacity=100

a1.sinks.k1.type = org.apache.flume.sink.kafka.KafkaSink
a1.sinks.k1.topic = log_monitor
a1.sinks.k1.brokerList = kafka01:9092
a1.sinks.k1.requiredAcks = 1
a1.sinks.k1.batchSize = 20
a1.sinks.k1.channel = c1
```

#### 	4.3.2 启动kafka和消费者

```
启动：
source /etc/profile;nohup flume-ng agent -n a1 -c /usr/local/src/flume/apache-flume/myconfig/app_interceptor conf -f /usr/local/src/flume/apache-flume/myconfig/app_interceptor/app_interceptor.conf >/dev/null 2>&1 &

消费者：
kafka-console-consumer.sh --zookeeper zk01:2181 --from-beginning --topic log_monitor
```

### 4.4 Storm 程序

```
导入kafka整合storm的依赖
<dependency>
     <groupId>org.apache.storm</groupId>
     <artifactId>storm-core</artifactId>
     <version>1.1.1</version>
</dependency>
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

### 4.5 业务逻辑

```
1）KafkaSpout 负责读取数据。并行度设置6个，在一个消费组中启动6个消费者（Task）
2）ProcessBolt 负责检验数据中是否包含关键词（这个继承BaseBasicBolt）
	读取上游发送的数据，上游的数据会有五个字段。Value字段就是我们需要的。
	aid:1||msg:error java.lang.ArrayIndexOutOfBoundsException
	对数据进行分割，得到应用的编号和错误的信息。
	通过应用编号获得应用所有的规则信息。
	迭代所有的规则信息，如果有一个规则被触发了，直接返回这个规则的编号。
	返回规则编号，如果不是0000，就向下游发送数据。
3）notifyBolt 发送短信和发送邮件等信息
	先判断这个应用的这个规则，是否是在五分钟内已经发送过短信。
	如果没有发过，发送短信（聚合数据API,3分钱一条）、发送邮件（自己编写）
	拼装以下触发规则的信息，准备保存到数据库。
4）save2db 将触发规则的信息保存到数据库.
```
### 4.6 分布式锁（五分钟只加载一次）

```
 private static boolean tag = true;
    /**
     * 5分钟加载一次，并且只加载一次
     */
    public static void reload() {
        //五分钟加载一次
        Date date = new Date();
        //获取分钟值
        SimpleDateFormat simpleDateFormat = new SimpleDateFormat("mm");
        String value = simpleDateFormat.format(date);
        //%5=0,1,2,3,4
        if (Integer.parseInt(value) % 5 == 0) {
            doReloadData();
        } else {
            tag = true;
        }

    }
    //锁
    private static void doReloadData() {
        synchronized (LogMonitorHandler.class) {
            if (tag) {
                System.out.println("-------------我要重新加载数据-----------------");
                loadData();
                System.out.println("---------------加载数据完毕-------------------");
                tag = false;
            }
        }
    }
```