# 05-离线阶段HDFS,MapReduce

## 1.Namenode基本原理

```
1.namenome是HDHS的核心; 2.也称master; 3.仅存储HDFS的元数据; 4.不存储实际的数据; 5.namenode知道HDFS的任何给定文件的块列表及其位置; 6.并不持久化存储各个块所在的datanode位置信息,保存在内存中; 7.namenode对于HDFS关闭时.集群无法访问; 8.namenode是集群中的单点故障; 9.Namenode所在机器通常配置大量内存.
```

## 2.DataNode概述

```
1.负责将实际数据存储在HDFS中; 2.也称Slave; 3.Namenode和datanode不断通信; 4.启动时,他将自己发布到namenode并汇报自己负责持有的块列表; 5.关闭时,不会影响集群可运行,安排副本复制; 6.需要大量磁盘空间; 7.定期向namenode默认3秒发送心跳; 8.block汇报时间hfs,blockreport.in tervalMsec 未配置默认6小时.
```

## 3.HDFS工作机制

```
内部工作机制对客户端保存透明,客户端请求访问HDFS都是向NameNode.然后NameNode指挥DataNode进行操作.
```

### 3.1写的流程

```
	一个Namenode,多个datanode.nn:需要响应所有客户端的请求;维护文件系统的目录树;管理DNS.
	1.客户端(RPC)请求上传文件.先切分(每个128MB); 2.请求NameNode; 3.NameNode检测文件系统的目录树(只能上传没有的文件); 4.请求Namenode上传第一块(block); 5.检测dataNode的信息池; 6.返回可用的datanode(通过网络拓扑图选择); 7.根据返回的参数建立连接,请求数据传输,建立pipeline(数据传输的管道); 8.反向pipeline告知建立完毕; 9.建立数据传输流(以64k为单位的包传输); 10.保存传输过来的源源不断的数据包; 11.反向数据校验,数据保存成功.(第二块也是相同的思路也可以参考上传图)
```

### 3.2 读的流程

```
	1.客户端请求获取文件; 2.nn返回跟请求相关的元数据信息. 3.1请求下载文件1;请求下载文件2;请求下载文件
4.客户端把该文件的所有下载过了的下载过了进行合并.
```
## 4.MapReduce思想

```
分而治之,map负责分;reduce负责合.
```

### 4.1设计构思

```
统计一个文档中,每个单词的出现次数
	
```

### 4.2 编程规范

```
Mapper;Reduce;Driver
reduce的输入数据类对呀mapper的输出数据类型.也是kv
用户自定义的 Mapper 和 Reducer 都要继承各自的父类
整个程序需要一个 Drvier 来进行提交，提交的是一个描述了各种必要信息的 job 对象
```