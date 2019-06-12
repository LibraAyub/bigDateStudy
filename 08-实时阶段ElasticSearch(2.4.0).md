# 08- ElasticSearch

```
基于Lucene的搜索服务器，它提供分布式多用用户能力的全文搜索引擎，基于RestFul web接口。Elasticsearch（构建于 Lucene 之上）在一个容易管理的包中提供了高性能的全文搜索功能，支持开箱即用地集群化扩展。您可以通过标准的 REST API 或从特定于编程语言的客户端库与 Elasticsearch 进行交互。
```

##  1 安装（window版）

```
解压elasticSearch；
运行elasticSearch/bin/elasticsearch.bat 文件 
访问 ：127.0.0.1：9200 进入代表安装成功。
```

### 1.2 安装服务

```
将Elasticsearch注册到window的服务上，不用每次启动Elasticsearch
使用CMD命令进入到bin目录，输入service install  >>  service start 
	安装：install
	启动：start
	停止：stop
	卸载：remove
	管理：manager
	
有时候无法启动服务，需要管理修改勾选Java的Use Default设置。
```

### 1.3 安装es head

```
方法一联网下载：
	1.elasticsearch/bin/plugin.bat -install mobz/elasticsearch-head
	2.运行es
	3.打开http://localhost:9200/_plugin/head/
方法二解压：
	1.https://github.com/mobz/elasticsearch-head 下载插件。
	2.将下载下的zip文件，解压缩到plugins/head目录下目录路径如下：elasticsearch-2.4.0/plugins/head/
	3.启动es bin/elasticsearch
	4、访问集群浏览器地址栏输入http://localhost:9200/_plugin/head/
```

### 1.4 安装CURL

```
http://curl.haxx.se/download.html下载 。解压后添加环境变量。 CMD 输入curl -help 有即使完成。
```

### 1.5 安装IK分词器

```
1）下载 https://github.com/medcl/elasticsearch-analysis-ik/tree/2.x
2）打包ik分词器	mvn clean 清空
3）mvn  packge 打包，会生成两个jar包
4）进入target/release目录，除了elasticsearch-analysis-ik-1.10.0和.zip文件。拷贝到%es%/plugins/analysis-ik
5）进入target/release/config 目录，将所有配置文件，复制 %es%/config 下
6）配置elasticsearch.yml文件 ，在%es%/config下添加：
	index.analysis.analyzer.ik.type: "ik"
7）重启es 和服务。发现ik分词器被加载 。
8）验证：
	http://localhost:9200/_analyze?analyzer=ik&pretty=true&text=我是中国人
```

## 2 CURL命令操作执行rest命令

### 2.1  创建一个索引

```
curl -XPUT "http://localhost:9200/blog01/"
```

### 2.2 插入一个文档

```
curl -XPUT "http://localhost:9200/blog01/article/1" -d  "{"""id""": """1""", """title""": """Whatiselasticsearch"""}"
```

### 2.3 查看文档

```
curl -XGET "http://localhost:9200/blog01/article/1"
```

### 2.4 更新文档

```
curl -XPUT "http://localhost:9200/blog01/article/1" -d "{"""id""": """1""", """title""": """Whatislucene"""}"
```

### 2.5 搜索文档

```
curl -XGET "http://localhost:9200/blog01/article/_search?q=title:'whatislucene'"
```

### 2.6 删除文档

```
curl -XDELETE "http://localhost:9200/blog01/article/1"
```

### 2.7 删除索引

```
curl -XDELETE "http://localhost:9200/blog01"
```

## 3 使用Java操作客户端

```
	运行一个 Java 应用程序和 Elasticsearch 时，有两种操作模式可供使用。该应用程序可在 Elasticsearch 集群中扮演更加主动或更加被动的角色。在更加主动的情况下（称为 Node Client），应用程序实例将从集群接收请求，确定哪个节点应处理该请求，就像正常节点所做的一样。（应用程序甚至可以托管索引和处理请求。）另一种模式称为 Transport Client，它将所有请求都转发到另一个 Elasticsearch 节点，由后者来确定最终目标。
```

### 3.1  条件查询QueryBuilder

```
boolQuery() 布尔查询，可以用来组合多个查询条件 
fuzzyQuery() 相似度查询 
matchAllQuery() 查询所有数据 
regexpQuery() 正则表达式查询 
termQuery() 词条查询 
wildcardQuery() 模糊查询 
```