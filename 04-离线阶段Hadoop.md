# 04-离线阶段_Hadoop

## 1.Hadoop

```
	Hadoop 是 Apache 旗下的一个用 java 语言实现开源软件框架，是一个开发和运行处理大规模数据的软件平台。允许使用简单的编程模型在大量计算机集群上对大型数据集进行分布式处理。
```

```
HDFS（分布式文件系统）：解决海量数据存储;角色有:NameNode,DataNode,SecondaryNameNode(秘书)
YARN（作业调度和集群资源管理的框架）：解决资源任务调度;角色有:ResourceManager、NodeManager
MAPREDUCE（分布式运算编程框架）：解决海量数据计算
```

### 1.1Hadoop特性

```
1.扩容能力;2.成本低;3.高效率;4.可靠性.
```

### 1.2 国内应用

```
个人征信;投资模型;车辆,路况;用户上网行为分析;互联网;
```

## 2集群

```
HDFS集群和YARN集群:两者逻辑上分离,但物理上常在一起.

以三个节点为例:角色的分配.
	node-01 NameNode DataNode ResourceManager
	node-02 DataNode NodeManager SecondaryNameNode
	node-03 DataNode NodeManager

切记同步时间;安装JDK;防火墙
date -s "2017-03-03 03:03:03"
```
## 3.启动

```
	首次启动HDFS时,必须对其进行格式化操作.
	其实就是初始化清理和准备工作.
	格式化之后,集群启动成功,就不需要格式化.其实就是更改version的ID.
	格式化的操作在HDFS集群的主角色(NameNode).
	如果初始化多次后就需要删除data下的所有节点的ID文件.然后再重新格式化生成文件.
	start-dfs.sh(需要配置path路径)
	start-yarn.sh(需要配置path路径)
	
	也可以使用start-all.sh启动；停止：stop-all.sh
```

### 3.1 初识hdfs

```
hdfs dfs -ls/	查看
hdfs dfs -mkdir /hello 创建文件夹
hdfs dfs -put 文件名称 /hello 上传文件
```

### 3.2初识mapreduce

```
在 Hadoop 安装包的 hadoop-2.7.4/share/hadoop/mapreduce 下有官方自带的 mapreduce 程序。我们可以使用如下的命令进行运行测试。
示例程序 jar:
	hadoop-mapreduce-examples-2.7.4.jar
计算圆周率:
	hadoop jar hadoop-mapreduce-examples-2.7.4.jar pi 20 50
	
NameNode http://nn_host:port/ 默认 50070.
ResourceManager http://rm_host:port/ 默认 8088.
```

## 4 HDFS入门

```
传统的存储模式:
	1.上传下载耗时;
	2.遇到存储瓶颈.
		纵向:加磁盘,加内存,有上限.
		横向:加机器.多台

设计目标:不支持写 值最加
```

### 4.1 重要特性

```
首先是一个文件系统;他是分布式.
1.master/slave架构;
2.分块存储;128MB;
3.NameSpace;
4.Namenode元数据管理.我们把目录结构及文件分块位置信息就叫元数据.
5.Datanode数据存储;datanode需要定时向Namenode汇报自己持有的block信息.
6.副本机制;为了容错.包括自己
7.一次写入多次读出. 一次写入,多次读取.且不支持文件的修改.并不适合用来做网盘.
```

## 5 Shell命令行客户端

```
hadoop fs -ls -h hdfs://namenode:host/b  查看数据显示大小为k

hadoop fs -mkdir [-p] <paths>在 hdfs 上创建目录，-p 表示会创建路径中的各级父目录。

hadoop fs -put [-f] [-p] [ -|<localsrc1> .. ]. <dst>
	功能：将单个 src 或多个 srcs 从本地文件系统复制到目标文件系统（HDFS）。
	-p：保留访问和修改时间，所有权和权限。
	-f：覆盖目的地（如果已经存在）
	示例：hadoop fs -put -f localfile1 localfile2 /user/hadoop/hadoopdir

hadoop fs -get [-ignorecrc] [-crc] [-p] [-f] <src> <localdst>
	-ignorecrc：跳过对下载文件的 CRC 检查。
	-crc：为下载的文件写 CRC 校验和。
	功能：将文件复制到本地文件系统。
	示例：hadoop fs -get hdfs://host:port/user/hadoop/file localfile

hadoop fs -appendToFile <localsrc> ... <dst>
	功能：追加一个文件到已经存在的文件末尾
	示例：hadoop fs -appendToFile localfile /hadoop/hadoopfile
	
-getmerge 
功能：合并下载多个文件
示例：比如 hdfs 的目录 /aaa/下有多个文件:log.1, log.2,log.3,...
hadoop fs -getmerge /aaa/log.* ./log.sum

-setrep 
功能：改变一个文件的副本系数。-R 选项用于递归改变目录下所有文件的副本系数。
示例：hadoop fs -setrep -w 3 -R /user/hadoop/dir1
```