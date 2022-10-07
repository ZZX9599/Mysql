# MyCat

# 1:日志

## 1.1:错误日志

错误日志是 MySQL 中最重要的日志之一，它记录了当 mysqld 启动和停止时，以及服务器在运行过程中发生任何严重错误时的相关信息。当数据库出现任何故障导致无法正常使用时，建议首先查看此日志

查看日志位置：`show variables like '%log_error%'`

![image-20221006152241252](.\assets\image-20221006152241252.png)

## 1.2:二进制日志 

### 1:介绍

二进制日志（BINLOG）记录了所有的 DDL（数据定义语言）语句和 DML（数据操纵语言）语句，但 不包括数据查询（SELECT、SHOW）语句

作用：

①. 灾难时的数据恢复

②. MySQL的主从复制【主从复制原理就是基于二进制日志来实现的】

在MySQL8版本中，默认二进制日志是开启的

涉及到的参数`show variables like '%log_bin%`

![image-20221006153120644](.\assets\image-20221006153120644.png)

参数说明： 

log_bin_basename：当前数据库服务器的binlog日志的基础名称(前缀)，具体的binlog文件名需要再该

basename的基础上加上编号(编号从000001开始)

log_bin_index：binlog的索引文件，里面记录了当前服务器关联的binlog文件有哪些

![image-20221006153253370](.\assets\image-20221006153253370.png)

### 2:格式

MySQL服务器中提供了多种格式来记录二进制日志，具体格式及特点如下：

![image-20221006153329661](.\assets\image-20221006153329661.png)

`show variables like '%binlog_format%'`

![image-20221006153422508](.\assets\image-20221006153422508.png)

什么意思？假设我执行一条sql更新的语句影响了五行记录

如果是statement，则记录的是我这条SQL语句

如果是row，则记录的是我之前是什么数据，执行SQL之后变成了什么数据



### 3:删除

对于比较繁忙的业务系统，每天生成的binlog数据巨大，如果长时间不清除，将会占用大量磁盘空间

可以通过以下几种方式清理日志：

![image-20221006155124263](.\assets\image-20221006155124263.png)

也可以在mysql的配置文件中配置二进制日志的过期时间，设置了之后，二进制日志过期会自动删除

`show variables like '%binlog_expire_logs_seconds%'`



## 1.3:查询日志

查询日志中记录了客户端的`所有操作语句`，而二进制日志不包含查询数据的SQL语句

默认情况下：查询日志是未开启的

如果需要开启查询日志，可以修改MySQL的配置文件，添加如下内容

```
#该选项用来开启查询日志 ， 可选值 ： 0 或者 1 ； 0 代表关闭， 1 代表开启
general_log=1

#设置日志的文件名 ， 如果没有指定， 默认的文件名为 host_name.log
general_log_file=mysql_query.log
```

开启了查询日志之后，在MySQL的数据存放目录下就会出现 mysql_query.log 文件。之后所有的客户端的增删改查操作都会记录在该日志文件之中，长时间运行后，该日志文件将会非常的大



## 1.4:慢查询日志

慢查询日志记录了所有执行时间超过参数 long_query_time 设置值并且扫描记录数不小于 min_examined_row_limit 的所有的SQL语句的日志，默认未开启。long_query_time 默认为 10 秒，最小为 0， 精度可以到微秒。 如果需要开启慢查询日志，需要在MySQL的配置文件中配置如下参数：

```
#慢查询日志
slow_query_log=1

#执行时间参数
long_query_time=2
```

默认情况下，不会记录管理语句，也`不会记录不使用索引进行查找的查询`。可以使用更改此行为 

```
#记录执行较慢的管理语句
log_slow_admin_statements =1

#记录执行较慢的未使用索引的语句
log_queries_not_using_indexes = 1
```



# 2:主从复制

## 2.1:概述 

主从复制是指将主数据库的 DDL 和 DML 操作`通过二进制日志`传到从库服务器中，然后在从库上对这些日志重新执行（也叫重做），从而使得从库和主库的数据保持同步。 MySQL支持一台主库同时向多台从库进行复制， 从库同时也可以作为其他从服务器的主库，实现链状复制

MySQL 复制的优点主要包含以下三个方面： 

- 主库出现问题，可以快速切换到从库提供服务
- 实现读写分离，降低主库的访问压力
- 可以在从库中执行备份，以避免备份期间影响主库服务



## 2.2:原理

MySQL主从复制的核心就是 `二进制日志`，具体的过程如下：

![image-20221006160155062](.\assets\image-20221006160155062.png)

从上图来看，复制分成三步： 

1. Master 主库在事务提交时，会把`数据变更记录`在二进制日志文件 Binlog 中
2. 从库的IOThread`读取主库`的二进制日志文件 Binlog ，`写入到从库的中继日志 Relay Log`
3.  从库的SQLThread`重做中继日志`中的事件，将改变反映它自己的数据



## 2.3:搭建

![image-20221006160612873](.\assets\image-20221006160612873.png)

准备好两台服务器之后，在上述的两台服务器中分别安装好MySQL，并完成基础的初始化准备(安装、 密码配置等操作)工作。 其中： 

- 192.168.200.200 作为主服务器master 
- 192.168.200.201 作为从服务器slave



### 1:主库配置

1：修改配置文件

```shell
#mysql 服务ID，保证整个集群环境中唯一，取值范围：1 – 232-1，默认为1
server-id=1

#是否只读,1 代表只读, 0 代表既可以读又可以写
read-only=0

#忽略的数据, 指不需要同步的数据库
#binlog-ignore-db=mysql

#指定同步的数据库
#binlog-do-db=db01
```

2：重启MySQL服务器

```shell
systemctl restart mysqld
```

3：登录mysql，创建远程连接的账号，并授予主从复制权限

```shell
#创建用户，并设置密码，该用户可在任意主机连接该MySQL服务
CREATE USER 'ZZX'@'%' IDENTIFIED WITH mysql_native_password BY 'JXLZZX79';

#为 'ZZX'@'%' 用户分配主从复制权限
GRANT REPLICATION SLAVE ON *.* TO 'ZZX'@'%';
```

4：通过指令，查看二进制日志坐标

```shell
show master status 
```

![image-20221006161608635](.\assets\image-20221006161608635.png)

字段含义说明： 

- file : 从哪个日志文件开始推送日志文件 
- position ： 从哪个位置开始推送日志 【类似于redis的那个圈】
- binlog_ignore_db : 指定不需要同步的数据库



### 2:从库配置

1：修改配置文件

```shell
#mysql 服务ID，保证整个集群环境中唯一，取值范围：1 – 2^32-1，和主库不一样即可
server-id=2

#是否只读,1 代表只读, 0 代表读写
read-only=1
```

2：重新启动MySQL服务 

```shell
systemctl restart mysqld
```

3：登录mysql，设置主库配置 

```shell
CHANGE REPLICATION SOURCE TO SOURCE_HOST='192.168.200.200', SOURCE_USER='ZZX',
SOURCE_PASSWORD='JXLZZX79', SOURCE_LOG_FILE='binlog.000004',
SOURCE_LOG_POS=663;
```

上述是8.0.23中的语法。如果mysql是 8.0.23 之前的

```shell
CHANGE MASTER TO MASTER_HOST='192.168.200.200', MASTER_USER='ZZX',
MASTER_PASSWORD='JCLZZX79', MASTER_LOG_FILE='binlog.000004',
MASTER_LOG_POS=663;
```

![image-20221006161700625](.\assets\image-20221006161700625.png)

4：开启同步操作

```shell
#8.0.22之后
start replica ; 

#8.0.22之前
start slave ; 
```

5：查看主从同步状态

```shell
#8.0.22之后
show replica status ; 

#8.0.22之前
show slave status ; 
```



## 2.4:测试

1：在主库 192.168.200.200 上创建数据库、表，并插入数据

2：在从库 192.168.200.201 中查询数据，验证主从是否同步



# 3:分库分表

## 3.1:问题引入

我们现在都是一个数据库，就算是主从复制，也是多个数据库存储相同的数据，只是很多份罢了，数据库的压力会很大，若采用单数据库进行数据存 储，存在以下性能瓶颈

- IO瓶颈：我们知道数据库的内存大都分给了缓存，如果热点数据太多，数据库的缓存不足，就会产生大

  量的，磁盘IO，效率较低。 请求数据太多，带宽不够，造成网络IO瓶颈

- CPU瓶颈：排序、分组、连接查询、聚合统计等SQL会耗费大量的CPU资源，请求数太多，CPU出现瓶颈

为了解决上述问题，我们需要对数据库进行分库分表处理

![image-20221006162519391](.\assets\image-20221006162519391.png)

分库分表的`中心思想都是将数据分散存储，使得单一数据库/表的数据量变小来缓解单一数据库的性能` 问题

从而达到提升数据库性能的目的



## 3.2:拆分策略

分库分表的形式，主要是两种：垂直拆分和水平拆分。而拆分的粒度，一般又分为分库和分表，所以组成的拆分策略最终如下：

![image-20221006162701550](.\assets\image-20221006162701550.png)



### 1:垂直分库

![image-20221006162758889](.\assets\image-20221006162758889.png)

垂直分库：以表为依据，根据业务将不同表拆分到不同库中

比如上面：把用户相关的表存放到一个库，订单相关的放到一个库，商品相关的放到一个库

特点： 

- 每个库的表结构都不一样
- 每个库的数据也不一样
- 所有库的并集是全量数据



### 2:垂直分表

![image-20221006163009039](.\assets\image-20221006163009039.png)

垂直分表：以字段为依据，根据字段属性将不同字段拆分到不同表中

 特点： 

- 每个表的结构都不一样
- 每个表的数据也不一样，一般通过一列（主键/外键）关联
- 所有表的并集是全量数据



### 3:水平分库

![image-20221006163132404](.\assets\image-20221006163132404.png)

水平分库：以字段为依据，按照一定策略，将一个库的数据拆分到多个库中

每个库的表结构是一样的，但是数据是不一样的

特点： 

- 每个库的表结构都一样
- 每个库的数据都不一样
- 所有库的并集是全量数据



### 4:水平分表

![image-20221006163312088](.\assets\image-20221006163312088.png)



水平分表：以字段为依据，按照一定策略，将一个表的数据拆分到多个表中

特点： 

- 每个表的表结构都一样
- 每个表的数据都不一样
- 所有表的并集是全量数据



在业务系统中，为了缓解磁盘IO及CPU的性能瓶颈，到底是垂直拆分，还是水平拆分

具体是分库，还是分表，都需要根据具体的业务需求具体分析



### 5:实现技术

之前只需要访问一个数据库，现在需要访问多个数据库，怎么实现

shardingJDBC：基于AOP原理，在应用程序中`对本地执行的SQL进行拦截`，解析、改写、路由处理到对应的数据库。需要自行编码配置实现，只支持java语言，性能较高

MyCat：数据库分库分表中间件，不用调整代码即可实现分库分表，支持多种语言，性能不及前者

我们使用Mycat

![image-20221006163736884](.\assets\image-20221006163736884.png)



直接访问 Mycat 就可以了，是无感知的，集体的内容由Mycat来处理



# 4:Mycat基本

## 4.1:介绍 

Mycat是开源的、活跃的、基于Java语言编写的MySQL数据库中间件。可像使用mysql一样来使用mycat，就好像是一个伪装的Mysql，对于开发人员来说根本感觉不到mycat的存在。 开发人员只需连接MyCat即可，而具体底层用到几台数据库，每一台数据库服务器里面存储了什么数据，都无需关心。 具体的分库分表的策略，只需要在MyCat中配置即可

<img src=".\assets\image-20221006164303641.png" alt="image-20221006164303641" style="zoom: 50%;" />

## 4.2:安装 

Mycat是采用java语言开发的开源的数据库中间件，支持Windows和Linux运行环境，下面介绍 MyCat的Linux中的环境搭建。我们需要在准备好的服务器中安装如下软件。 MySQL JDK Mycat

![image-20221006164409887](.\assets\image-20221006164409887.png)



1：上传压缩包

2：解压该压缩包



- bin : 存放可执行文件，用于启动停止mycat 
- conf：存放mycat的配置文件
- lib：存放mycat的项目依赖包（jar） 
- logs：存放mycat的日志文件



## 4.3:概念介绍

在MyCat的整体结构中，分为两个部分：

- 上面的逻辑结构，也就是Mycat【并不存储数据】
- 下面的物理结构【具体的真实的服务器】

![image-20221006164923758](.\assets\image-20221006164923758.png)

在MyCat的逻辑结构主要负责逻辑库、逻辑表、分片规则、分片节点等逻辑结构的处理，而具体的数据存储还是在物理结构，也就是数据库服务器中存储的。 在后面讲解MyCat入门以及MyCat分片时，还会讲到上面所提到的概念



## 4.3:需求 

由于 tb_order 表中数据量很大，磁盘IO及容量都到达了瓶颈，现在需要对 tb_order 表进行数 据分片，分为三个数据节点，每一个节点主机位于不同的服务器上, 具体的结构，参考下图：

![image-20221006165154867](.\assets\image-20221006165154867.png)

上面是水平分表



## 4.4:实现

### 1: 环境准备

准备3台服务器： 

- 192.168.200.210：MyCat中间件服务器，同时也是第一个分片服务器
- 192.168.200.213：第二个分片服务器
- 192.168.200.214：第三个分片服务器

并且在上述3台数据库中创建数据库 db01



### 2:配置

1：Mycat核心配置：conf下的`schema.xml`

在schema.xml中配置逻辑库、逻辑表、数据节点、节点主机等相关信息。具体的配置如下

```xml
<?xml version="1.0"?>
<!DOCTYPE mycat:schema SYSTEM "schema.dtd">
<mycat:schema xmlns:mycat="http://io.mycat/">

<!--逻辑库DB01-->
<schema name="DB01" checkSQLschema="true" sqlMaxLimit="100">
	<!--逻辑表TB_ORDER，具体的分片规则auto-sharding-long-->
	<table name="TB_ORDER" dataNode="dn1,dn2,dn3" rule="auto-sharding-long"/>
</schema>

<!--数据节点，把dhost1关联到这台服务器的db01数据库-->
<dataNode name="dn1" dataHost="dhost1" database="db01" />
<!--数据节点，把dhost2关联到这台服务器的db01数据库-->
<dataNode name="dn2" dataHost="dhost2" database="db01" />
<!--数据节点，把dhost3关联到这台服务器的db01数据库-->    
<dataNode name="dn3" dataHost="dhost3" database="db01" />

<!--节点主机dhost1-->
<dataHost name="dhost1" maxCon="1000" minCon="10" balance="0" writeType="0" dbType="mysql" dbDriver="jdbc" switchType="1" slaveThreshold="100">
<heartbeat>select user()</heartbeat>
<!--数据库连接信息-->
<writeHost host="master" url="jdbc:mysql://192.168.200.210:3306?useSSL=false&amp;serverTimezone=Asia/Shanghai&amp;characterEncoding=utf8"
user="root" password="1234" />
</dataHost>

<!--节点主机dhost2-->
<dataHost name="dhost2" maxCon="1000" minCon="10" balance="0"
writeType="0" dbType="mysql" dbDriver="jdbc" switchType="1"
slaveThreshold="100">
<heartbeat>select user()</heartbeat>
<!--数据库连接信息-->
<writeHost host="master" url="jdbc:mysql://192.168.200.213:3306?
useSSL=false&amp;serverTimezone=Asia/Shanghai&amp;characterEncoding=utf8"
user="root" password="1234" />
</dataHost>

<!--节点主机dhost3-->
<dataHost name="dhost3" maxCon="1000" minCon="10" balance="0"
writeType="0" dbType="mysql" dbDriver="jdbc" switchType="1"
slaveThreshold="100">
<heartbeat>select user()</heartbeat>
<!--数据库连接信息-->    
<writeHost host="master" url="jdbc:mysql://192.168.200.214:3306?
useSSL=false&amp;serverTimezone=Asia/Shanghai&amp;characterEncoding=utf8"
user="root" password="1234" />
</dataHost>
</mycat:schema>
```

2：conf下的`server.xml`

需要在server.xml中配置用户名、密码，以及用户的访问权限信息，具体的配置如下：

```xml
<user name="root" defaultAccount="true">
<property name="password">123456</property>
<property name="schemas">DB01</property>
</user>

<user name="user">
<property name="password">123456</property>
<property name="schemas">DB01</property>
<property name="readOnly">true</property>
</user>
```

上述的配置表示，定义了两个用户 root 和 user ，这两个用户都可以访问 DB01 这个逻辑库，访问密码都是123456，但是root用户访问DB01逻辑库，既可以读，又可以写，但是 user用户访问 DB01逻辑库是只读的



### 3:测试

配置完毕后，先启动涉及到的3台分片服务器，然后启动MyCat服务器。切换到Mycat的安装目录，执行如下指令，启动Mycat：

```shell
#启动
bin/mycat start

#停止
bin/mycat stop
```

Mycat启动之后，占用端口号 8066



1：连接MyCat 通过如下指令，就可以连接并登陆MyCat

```shell
mysql -h 192.168.200.210 -P 8066 -uroot -p123456
```

我们看到我们是通过MySQL的指令来连接的MyCat，因为MyCat在底层实际上是模拟了MySQL的协议



2：数据测试 然后就可以在MyCat中来创建表，并往表结构中插入数据，查看数据在MySQL中的分布情况

```sql
CREATE TABLE TB_ORDER (
	id BIGINT(20) NOT NULL,
	title VARCHAR(100) NOT NULL ,
	PRIMARY KEY (id)
) ENGINE=INNODB DEFAULT CHARSET=utf8 ;

INSERT INTO TB_ORDER(id,title) VALUES(1,'goods1');
INSERT INTO TB_ORDER(id,title) VALUES(2,'goods2');
INSERT INTO TB_ORDER(id,title) VALUES(3,'goods3');
INSERT INTO TB_ORDER(id,title) VALUES(1,'goods1');
INSERT INTO TB_ORDER(id,title) VALUES(2,'goods2');
INSERT INTO TB_ORDER(id,title) VALUES(3,'goods3');
INSERT INTO TB_ORDER(id,title) VALUES(5000000,'goods5000000');
INSERT INTO TB_ORDER(id,title) VALUES(10000000,'goods10000000');
INSERT INTO TB_ORDER(id,title) VALUES(10000001,'goods10000001');
INSERT INTO TB_ORDER(id,title) VALUES(15000000,'goods15000000');
INSERT INTO TB_ORDER(id,title) VALUES(15000001,'goods15000001');
```

经过测试，我们发现，在往 TB_ORDER 表中插入数据时： 

- 如果id的值在1-500w之间，数据将会存储在第一个分片数据库中
- 如果id的值在500w-1000w之间，数据将会存储在第二个分片数据库中
- 如果id的值在1000w-1500w之间，数据将会存储在第三个分片数据库中
- 如果id的值超出1500w，在插入数据时，将会报错



为什么会出现这种现象，数据到底落在哪一个分片服务器到底是如何决定的呢？ 这是由逻辑表配置时的一个参数 rule 决定的，是conf下面的Rule文件决定的，而这个参数配置的就是分片规则，关于分片规则的配置，后面会详细讲解



## 4.5:配置详解

<img src=".\assets\image-20221006172106693.png" alt="image-20221006172106693" style="zoom:150%;" />

主要包含以下三组标签： schema标签 datanode标签 datahost标签



### 1: schema.xml

schema.xml 作为MyCat中最重要的配置文件之一 

涵盖了MyCat的逻辑库 、 逻辑表 、 分片规则、分片节点及数据源的配置



1：schema 定义逻辑库

![image-20221006172235576](.\assets\image-20221006172235576.png)

schema 标签用于定义 MyCat实例中的逻辑库 , 一个MyCat实例中, 可以有多个逻辑库 , 可以通 过 schema 标签来划分不同的逻辑库。MyCat中的逻辑库的概念，等同于MySQL中的数据库概念 , 需要操作某个逻辑库下的表时, 也需要切换逻辑库(use xxx)



核心属性： 

- name：指定自定义的逻辑库库名 
- sqlMaxLimit：如果未指定limit进行查询，列表查询模式查询多少条记录



2：schema 中的table定义逻辑表

![image-20221006172546242](.\assets\image-20221006172546242.png)

table 标签定义了MyCat中逻辑库schema下的逻辑表 , 所有需要拆分的表都需要在table标签中定义

核心属性： 

- name：定义逻辑表表名，在该逻辑库下唯一 
- dataNode：定义逻辑表所属的dataNode，需要与dataNode标签中name对应；多个 dataNode逗号分隔
- rule：分片规则的名字，分片规则名字是在rule.xml中定义的 
- primaryKey：逻辑表对应真实表的主键 
- type：逻辑表的类型，目前逻辑表只有全局表和普通表
  - 如果未配置，就是普通表
  - 配置为 global，就是全局表



3：datanode标签

![image-20221006172743080](.\assets\image-20221006172743080.png)

核心属性： 

- name：定义数据节点名称 
- dataHost：数据库实例主机名称，引用自 dataHost 标签中name属性 
- database：定义分片所属数据库



4：datahost标签

![image-20221006172834902](.\assets\image-20221006172834902.png)

该标签在MyCat逻辑库中作为底层标签存在, 直接定义了具体的数据库实例、读写分离、心跳语句

核心属性： 

- name：唯一标识，供上层标签使用 
- maxCon/minCon：最大连接数/最小连接数 
- balance：负载均衡策略，取值 0,1,2,3 
- writeType：写操作分发方式
  - 0：写操作转发到第一个writeHost，第一个挂了，切换到第二个
  - 1：写操作随机分发到配置的writeHost） 
- dbDriver：数据库驱动，支持 native、jdbc



### 2:rule.xml

rule.xml中定义所有拆分表的规则, 在使用过程中可以灵活的使用分片算法, 或者对同一个分片算法使用不同的参数, 它让分片过程可配置化。主要包含两类标签：tableRule、Function

![image-20221006173102287](.\assets\image-20221006173102287.png)



### 3:server.xml

server.xml配置文件包含了MyCat的系统配置信息，主要有两个重要的标签：system、user



1：system标签

![image-20221006173158749](.\assets\image-20221006173158749.png)



2：user标签 

配置MyCat中的用户、访问密码，以及用户针对于逻辑库、逻辑表的权限信息，具体的权限描述方式及配置说明如下：

![image-20221006173351248](.\assets\image-20221006173351248.png)



## 4.6:MyCat分片

### 4.6.1: 实现垂直分库

在业务系统中， 涉及以下表结构 ，但是由于用户与订单每天都会产生大量的数据, 单台服务器的数据 存储及处理能力是有限的, 可以对数据库表进行拆分，原有的数据库表如下

![image-20221006173610714](.\assets\image-20221006173610714.png)

现在考虑将其进行垂直分库操作，将商品相关的表拆分到一个数据库服务器，订单表拆分的一个数据库 服务器，用户及省市区表拆分到一个服务器。最终结构如下：

![image-20221006173636800](.\assets\image-20221006173636800.png)

准备三台服务器，IP地址如图所示

![image-20221006173717526](.\assets\image-20221006173717526.png)

并且在192.168.200.210，192.168.200.213，192.168.200.214上面创建数据库 shopping



#### 1: schema.xml

```xml
<schema name="SHOPPING" checkSQLschema="true" sqlMaxLimit="100">
    <!--节点一相关-->
	<table name="tb_goods_base" dataNode="dn1" primaryKey="id" />
	<table name="tb_goods_brand" dataNode="dn1" primaryKey="id" />
	<table name="tb_goods_cat" dataNode="dn1" primaryKey="id" />
	<table name="tb_goods_desc" dataNode="dn1" primaryKey="goods_id" />
	<table name="tb_goods_item" dataNode="dn1" primaryKey="id" />
    
    <!--节点二相关-->
	<table name="tb_order_item" dataNode="dn2" primaryKey="id" />
	<table name="tb_order_master" dataNode="dn2" primaryKey="order_id" />
	<table name="tb_order_pay_log" dataNode="dn2" primaryKey="out_trade_no" />
    
    <!--节点三相关-->
	<table name="tb_user" dataNode="dn3" primaryKey="id" />
	<table name="tb_user_address" dataNode="dn3" primaryKey="id" />
	<table name="tb_areas_provinces" dataNode="dn3" primaryKey="id"/>
	<table name="tb_areas_city" dataNode="dn3" primaryKey="id"/>
	<table name="tb_areas_region" dataNode="dn3" primaryKey="id"/>
</schema>

<!--三个节点-->
<dataNode name="dn1" dataHost="dhost1" database="shopping" />
<dataNode name="dn2" dataHost="dhost2" database="shopping" />
<dataNode name="dn3" dataHost="dhost3" database="shopping" />

<dataHost name="dhost1" maxCon="1000" minCon="10" balance="0" writeType="0" dbType="mysql" dbDriver="jdbc" switchType="1" slaveThreshold="100">
<heartbeat>select user()</heartbeat>
<writeHost host="master" url="jdbc:mysql://192.168.200.210:3306?
useSSL=false&amp;serverTimezone=Asia/Shanghai&amp;characterEncoding=utf8"
user="root" password="1234" />
</dataHost>

<dataHost name="dhost2" maxCon="1000" minCon="10" balance="0" writeType="0" dbType="mysql" dbDriver="jdbc" switchType="1" slaveThreshold="100">
<heartbeat>select user()</heartbeat>
<writeHost host="master" url="jdbc:mysql://192.168.200.213:3306?
useSSL=false&amp;serverTimezone=Asia/Shanghai&amp;characterEncoding=utf8"
user="root" password="1234" />
</dataHost>

<dataHost name="dhost3" maxCon="1000" minCon="10" balance="0" writeType="0" dbType="mysql" dbDriver="jdbc" switchType="1" slaveThreshold="100">
<heartbeat>select user()</heartbeat>
<writeHost host="master" url="jdbc:mysql://192.168.200.214:3306?
useSSL=false&amp;serverTimezone=Asia/Shanghai&amp;characterEncoding=utf8"
user="root" password="1234" />
</dataHost>
```



#### 2:server.xml

```xml
<user name="root" defaultAccount="true">
<property name="password">123456</property>
<property name="schemas">SHOPPING</property>
</user>

<user name="user">
<property name="password">123456</property>
<property name="schemas">SHOPPING</property>
<property name="readOnly">true</property>
</user>
```



#### 3:测试

连接mycat，发现有库，有表，但是都是虚拟的，目前真实的服务器并不存在，我们现在导入数据

将表结构及对应的测试数据导入之后，可以检查一下各个数据库服务器中的表结构分布情况

在MyCat的命令行中，当我们执行以下多表联查的SQL语句【处于一个节点时】可以正常查询出数据

但是现在存在一个问题，订单相关的表结构是在 192.168.200.213 数据库服务器中，而省市区的数据库表是

在 192.168.200.214 数据库服务器中。执行这种的跨节点的多表联查，那么在MyCat中执行是否可以成功呢



经过测试，我们看到，SQL语句执行报错。原因就是因为MyCat在执行该SQL语句时，需要往具体的数据库服务器中路由，而当前没有一个数据库服务器完全包含了订单以及省市区的表结构，造成SQL语句报错，对于上述的这种现象，我们如何来解决呢？ 下面我们介绍的全局表，就可以轻松解决这个问题



#### 4:全局表

对于省、市、区/县表tb_areas_provinces , tb_areas_city , tb_areas_region，是属于数据字典表，在多个业务模块中都可能会遇到，可以将其设置为全局表，利于业务操作。 修改schema.xml中的逻辑表的配置，修改 tb_areas_provinces、tb_areas_city、 tb_areas_region 三个逻辑表，增加 type 属性，配置为global，就代表该表是全局表，就会在所涉及到的dataNode中创建这个表。对于当前配置来说，也就意味着所有的节点中都有该表了

```xml
<table name="tb_areas_provinces" dataNode="dn1,dn2,dn3" primaryKey="id" type="global"/>

<table name="tb_areas_city" dataNode="dn1,dn2,dn3" primaryKey="id" type="global"/>

<table name="tb_areas_region" dataNode="dn1,dn2,dn3" primaryKey="id" type="global"/>
```



现在就相当于是：

![image-20221006175123882](.\assets\image-20221006175123882.png)

之前的跨节点的联查可以成功，并且在MyCat中更新全局表的时候，我们可以看到，所有分片节点中的数据都发生了变化，每个节点的全局表数据时刻保持一致



### 4.6.2:水平分表

在业务系统中, 有一张表(日志表), 业务系统每天都会产生大量的日志数据 , 单台服务器的数据存 储及处理能力是有限的, 可以对数据库表进行拆分

![image-20221007094928131](.\assets\image-20221007094928131.png)

![image-20221007095000520](.\assets\image-20221007095000520.png)

并且，在三台数据库服务器中分表创建一个数据库itcast



#### 1:schema.xml

```xml
<schema name="ITCAST" checkSQLschema="true" sqlMaxLimit="100">
	<table name="tb_log" dataNode="dn4,dn5,dn6" primaryKey="id" rule="mod-long" />
</schema>

<dataNode name="dn4" dataHost="dhost1" database="itcast" />
<dataNode name="dn5" dataHost="dhost2" database="itcast" />
<dataNode name="dn6" dataHost="dhost3" database="itcast" />
```

tb_log表最终落在3个节点中，分别是 dn4、dn5、dn6 

而具体的数据分别存储在 dhost1、 dhost2、dhost3的itcast数据库中



#### 2:server.xml

 配置root用户既可以访问 SHOPPING 逻辑库，又可以访问ITCAST逻辑库

```xml
<user name="root" defaultAccount="true">
	<property name="password">123456</property>
	<property name="schemas">SHOPPING,ITCAST</property>
</user>
```

#### 3:测试

配置完毕后，重新启动MyCat，然后在mycat的命令行中

执行创建表的SQL、并插入数据，查看数据分布情况



# 5:Mycat分片规则

## 5.1:范围分片

根据指定的字段及其配置的范围与数据节点的对应情况， 来决定该数据属于哪一个分片

![image-20221007095626632](.\assets\image-20221007095626632.png)

### 1:schema.xml

```xml
<table name="TB_ORDER" dataNode="dn1,dn2,dn3" rule="auto-sharding-long" />

<dataNode name="dn1" dataHost="dhost1" database="db01" />
<dataNode name="dn2" dataHost="dhost2" database="db01" />
<dataNode name="dn3" dataHost="dhost3" database="db01" />
```

### 2:rule.xml

```xml
<tableRule name="auto-sharding-long">
	<rule>
		<columns>id</columns>
		<algorithm>rang-long</algorithm>
	</rule>
</tableRule>

<function name="rang-long" class="io.mycat.route.function.AutoPartitionByLong">
	<property name="mapFile">autopartition-long.txt</property>
	<property name="defaultNode">0</property>
</function>
```

分片规则配置属性含义：

![image-20221007095901401](.\assets\image-20221007095901401.png)

在rule.xml中配置分片规则时，关联了一个映射配置文件 autopartition-long.txt，该配置文 件的配置如下：

```
# range start-end ,data node index
# K=1000,M=10000.

0-500M=0
500M-1000M=1
1000M-1500M=2
```

### 3:含义

0-500万之间的值，存储在0号数据节点(数据节点的索引从0开始) 

500万-1000万之间的 数据存储在1号数据节点 

1000万-1500万的数据节点存储在2号节点

该分片规则，主要是针对于数字类型的字段适用



## 5.2: 取模分片

根据指定的字段值与节点数量进行求模运算，根据运算结果， 来决定该数据属于哪一个分片

![image-20221007100059225](.\assets\image-20221007100059225.png)

### 1:schema.xml

```xml
<table name="tb_log" dataNode="dn4,dn5,dn6" primaryKey="id" rule="mod-long" />
<dataNode name="dn4" dataHost="dhost1" database="itcast" />
<dataNode name="dn5" dataHost="dhost2" database="itcast" />
<dataNode name="dn6" dataHost="dhost3" database="itcast" />
```



### 3:rule.xml

```xml
<tableRule name="mod-long">
	<rule>
		<columns>id</columns>
		<algorithm>mod-long</algorithm>
	</rule>
</tableRule>

<function name="mod-long" class="io.mycat.route.function.PartitionByMod">
    <!--节点数量，用于取模-->
	<property name="count">3</property>
</function>
```

该分片规则，主要是针对于数字类型的字段适用。 在前面水平拆分的演示中，我们选择的就是取模分片

![image-20221007101242749](.\assets\image-20221007101242749.png)

## 5.3:一致性hash分片

前面的范围和取模都适用于数字，并且都是根据自身的值来进行判断，hash则是根据自身算出结果进行判断

所谓一致性哈希，相同的哈希因子计算值总是被划分到相同的分区表中，不会因为分区节点的增加而改变原来数据的分区位置，有效的解决了分布式数据的拓容问题

![image-20221007100438121](.\assets\image-20221007100438121.png)

### 1:schema.xml

```xml
<table name="tb_order" dataNode="dn4,dn5,dn6" rule="sharding-by-murmur" />
<dataNode name="dn4" dataHost="dhost1" database="itcast" />
<dataNode name="dn5" dataHost="dhost2" database="itcast" />
<dataNode name="dn6" dataHost="dhost3" database="itcast" />
```

### 2:rule.xml

```xml
<tableRule name="sharding-by-murmur">
	<rule>
		<columns>id</columns>
		<algorithm>murmur</algorithm>
	</rule>
</tableRule>

<function name="murmur" class="io.mycat.route.function.PartitionByMurmurHash">
	<property name="seed">0</property><!-- 默认是0 -->
	<property name="count">3</property>
	<property name="virtualBucketTimes">160</property>
</function>
```

![image-20221007101421520](.\assets\image-20221007101421520.png)



## 5.4:枚举分片

通过在配置文件中配置可能的枚举值, 指定数据分布到不同数据节点上, 本规则适用于按照省份、性别、状态拆分数据等业务

![image-20221007100721822](.\assets\image-20221007100721822.png)

### 1:schema.xml

```xml
<table name="tb_user" dataNode="dn4,dn5,dn6" rule="sharding-by-intfile-enumstatus"/>

<dataNode name="dn4" dataHost="dhost1" database="itcast" />
<dataNode name="dn5" dataHost="dhost2" database="itcast" />
<dataNode name="dn6" dataHost="dhost3" database="itcast" />
```



### 2:rule.xml

```xml
<tableRule name="sharding-by-intfile">
	<rule>
		<columns>sharding_id</columns>
		<algorithm>hash-int</algorithm>
	</rule>
</tableRule>

<!-- 自己增加 tableRule -->
<tableRule name="sharding-by-intfile-enumstatus">
	<rule>
		<columns>status</columns>
		<algorithm>hash-int</algorithm>
	</rule>
</tableRule>

<function name="hash-int" class="io.mycat.route.function.PartitionByFileMap">
<property name="defaultNode">2</property>
<property name="mapFile">partition-hash-int.txt</property>
</function>
```

partition-hash-int.txt ，内容如下 :

```
1=0
2=1
3=2
```

枚举为1，进入0节点

枚举为2，进入1节点

枚举为3，进入2节点



![image-20221007101208301](.\assets\image-20221007101208301.png)



## 5.5:应用指定算法

运行阶段由应用自主决定路由到那个分片 , 直接根据字符子串（求出的字串必须是数字）计算分片号

![image-20221007101556615](.\assets\image-20221007101556615.png)



### 1:schema.xml

```xml
<table name="tb_app" dataNode="dn4,dn5,dn6" rule="sharding-by-substring" />

<dataNode name="dn4" dataHost="dhost1" database="itcast" />
<dataNode name="dn5" dataHost="dhost2" database="itcast" />
<dataNode name="dn6" dataHost="dhost3" database="itcast" />
```

### 2:rule.xml

```xml
<tableRule name="sharding-by-substring">
	<rule>
		<columns>id</columns>
		<algorithm>sharding-by-substring</algorithm>
	</rule>
</tableRule>

<function name="sharding-by-substring"
	class="io.mycat.route.function.PartitionDirectBySubString">
	
	<property name="startIndex">0</property> <!-- zero-based -->
	<property name="size">2</property>
	<property name="partitionCount">3</property>
	<property name="defaultPartition">0</property>
</function>
```

![image-20221007101833315](.\assets\image-20221007101833315.png)

示例说明 : id=05-100000002 , 在此配置中代表根据id中从 startIndex=0，开始，截取siz=2位数字即 05，05就是获取的分区，如果没找到对应的分片则默认分配到defaultPartition



## 5.6:固定分片hash算法

该算法类似于十进制的求模运算，但是`为二进制的操作`，例如，取 id 的二进制低 10 位 与 1111111111 进行位 & 运算，位与运算最小值为 0000000000，最大值为1111111111，转换为十 进制，也就是位于0-1023之间

![image-20221007102053022](.\assets\image-20221007102053022.png)



同时为1，则为1，存在0则为0

特点： 

- 如果是求模，连续的值，分别分配到各个不同的分片
- 但是此算法会将连续的值可能分配到相同的分片，降低事务处理的难度
- 分片字段必须为数字类型。



### 1:schema.xml

```xml
<table name="tb_longhash" dataNode="dn4,dn5,dn6" rule="sharding-by-long-hash" />

<dataNode name="dn4" dataHost="dhost1" database="itcast" />
<dataNode name="dn5" dataHost="dhost2" database="itcast" />
<dataNode name="dn6" dataHost="dhost3" database="itcast" />
```



### 2:rule.xml

```xml
<tableRule name="sharding-by-long-hash">
	<rule>
		<columns>id</columns>
		<algorithm>sharding-by-long-hash</algorithm>
	</rule>
</tableRule>

<!-- 分片总长度为1024，count与length数组长度必须一致； -->
<function name="sharding-by-long-hash"
class="io.mycat.route.function.PartitionByLong">
<property name="partitionCount">2,1</property>
<property name="partitionLength">256,512</property>
</function>
```

`2*256 + 1*512 = 1024`

底层初始化一个长度1024的数组

![image-20221007102504130](.\assets\image-20221007102504130.png)

![image-20221007102533176](.\assets\image-20221007102533176.png)

约束 : 

- 分片长度 : 默认最大2^10 , 为 1024
- count, length的数组长度必须是一致的
- 以上分为三个分区:0-255,256-511,512-1023



## 5.7:字符串hash解析算法

截取字符串中的指定位置的子字符串，进行hash算法，算出hash值，在进行与运算，算出分片

![image-20221007102638623](.\assets\image-20221007102638623.png)

### 1:schema.xml

```xml
<table name="tb_strhash" dataNode="dn4,dn5" rule="sharding-by-stringhash" />

<dataNode name="dn4" dataHost="dhost1" database="itcast" />
<dataNode name="dn5" dataHost="dhost2" database="itcast" />
```



### 2:rule.xml

```xml
<tableRule name="sharding-by-stringhash">
	<rule>
		<columns>name</columns>
		<algorithm>sharding-by-stringhash</algorithm>
	</rule>
</tableRule>

<function name="sharding-by-stringhash" class="io.mycat.route.function.PartitionByString">
	<property name="partitionLength">512</property> <!-- zero-based -->
	<property name="partitionCount">2</property>
	<property name="hashSlice">0:2</property>
</function>
```

`0代表str.length()`

分片长度 2，每个长度是512

![image-20221007103015295](.\assets\image-20221007103015295.png)

![image-20221007103045878](.\assets\image-20221007103045878.png)



## 5.8: 按天分片

![image-20221007103124229](.\assets\image-20221007103124229.png)

### 1:schema.xml

```xml
<table name="tb_datepart" dataNode="dn4,dn5,dn6" rule="sharding-by-date" />
<dataNode name="dn4" dataHost="dhost1" database="itcast" />
<dataNode name="dn5" dataHost="dhost2" database="itcast" />
<dataNode name="dn6" dataHost="dhost3" database="itcast" />
```



### 2:rule.xml

```xml
<tableRule name="sharding-by-date">
	<rule>
		<columns>create_time</columns>
		<algorithm>sharding-by-date</algorithm>
	</rule>
</tableRule>

<function name="sharding-by-date" class="io.mycat.route.function.PartitionByDate">
	<property name="dateFormat">yyyy-MM-dd</property>
	<property name="sBeginDate">2022-01-01</property>
	<property name="sEndDate">2022-01-30</property>
	<property name="sPartionDay">10</property>
</function>
```

从开始时间开始，每10天为一个分片，到达结束时间之后，会重复开始分片插入，也就是

- 2022-02-01~2022-02-10  -----------> 0
- 2022-02-11~2022-02-20  -----------> 1
- 2022-02-21~2022-02-30  -----------> 3

配置表的 dataNode 的分片，必须和分片规则数量一致

例如 2022-01-01 到 2022-12-31 ，每10天一个分片，则一共需要37个分片

![image-20221007103547769](.\assets\image-20221007103547769.png)



## 5.9:自然月分片

![image-20221007103650695](.\assets\image-20221007103650695.png)

- 四月份进0
- 五月份进1
- 六月份进2
- 以此类推

### 1:schema.xml

```xml
<table name="tb_monthpart" dataNode="dn4,dn5,dn6" rule="sharding-by-month" />

<dataNode name="dn4" dataHost="dhost1" database="itcast" />
<dataNode name="dn5" dataHost="dhost2" database="itcast" />
<dataNode name="dn6" dataHost="dhost3" database="itcast" />
```



### 2:rule.xml

```xml
<tableRule name="sharding-by-month">
	<rule>
		<columns>create_time</columns>
		<algorithm>partbymonth</algorithm>
	</rule>
</tableRule>

<function name="partbymonth" class="io.mycat.route.function.PartitionByMonth">
	<property name="dateFormat">yyyy-MM-dd</property>
	<property name="sBeginDate">2022-01-01</property>
	<property name="sEndDate">2022-03-31</property>
</function>
```

从开始时间开始，一个月为一个分片，到达结束时间之后，会重复开始分片插入
配置表的 dataNode 的分片，必须和分片规则数量一致，例如 2022-01-01 到 2022-12-31 ，共需要12个分片

![image-20221007103921057](.\assets\image-20221007103921057.png)



# 6:Mycat监控

## 6.1:原理

![image-20221007104035306](.\assets\image-20221007104035306.png)

在MyCat中，当执行一条SQL语句时，MyCat需要进行SQL解析、分片分析、路由分析、读写分离分析 等操作，最终经过一系列的分析决定将当前的SQL语句到底路由到那几个(或哪一个)节点数据库，数据库将数据执行完毕后，如果有返回的结果，则将结果返回给MyCat，最终还需要`在MyCat中进行结果合并、聚合处理、排序处理、分页处理等操作`，最终再将结果返回给客户端

而在MyCat的使用过程中，MyCat官方也提供了一个管理监控平台MyCat-Web（MyCat-eye）。 Mycat-web 是 Mycat 可视化运维的管理和监控平台，弥补了 Mycat 在监控上的空白。帮 Mycat 分担统计任务和配置管理任务。Mycat-web 引入了 ZooKeeper 作为配置中心，可以管理多个节 点。Mycat-web 主要管理和监控 Mycat 的流量、连接、活动线程和内存等，具备 IP 白名单、邮件告警等模块，还可以统计 SQL 并分析慢 SQL 和高频 SQL 等。为优化 SQL 提供依据



## 6.2命令行管理

Mycat默认开通2个端口，可以在server.xml中进行修改

- 8066 数据访问端口，即进行 DML 和 DDL 操作

- 9066 数据库管理端口，即 mycat 服务管理控制功能，管理mycat的整个集群状态



连接MyCat的管理控制台：

`mysql -h 192.168.200.210 -p 9066 -uroot -p123456`

![image-20221007104744589](.\assets\image-20221007104744589.png)

## 6.3:图形化管理

Mycat-web(Mycat-eye)是对mycat-server提供监控服务，功能不局限于对mycat-server使 用。他通过JDBC连接

对Mycat、Mysql监控，监控远程服务器(目前仅限于linux系统)的cpu、内存、网络、磁盘

Mycat-eye运行过程中需要依赖zookeeper，因此需要先安装zookeeper

### 1:安装Zookeeper

上传安装包 
zookeeper-3.4.6.tar.gz
	
解压
tar -zxvf zookeeper-3.4.6.tar.gz -C /usr/local/

​	

创建数据存放目录
cd /usr/local/zookeeper-3.4.6/
mkdir data

​	

修改配置文件名称并配置

cd config

mv zoo_sample.cfg zoo.cfg

​	

配置数据存放目录
dataDir=/usr/local/zookeeper-3.4.6/data
	
启动Zookeeper
bin/zkServer.sh start

bin/zkServer.sh status



### 2:安装Mycat-web

上传安装包 
Mycat-web.tar.gz
	
解压
tar -zxvf Mycat-web.tar.gz -C /usr/local/

​	

目录介绍
etc         ----> jetty配置文件
lib         ----> 依赖jar包
mycat-web   ----> mycat-web项目
readme.txt
start.jar   ----> 启动jar
start.sh    ----> linux启动脚本

​	

启动
sh start.sh
	
访问
http://192.168.200.210:8082/mycat



### 3:配置

 在Mycat监控界面配置服务地址，把自己的Mycat信息配置上去

![image-20221007105335840](.\assets\image-20221007105335840.png)



### 4:测试

配置好了之后，我们可以通过MyCat执行一系列的增删改查的测试，然后过一段时间之后，打开 mycat-eye的管理界面，查看mycat-eye监控到的数据信息



### 5:查看监控

1： 物理节点

![image-20221007105512861](.\assets\image-20221007105512861.png)

2：SQL统计

![image-20221007105530485](.\assets\image-20221007105530485.png)

3：SQL表分析

![image-20221007105549399](.\assets\image-20221007105549399.png)



4：SQL监控

![image-20221007105616100](.\assets\image-20221007105616100.png)

5：高频Sql

![image-20221007105650312](.\assets\image-20221007105650312.png)







# 7:Mycat读写分离

## 7.1:介绍

读写分离,简单地说是把对数据库的读和写操作分开,以对应不同的数据库服务器。主数据库提供写操作，从数据库提供读操作，这样能有效地减轻单台数据库的压力。 通过MyCat即可轻易实现上述功能，不仅可以支持MySQL，也可以支持Oracle和SQL Server。

![image-20221007105829572](.\assets\image-20221007105829572.png)

## 7.2:一主一从实现

### 1:原理

MySQL的主从复制，是基于二进制日志（binlog）实现的

![image-20221007105915592](.\assets\image-20221007105915592.png)

### 2:准备

![image-20221007105936838](.\assets\image-20221007105936838.png)



### 3:主从复制搭建

主从复制的搭建，可以参考前面的笔记



### 4:读写分离实现

MyCat控制后台数据库的读写分离和负载均衡由schema.xml文件datahost标签的balance属性控制

#### 1:schema.xml

```xml
<!-- 配置逻辑库 -->
<schema name="ITCAST_RW" checkSQLschema="true" sqlMaxLimit="100" dataNode="dn7">
</schema>

<dataNode name="dn7" dataHost="dhost7" database="itcast" />

<dataHost name="dhost7" maxCon="1000" minCon="10" balance="1" writeType="0"
dbType="mysql" dbDriver="jdbc" switchType="1" slaveThreshold="100">
	<heartbeat>select user()</heartbeat>
    <!--写操作对应的数据库-->
	<writeHost host="master1" url="jdbc:mysql://192.168.200.211:3306?useSSL=false&amp;serverTimezone=Asia/Shanghai&amp;characterEncoding=utf8"
user="root" password="1234" >
     
    <!--读操作对应的数据库-->   
	<readHost host="slave1" url="jdbc:mysql://192.168.200.212:3306?useSSL=false&amp;serverTimezone=Asia/Shanghai&amp;characterEncoding=utf8"
user="root" password="1234" />
        
	</writeHost>
</dataHost>
```

writeHost代表的是写操作对应的数据库，readHost代表的是读操作对应的数据库。 所以我们要想 实现读写分离，就得配置writeHost关联的是主库，readHost关联的是从库。 而仅仅配置好了writeHost以及readHost还不能完成读写分离，还需要配置一个非常重要的负责均衡 的参数 balance，取值有4种，具体含义如下

![image-20221007110417531](.\assets\image-20221007110417531.png)

一般设置为 1 或者 3



#### 2:server.xml

```xml
<user name="root" defaultAccount="true">
	<property name="password">123456</property>
	<property name="schemas">SHOPPING,ITCAST,ITCAST_RW</property>
</user>
```

测试中，我们可以发现当主节点Master宕机之后，业务系统就只能够读，而不能写入数据了

那如何解决这个问题呢？这个时候我们就得通过另外一种主从复制结构来解决了

也就是我们接下来的双主双从



## 7.3:双主双从实现

一个主机 Master1 用于处理所有写请求，它的从机 Slave1 和另一台主机 Master2 还有它的从 机 Slave2 负责所有读请求。当 Master1 主机宕机后，Master2 主机负责写请求，Master1 、 Master2 互为备机

![image-20221007110631870](.\assets\image-20221007110631870.png)

这里不实现了，理解思想即可
