# 08-离线阶段,Hive

```
上传到HDFS时，我们需要上传（-put）到HDFS的/user/hive/warehouse/下目录。这样才能映射。
本地模式
	set hive.exec.mode.local.auto=true;
```

## 1 分隔符

```
建立表的时候一定要根据结构化数据文件的数据文件的分隔符类型,指定分隔符.建表的字段个数和字段类型,要跟结构化的数据中的个数类型一致.一般使用默认的(ROW FORMAT DELIMITED fields terminated by ',')来指定分隔符.所以我们在创建表的时候必须指定分隔符。
create table t_t1(id int,name string) row format delimited fields terminated by ','; 指定“，”
DROP TABLE [IF EXISTS] t_t1;删除表
hadoop fs -put /root/hivedata/1.txt /user/hive/warehouse/student ; 上传
```

默认分隔符\001   Ctrl+V  Ctrl+A

###  1.1 复杂的建表语句

```
数据如下：
	 zhangsan	beijing,shanghai,tianjin,hangzhou
	 wangwu	shanghai,chengdu,wuhan,haerbin
	 
hive语句：
	create table complex_array(name string,work_locations array<string>) ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t' COLLECTION ITEMS TERMINATED BY ',';
```

## 2 Hive 分区（partitioned  by）

```
PARTITIONED BY (dt string   )  分区表的字段一定不能用已经有的字段名字.
分区字段是一个虚拟字段,不存放数据.分区字段在HDFS中就生成一个文件夹。一个表可以拥有多个分区。
分区字段的数据来自于装载分区表数据的时候指定的

创建分区表语句：
	create table t_user(id int,name string) partitioned by (contry string) row format delimited fields terminated by ',';
	
创建双分区表：
	create table day_hour_table (id int,name string) partitioned by (dt string,hour string) row format delimited fields terminated by ',';
```

###  2.1 效果

```
分区表的字段,在hdfs上效果就是在建立表的文件夹;下面又检测了子文件夹.这样就是为了把目的数据划分的细致.查询也会更加方便。
```

###  2.2 导入数据

```
我们在装载到分区表的时候，不能使用hadoop的put的命令。需要使用
	load data local  (local代表数据来自linux上本机上。不加代表不是。)
导入数据语句：
	load data local inpath '/root/hivedata/1.txt' into table t_user partition(country='USA');
	
内部表加载数据：
	LOAD DATA LOCAL INPATH '/root/hivedata/1.txt' OVERWRITE INTO TABLE t_user PARTITION(contry='CHN');
	
导入双分区：
	LOAD DATA LOCAL INPATH '/root/hivedata/1.txt' OVERWRITE INTO TABLE day_hour_table PARTITION(dt='2017-12-1',hour='12');
```

## 3 分桶表（cluster by  into num buckets）

```
默认是没有开启分桶设置。
指定开启分桶 ：set hive.enforce.bucketing = true;
分几桶：set mapreduce.job.reduces=4;
```

###  3.1 分桶表创建

```
设置分桶的时候需要设置开启功能.在分桶表在创建的时候,分桶表的字段必须是表中已经存储的字段.
分桶表创建：
	create table stu_buck(Sno int,Sname string,Sex string,Sage int,Sdept string) clustered by(Sno) sorted by(Sno DESC)into 4 bucketsrow format delimitedfields terminated by ',';
```

###  3.2 分桶数据导入

```
分桶表的数据导入:load data的方式不能保存导入成分桶的数据,没有分桶的效果.因为我们使用load data的时候没有调用MR程序,所以没有分桶的效果.

分桶表的数据 采用 insert+select 插入的数据来自于查询结果（查询时候执行了mr程序），对应MR当中的partitioneer导入数据的时候需要一个主表和一个桶表。
	对应mr当中的partitioner
	默认分桶规则 按照你指定的分桶字段clustered by哈希值 & 分桶的个数 set mapreduce.job.reduces=？
分桶表语句：
	insert overwrite table stu_buck
	select * from student cluster by(Sno);
	
如果对于某一列需要分桶，对于某一列需要排序：
insert overwrite table stu_buck
select * from student distribute by(Sno) sort by(Sage asc);

总结：
cluster（分且排序，必须一样）==distribute（分） + sort（排序）（可以不一样）  
```

###  3.3 分桶的好处(能提高效率)

```
MR 在查询的时候会采用笛卡尔积的查询方法 这样的效率特别慢.分桶表也是吧表锁映射的结构化数据文件分成更细致的部分。使用最多的就是join查询提高效率之上.按照校验字段进行分桶.
```

## 4内部表和外部表

```
exeranl  ：  CREATE [EXTERNAL] TABLE [IF NOT EXISTS] table_name 
在建表的时候加了就是external 就是外部表，没加就是内部表。默认内部表。在删除表的时候，内部表的元数据和数据会被一起删除，而外部表只删除元数据，不删除数据。内部表是受hive所管理的。外部表不受管理。
```
###  4.1内部表

```
创建内部表时，想要映射表，必须要把文件移动到创建时对应的目录下。这就是内部表的特色。
	create table student(Sno int,Sname string,Sex string,Sage int,Sdept string) row format delimited fields terminated by ',';
```

###  4.2外部表

```
外部表产生映射不需要把文件移动到指定的目录只需要location指定就可以了。数据在根目录下。
	create external table student_ext(Sno int,Sname string,Sex string,Sage int,Sdept string) row format delimited fields terminated by ',' location '/stu';
```

###  4.3 like 复制表

```
允许用户复制现有的表结构，但是不复制数据。
	CREATE [EXTERNAL] TABLE [IF NOT EXISTS] [db_name.]table_name LIKE existing_table;
```

## 5 修改表

###  5.1 分区操作

#### 	5.1.1 增加分区

```
ALTER TABLE table_name ADD PARTITION (dt='20170101') location '/user/hadoop/warehouse/table_name/dt=20170101'; //一次添加一个分区

ALTER TABLE table_name ADD PARTITION (dt='2008-08-08', country='us') location '/path/to/us/part080808' PARTITION (dt='2008-08-09', country='us') location '/path/to/us/part080809'; //一次添加多个分区
```

#### 	5.1.2 删除分区

```
ALTER TABLE table_name DROP IF EXISTS PARTITION (dt='2008-08-08');//删除一个分区

ALTER TABLE table_name DROP IF EXISTS PARTITION (dt='2008-08-08', country='us');//删除多个分区
```

#### 	5.1.3 修改分区

```
ALTER TABLE table_name PARTITION (dt='2008-08-08') RENAME TO PARTITION (dt='20080808');
```

### 5.2 列操作

#### 	5.2.1 添加列

```
ALTER TABLE table_name ADD|REPLACE COLUMNS (col_name STRING); 
  注：ADD 是代表新增一个字段，新增字段位置在所有列后面(partition 列前)
  REPLACE 则是表示替换表中所有字段。
```

#### 	5.2.2 修改列

```
已知：test_change (a int, b int, c int);

ALTER TABLE test_change CHANGE a a1 INT; //修改 a 字段名为a1

// will change column a's name to a1, a's data type to string, and put it after column b. The new 
table's structure is: b int, a1 string, c int
ALTER TABLE test_change CHANGE a a1 STRING AFTER b; //使用这句使列变成了。（b int, a1 string, c int）

// will change column b's name to b1, and put it as the first column. The new table's structure is: 
b1 int, a ints, c int
ALTER TABLE test_change CHANGE b b1 INT FIRST; 使用这句后变为（b1 int, a ints, c int）
```

###   5.3 表重命名

```
ALTER TABLE table_name RENAME TO new_table_name
```

###  5.4 显示命令

```
show tables; 显示当前数据库所有表。
show databases; 显示所有数据库。
show partitions table_name; 显示表分区信息，不是分区表执行报错。
show functions; 显示当前版本hive支持的所有的方法。
describe database database_name; 查看数据库相关信息
```

## 6  DML操作

###  6.1 Load  加载

```
在将数据加载到表中时，Hive 不会进行任何转换。加载操作是将数据文件移动到与 Hive表对应的位置的纯复制/移动操作。
语法结构
LOAD DATA [LOCAL] INPATH 'filepath' [OVERWRITE] INTO 
TABLE tablename [PARTITION (partcol1=val1, partcol2=val2 ...)]

LOCAL:如果指定了 LOCAL， load 命令将在本地文件系统中查找文件路径。如果没有指定 LOCAL 关键字，如果 filepath 指向的是一个完整的 URI，hive 
会直接使用这个 URI。
'filepath'：可以指定一个文件，也可以指定一个目录。
OVERWRITE：代表覆盖。则目标表（或者分区）中的内容会被删除，然后再将 filepath 指向的文件/目录中的内容添加到表/分区中。
```

###  6.2 Insert 插入

```
Hive 中 insert 主要是结合 select 查询语句使用，将查询结果插入到表中，例如：
  insert overwrite table stu_buck
  select * from student cluster by(Sno);
需要保证查询结果列的数目和需要插入数据表格的列数目一致.
如果查询出来的数据类型和插入表格对应的列数据类型不一致，将会进行转换，但是不能保证转换一定成功，转换失败的数据将会为 NULL。
```

####  	6.2.1 Multi Inserts 多重插入:

```
Multi Inserts 多重插入:即select一次就可以进行多重插入。
create table source_table (id int, name string) row format delimited fields terminated by ',';
create table test_insert1 (id int) row format delimited fields terminated by ',';
create table test_insert2 (name string) row format delimited fields terminated by ',';


from source_table                     
insert overwrite table test_insert1 
select id
insert overwrite table test_insert2
select name;
```

####  	6.2.2 Dynamic partition inserts 动态分区插入:

```
即可以将不同字段插入不同的表中。
set hive.exec.dynamic.partition=true;    #是否开启动态分区功能，默认false关闭。
set hive.exec.dynamic.partition.mode=nonstrict;   #动态分区的模式，默认strict，表示必须指定至少一个分区为静态分区，nonstrict模式表示允许所有的分区字段都可以使用动态分区。

INSERT OVERWRITE TABLE tablename PARTITION (partcol1[=val1], partcol2[=val2] ...) 
select_statement FROM from_statement
动态分区是通过位置来对应分区值的。原始表 select 出来的值和输出 partition的值的关系仅仅是通过位置来确定的，和名字并没有关系。
```

### 6.3 导出数据

```
数据写入到文件系统时进行文本序列化，且每列用^A 来区分，\n 为换行符。
将查询结果保存到指定的文件目录（可以是本地，也可以是hdfs）
insert overwrite local directory '/root' 
select * from t_p;
或者
insert overwrite directory '/aaa/test'
select * from t_p;
```

### 6.4 select语句

```
说明：
	1、order by 会对输入做全局排序，因此只有一个 reducer，会导致当输入规模较大时，需要较长的计算时间。
	2、sort by 不是全局排序，其在数据进入 reducer 前完成排序。因此，如果用 sort by 进行排序，并且设置 mapred.reduce.tasks>1，则 sort by 只保证每个 reducer 的输出有序，不保证全局有序。
	3、distribute by(字段)根据指定字段将数据分到不同的 reducer，分发算法是 hash 散列。
```

###  6.5 join

```
**left join   
select * from a left join b on a.id=b.id;
+-------+---------+-------+---------+--+
| a.id  | a.name  | b.id  | b.name  |
+-------+---------+-------+---------+--+
| 1     | a       | NULL  | NULL    |
| 2     | b       | 2     | bb      |
| 3     | c       | 3     | cc      |
| 4     | d       | NULL  | NULL    |
| 7     | y       | 7     | yy      |
| 8     | u       | NULL  | NULL    |
+-------+---------+-------+---------+--+

**right join
select * from a right join b on a.id=b.id;

select * from b right join a on b.id=a.id;
+-------+---------+-------+---------+--+
| a.id  | a.name  | b.id  | b.name  |
+-------+---------+-------+---------+--+
| 2     | b       | 2     | bb      |
| 3     | c       | 3     | cc      |
| 7     | y       | 7     | yy      |
| NULL  | NULL    | 9     | pp      |
+-------+---------+-------+---------+--+
```

###  