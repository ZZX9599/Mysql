# Mysql原理



# 1:事务



## 1.2:概述

事务:是一组操作的集合，它是一个不可分割的工作单位，事务会把所有的操作作为一个整体一起向系 统提交或撤销操作请求，即这些操作要么同时成功，要么同时失败。 

就比如: 张三给李四转账1000块钱，张三银行账户的钱减少1000，而李四银行账户的钱要增加 1000。 这一组操作就必须在一个事务的范围内，要么都成功，要么都失败。



## 1.3:控制事务方式

### 1.3.1:方式一

查看/设置事务提交方式

```sql
-- 查看事务提交方式【mysql默认为1-->即自动提交】
SELECT @@AUTOCOMMIT;
-- 设置事务提交方式，1为自动提交，0为手动提交，该设置只对当前会话有效
SET @@AUTOCOMMIT = 0;
```

提交事务

```sql
COMMIT
```

回滚事务

```sql
ROLLBACK
```

注意：上述的这种方式，我们是修改了事务的自动提交行为, 把默认的自动提交修改为了手动提 交, 此时我们执行的DML语句都不会提交, 需要手动的执行commit进行提交。



### 1.3.2:方式二

开启事务

```sql
START TRANSACTION 或 BEGIN 
```

提交事务

```sql
COMMIT
```

回滚事务

```sql
ROLLBACK
```



## 1.4:事务特性

- 原子性(Atomicity)：事务是不可分割的最小操作单元，要么全部成功，要么全部失败
- 一致性(Consistency)：事务完成时，必须使所有数据都保持一致状态
- 隔离性(Isolation)：数据库系统提供的隔离机制，保证事务在不受外部并发操作影响的独立环境下运行
- 持久性(Durability)：事务一旦提交或回滚，它对数据库中的数据的改变就是永久的



## 1.5:并发事务

| 问题       | 描述                                                         |
| ---------- | ------------------------------------------------------------ |
| 脏读       | 一个事务读到另一个事务还没提交的数据                         |
| 不可重复读 | 一个事务先后读取同一条记录，但两次读取的数据不同             |
| 幻读       | 一个事务按照条件查询数据时，没有对应的数据行，但是再插入数据时，又发现这行数据已经存在 |



## 1.6:并发问题



### 1.6.1:脏读

![image-20220901164817873](.\assets\image-20220901164817873.png)



### 1.6.2:不可重复读

正常情况下，一个事务内，执行相同的select命令，查到的结果应该是一样的

![image-20220901165048581](.\assets\image-20220901165048581.png)



### 1.6.3:幻读

在一个事务内，正常情况是我之前查询没有数据，我在这个事务没执行插入前，也应该是没数据

![image-20220901165211229](.\assets\image-20220901165211229.png)





## 1.7:并发事务隔离级别

| 隔离级别                   | 脏读 | 不可重复读 | 幻读 |
| -------------------------- | ---- | ---------- | ---- |
| Read uncommitted           | √    | √          | √    |
| Read committed(Oracle默认) | ×    | √          | √    |
| Repeatable Read(Mysql默认) | ×    | ×          | √    |
| Serializable               | ×    | ×          | ×    |

- √表示在当前隔离级别下该问题会出现
- Serializable 性能最低；Read uncommitted 性能最高，数据安全性最差

查看事务隔离级别：
`SELECT @@TRANSACTION_ISOLATION;`



设置事务隔离级别：
`SET [ SESSION | GLOBAL ] TRANSACTION ISOLATION LEVEL {READ UNCOMMITTED | READ COMMITTED | REPEATABLE READ | SERIALIZABLE };`



SESSION 是会话级别，表示只针对当前会话有效，GLOBAL 表示对所有会话有效





# 2:存储引擎



## 2.1:MySQL体系结构

![image-20220901172629333](.\assets\image-20220901172629333.png)

![image-20220901172649763](.\assets\image-20220901172649763.png)

索引是在引擎层实现的，不同的引擎，索引结构是不同的



## 2.2:存储引擎介绍

大家可能没有听说过存储引擎，但是一定听过引擎这个词，引擎就是发动机，是一个机器的核心组件比如，对于舰载机、直升机、火箭来说，他们都有各自的引擎，是他们最为核心的组件。而我们在选择 引擎的时候，需要在合适的场景，选择合适的存储引擎

而对于存储引擎也是一样，他是mysql数据库的核心，我们也需要在合适的场景选择合适的存储引擎

- 存储引擎就是存储数据、建立索引、更新/查询数据等技术的实现方式 
- 存储引擎是基于表的，而不是基于库的，所以存储引擎也可被称为表类型
- 我们可以在创建表的时候，来指定选择的存储引擎，如果没有指定，将自动选择默认的存储引擎

没指定建表的存储引擎的时候，可以使用`show create table 表名`来查看

```sql
CREATE TABLE `student` (
  `id` int DEFAULT NULL,
  `num` int DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
```



- 查看当前数据库支持什么存储引擎

   

  引擎          是否支持  注释             是否支持  -->  事务    XA    保存点 

![image-20220901173520454](.\assets\image-20220901173520454.png)



- 建表时指定存储引擎

```sql
CREATE TABLE 表名(
字段1 字段1类型 [ COMMENT 字段1注释 ] ,
......
字段n 字段n类型 [COMMENT 字段n注释 ]
) ENGINE = INNODB [ COMMENT 表注释 ] ;
```



## 2.3:InnoDB

### 2.3.1:介绍

InnoDB是一种兼顾高可靠性和高性能的通用存储引擎，在 MySQL 5.5 之后

InnoDB是默认的 MySQL 存储引擎



### 2.3.1:特点

- DML操作遵循ACID模型，支持事务
- 行级锁，提高并发访问性能
- 支持外键FOREIGN KEY约束，保证数据的完整性和正确性
- 使用 InnoDB 的数据库在异常崩溃后，数据库重新启动的时候会保证数据库恢复到崩溃前的状态。这个恢复的过程依赖于 `redo log`
- InnoDB 支持 MVCC

事务，外键，行级锁



### 2.3.3:文件

xxx.ibd：xxx代表的是表名，innoDB引擎的每张表都会对应这样一个表空间文件

学生表 table---->对应一个student.idb文件

存储该表的表结构，数据，索引

参数：innodb_file_per_table

如果该参数开启，代表对于InnoDB引擎的表，每一张表都对应一个ibd文件,mysql8默认打开



查看nacos数据库里的表：

![image-20220901175233617](.\assets\image-20220901175233617.png)



查看mysql安装目录的data文件夹下

![image-20220901175328471](.\assets\image-20220901175328471.png)



### 2.3.4:逻辑存储结构

![image-20220901175441089](.\assets\image-20220901175441089.png)

- 表空间 : InnoDB存储引擎逻辑结构的最高层，ibd文件其实就是表空间文件，在表空间中可以 包含多个Segment段。 
- 段 : 表空间是由各个段组成的， 常见的段有数据段、索引段、回滚段等。InnoDB中对于段的管 理，都是引擎自身完成，不需要人为对其控制，一个段中包含多个区。 
- 区 : 区是表空间的单元结构，每个区的大小为1M。 默认情况下， InnoDB存储引擎页大小为 16K， 即一个区中一共有64个连续的页。 
- 页 : 页是组成区的最小单元，页也是InnoDB 存储引擎磁盘管理的最小单元，每个页的大小默 认为 16KB。为了保证页的连续性，InnoDB 存储引擎每次从磁盘申请 4-5 个区。 
- 行 : InnoDB 存储引擎是面向行的，也就是说数据是按行进行存放的，在每一行中除了定义表时 所指定的字段以外，还包含两个隐藏字段(后面会详细介绍)。



## 2.4:MyISAM



### 2.4.1:介绍

MyISAM是MySQL早期的默认存储引擎。



### 2.4.2:特点

- 不支持事务
- 不支持外键 支持表锁
- 不支持行锁 访问速度快
- 不支持数据库异常崩溃后的安全恢复
- 不支持 MVCC



### 2.4.3:文件

- xxx.sdi：存储表结构信息 

- xxx.MYD: 存储数据 

- xxx.MYI: 存储索引



## 2.5:Memory



### 2.5.1:介绍 

Memory引擎的表数据时存储在内存中的，由于受到硬件问题、或断电问题的影响，只能将这些表作为 临时表或缓存使用。



### 2.5.2:特点

内存存放 hash索引（默认）

MyISAM 只有表级锁，不支持行级锁

不支持事务

而且最大的缺陷就是崩溃后无法安全恢复



### 2.5.3:文件 

xxx.sdi：存储表结构信息



## 2.6:区别和特点

![image-20220901180733343](.\assets\image-20220901180733343.png)



![image-20220901180758616](.\assets\image-20220901180758616.png)

**回答上面主要的区别即可**



```apl
对比一下行级锁和表级锁
MyISAM 只有表级锁，不支持行级锁
而InnoDB 支持行级锁(row-level locking)和表级锁,默认为行级锁。

也就说，MyISAM 一锁就是锁住了整张表，这在并发写的情况下是多么滴憨憨啊！
这也是为什么 InnoDB 在并发写的时候，性能更牛皮了！

MyISAM 不支持，而 InnoDB 支持。
外键对于维护数据一致性非常有帮助，但是对性能有一定的损耗。因此，通常情况下，我们是不建议在实际生产项目中使用外键的，在业务代码中进行约束即可
阿里的《Java 开发手册》也是明确规定禁止使用外键的。
```



## 2.7:存储引擎选择

在选择存储引擎时，应该根据应用系统的特点选择合适的存储引擎。对于复杂的应用系统，还可以根据 实际情况选择多种存储引擎进行组合。 

- InnoDB：是Mysql的默认存储引擎，支持事务、外键。如果应用对事务的完整性有比较高的要 求，在并发条件下要求数据的一致性，数据操作除了插入和查询之外，还包含很多的更新、删除操 作，那么InnoDB存储引擎是比较合适的选择。 
- MyISAM：如果应用是以读操作和插入操作为主，只有很少的更新和删除操作，并且对事务的完 整性、并发性要求不是很高，那么选择这个存储引擎是非常合适的。 
- MEMORY：将所有数据保存在内存中，访问速度快，通常用于临时表及缓存。MEMORY的缺陷就是 对表的大小有限制，太大的表无法缓存在内存中，而且无法保障数据的安全性。



在实际开发的时候，数据库基本上都是InnoDB,使用MyISAM的场景基本都是被MongDB替代，使用MEMORY的场景基本都是被Redis取代



# 3:索引



## 3.1:索引概述



### 3.1.1:介绍

索引（index）是帮助MySQL高效获取数据的数据结构(有序)。在数据之外，数据库系统还维护着满足特定查找算法的数据结构，这些数据结构以某种方式引用（指向）数据， 这样就可以在这些数据结构上实现高级查找算法，这种数据结构就是索引。

本质：索引就是一种数据结构

作用：加速查找数据



### 3.1.2:无索引查找

演示

![image-20220901232605072](.\assets\image-20220901232605072.png)



假如我们要执行的SQL语句为 ： select * from user where age = 45



![image-20220901232628117](.\assets\image-20220901232628117.png)

在无索引情况下，就需要从第一行开始扫描，直到最后一行，我们称之为全表扫描，性能很低



### 3.1.3:有索引演示

如果我们针对于这张表建立了索引，假设索引结构就是二叉树，那么也就意味着，会对age这个字段建立一个二叉树的索引结构

![image-20220901232826220](.\assets\image-20220901232826220.png)



假如我们要执行的SQL语句为 ： select * from user where age = 45

此时我们在进行查询时，只需要扫描三次就可以找到数据了，极大的提高的查询的效率

先查根节点36不符合-->45比36大-->右边-->48-->45比48小-->左边-->45【找到了】

这里我们只是假设索引的结构是二叉树，介绍一下索引的大概原理

只是一个示意图，并不是索引的真实结构



![image-20220901233255853](.\assets\image-20220901233255853.png)

因为我们在insert update delete的时候，也需要对数据的修改进行索引数据结构的维护



## 3.2:索引结构



### 3.2.1:索引结构分类

索引是在引擎层实现的，不同的引擎，索引结构是不同的，主要有以下几种

![image-20220901233551687](.\assets\image-20220901233551687.png)



### 3.2.2:索引支持情况

上述是MySQL中所支持的所有的索引结构，以下是不同的存储引擎对于索引结构的支持情况

![image-20220902001802962](.\assets\image-20220902001802962.png)

我们平常所说的索引，如果没有特别指明，都是指Innodb引擎下的B+树结构组织的索引



### 3.2.3:二叉树

一个节点下面最多包含两个子节点



### 3.2.4:二叉排序树

- 左子树所有结点的值均小于根结点的值。
- 右子树所有结点的值均大于根结点的值。

![image-20220902091016332](.\assets\image-20220902091016332.png)



存在的问题：

如果主键是顺序插入的，则会形成一个单向链表，结构如下

![image-20220902091042068](.\assets\image-20220902091042068.png)

查找数字 17 就相当于遍历整个链表

所以，如果选择二叉树作为索引结构，会存在以下缺点： 

- 顺序插入时，会形成一个链表，查询性能大大降低。 
- 由于最多两个节点，大数据量情况下，层级较深，检索速度慢。

我们可以选择红黑树，红黑树是一颗自平衡二叉树，那这样即使是顺序插入数据，最终形成的数据结构也是一颗平衡的二叉树



### 3.2.5:红黑树

红黑树除了具有二叉排序树【二叉查找树】的特点外，还具有

- 结点是红色或黑色。

- 根结点是黑色。

- 每个叶子结点都是黑色的空结点（NIL结点）。

- 每个红色结点的两个子结点都是黑色(从每个叶子到根的所有路径上不能有两个连续的红色结点)

- 从任一结点到其每个叶子的所有路径都包含相同数目的黑色结点。

![image-20220902091617704](.\assets\image-20220902091617704.png)



缺点：大数据量情况下，层级较深，检索速度慢【二叉树都存在的问题】

所以，在MySQL的索引结构中，并没有选择二叉树或者红黑树，而选择的是B+Tree，那么什么是 B+Tree呢？在详解B+Tree之前，先来介绍一个B-Tree。



### 3.2.6:B-Tree

B-Tree，又叫做B树，是一种多叉路衡查找树，相对于二叉树，B树每个节点可以有多个分支，即多叉。以一颗最大度数(max-degree)为5(5阶)的b-tree为例，那这个B树每个节点最多存储4个key，5 个指针：

最大度数:每个节点的最多子节点个数

![image-20220902092937797](.\assets\image-20220902092937797.png)

可视化网站：https://www.cs.usfca.edu/~galles/visualization/BTree.html



构建过程：度为5



先加入四个元素

![image-20220902093824248](.\assets\image-20220902093824248.png)

再加入一个15，成为 12,15,35,45,98

中间元素向上分裂，形成两侧

![image-20220902093910140](.\assets\image-20220902093910140.png)



再加入100，比35大，走右边

![image-20220902094020984](.\assets\image-20220902094020984.png)



再加入199，比35大，走右边

![image-20220902094049961](.\assets\image-20220902094049961.png)



再加入88，比35大，走右边，但是超过了五阶，中间元素98向上分裂

![image-20220902094154048](.\assets\image-20220902094154048.png)



再插入23，26

![image-20220902094326442](.\assets\image-20220902094326442.png)



再插入32

![image-20220902094344002](.\assets\image-20220902094344002.png)



特点： 

- 5阶的B树，每一个节点最多存储4个key，对应5个指针。 
- 一旦节点存储的key数量到达5，就会裂变，中间元素向上分裂。 
- 在B树中，非叶子节点和叶子节点都会存放数据【注意和B+树区别】



### 3.2.7:B+树

B+Tree是B-Tree的变种，我们以一颗最大度数（max-degree）为4（4阶）的b+tree为例，来看一 下其结构示意图

![image-20220902094704107](.\assets\image-20220902094704107.png)

**可视化：https://www.cs.usfca.edu/~galles/visualization/BPlusTree.html**



最终我们看到，B+Tree 与 B-Tree相比，主要有以下三点区别： 

- 所有的数据都会出现在叶子节点。 
- 叶子节点形成一个单向链表。 
- 非叶子节点仅仅起到索引数据作用，具体的数据都是在叶子节点存放的。



演示：

先添加四个元素：

![image-20220902094811235](.\assets\image-20220902094811235.png)



再添加一个 176，则234是中间元素，向上分裂的同时，会把自己添加到叶子节点并形成单向链表

![image-20220902095107188](.\assets\image-20220902095107188.png)



再添加一个元素654

![image-20220902095148274](.\assets\image-20220902095148274.png)



再添加一个元素666

![image-20220902095218910](.\assets\image-20220902095218910.png)

上述我们所看到的结构是标准的B+Tree的数据结构，接下来，我们再来看看MySQL中优化之后的 B+Tree



### 3.2.8:Mysql优化的B+树

MySQL索引数据结构对经典的B+Tree进行了优化。在原B+Tree的基础上，增加一个指向相邻叶子节点 的链表指针，就形成了带有顺序指针的B+Tree，提高区间访问的性能，利于排序

![image-20220902095531710](.\assets\image-20220902095531710.png)

上面的非叶子节点只是起到索引的作用，叶子节点才是真实存储数据的

页是组成区的最小单元，页也是InnoDB 存储引擎磁盘管理的最小单元，每个页的大小默认为 16KB



### 3.2.9:Hash

MySQL中除了支持B+Tree索引，还支持一种索引类型---Hash索引。

结构：哈希索引就是采用一定的hash算法，将键值换算成新的hash值，映射到对应的槽位上，然后存储在 hash表中。

![image-20220902095808802](.\assets\image-20220902095808802.png)

![image-20220902095827603](.\assets\image-20220902095827603.png)

特点 

- Hash索引只能用于对等比较(=，in)，不支持范围查询（between，>，< ，...） 
- 无法利用索引完成排序操作 
- 查询效率高(如果不存在hash冲突的情况)只需要一次检索就可以了，效率通常要高于B+tree索引



存储引擎支持

在MySQL中，支持hash索引的是Memory存储引擎。 而InnoDB中具有自适应hash功能，hash索引是 InnoDB存储引擎根据B+Tree索引在指定条件下自动构建的



### 3.2.10:思考题

思考题:为什么InnoDB存储引擎选择使用B+tree索引结构

本道题就是为了对比以上常见的数据结构的区别

- 相对于二叉树，层级更少，搜索效率高； 
- 对于B-tree，无论是叶子节点还是非叶子节点，都会保存数据，这样导致一页中存储 的键值减少，指针跟着减少，要同样保存大量数据，只能增加树的高度，导致性能降低； 
- 相对Hash索引，B+tree支持范围匹配及排序操作；
- 还可以说出mysql的B+树进行了优化，改为了双向的链表指针



## 3.3:索引分类



### 3.3.1:索引常见分类

在MySQL数据库，将索引的具体类型主要分为以下几类：主键索引、唯一索引、常规索引、全文索引

![image-20220902100509939](.\assets\image-20220902100509939.png)

事实上，加入主键，唯一约束等信息之后，就会默认创建索引了



### 3.3.2:聚集索引&二级索引

在InnoDB存储引擎中，根据索引的存储形式，又可以分为以下两种：

![image-20220902100637789](.\assets\image-20220902100637789.png)

没有指定索引的话，存在聚集索引自动选取规则：

- 如果存在主键，主键索引就是聚集索引
- 如果不存在主键，将使用第一个唯一（UNIQUE）索引作为聚集索引
- 如果表没有主键，或没有合适的唯一索引，则InnoDB会自动生成一个rowid作为隐藏的聚集索 引。

也就是无论如何，聚集索引都会存在



聚集索引结构

![image-20220902101052954](.\assets\image-20220902101052954.png)

聚集索引的叶子节点的数据就是这一行的数据

比如叶子节点5下面悬挂的row就是  id:5,name:kit,gender:男



由于聚集索引只存在一个，所以name的索引就不是聚集索引，而是二级索引

![image-20220902101228654](.\assets\image-20220902101228654.png)

二级索引下面的数据存放的是该字段值对应的主键值

例如Arm 下面悬挂的数据是 id=10



- 聚集索引的叶子节点下挂的是这一行的数据 
- 二级索引的叶子节点下挂的是该字段值对应的主键值



### 3.3.3:分析执行SQL语句

接下来，我们来分析一下，当我们执行如下的SQL语句时，具体的查找过程是什么样子的

![image-20220902101449038](.\assets\image-20220902101449038.png)



```apl
where name='Arm'
不会走聚集索引，走了name的二级索引
二级索引叶子节点悬挂的是当前数据的id
找到对应的id
由于我们是select * 就会进行回表查询
根据id查询数据，走聚集索引
聚集索引下面的是这一行的数据
就查到了对应的行数据
```



回表查询： 这种先到二级索引中查找数据，找到主键值，然后再到聚集索引中根据主键值，获取数据的方式，就称之为回表查询。



### 3.3.4:思考题

以下两条SQL语句，那个执行效率高? 为什么? 

- select * from user where id = 10 
- select * from user where name = 'Arm'
- 备注: id为主键，name字段创建的有索引



```apl
因为id为主键，第一条根据id查询，走聚集索引，聚集索引的数据结构的叶子节点悬挂的就是当前行数据
第二条，name有索引，属于二级索引，叶子节点下面悬挂的是当前的id
会进行回表查询，效率更低
```



InnoDB主键索引的B+tree高度为多高呢?

![image-20220902102536581](.\assets\image-20220902102536581.png)

每个节点都是一个页

每个页的大小默认为 16 KB

所以每个页能存储的key和指针是有限的



假设: 

一行数据大小为1k，一页中可以存储16行这样的数据。InnoDB的指针占用6个字节的空间【固定】

主键为int，四个字节，即使为bigint，占用字节数为8



如果高度为2

n * 8 + (n + 1) * 6 = 16*1024 , 算出n约为 1170

```apl
n*8   ====> 主键为bigint的占用字节
(n+1)*6  ===> 指针所占用的字节
16*1024 ===> 每页16k固定的，也就是16*1024字节
算出n约为 1170
1171* 16 = 18736
```



如果高度为3

1171 * 1171 * 16 = 21939856 

也就是说，如果树的高度为3，则可以存储 2200w 左右的记录。



## 3.4:索引语法

数据准备

```sql
create table tb_user(
	id int primary key auto_increment comment '主键',
	name varchar(50) not null comment '用户名',
	phone varchar(11) not null comment '手机号',
	email varchar(100) comment '邮箱',
	profession varchar(11) comment '专业',
	age tinyint unsigned comment '年龄',
	gender char(1) comment '性别 , 1: 男, 2: 女',
	status char(1) comment '状态',
	createtime datetime comment '创建时间'
) comment '系统用户表';


INSERT INTO tb_user (name, phone, email, profession, age, gender, status, createtime) VALUES ('吕布', '17799990000', 'lvbu666@163.com', '软件工程', 23, '1', '6', '2001-02-02 00:00:00');
INSERT INTO tb_user (name, phone, email, profession, age, gender, status, createtime) VALUES ('曹操', '17799990001', 'caocao666@qq.com', '通讯工程', 33, '1', '0', '2001-03-05 00:00:00');
INSERT INTO tb_user (name, phone, email, profession, age, gender, status, createtime) VALUES ('赵云', '17799990002', '17799990@139.com', '英语', 34, '1', '2', '2002-03-02 00:00:00');
INSERT INTO tb_user (name, phone, email, profession, age, gender, status, createtime) VALUES ('孙悟空', '17799990003', '17799990@sina.com', '工程造价', 54, '1', '0', '2001-07-02 00:00:00');
INSERT INTO tb_user (name, phone, email, profession, age, gender, status, createtime) VALUES ('花木兰', '17799990004', '19980729@sina.com', '软件工程', 23, '2', '1', '2001-04-22 00:00:00');
INSERT INTO tb_user (name, phone, email, profession, age, gender, status, createtime) VALUES ('大乔', '17799990005', 'daqiao666@sina.com', '舞蹈', 22, '2', '0', '2001-02-07 00:00:00');
INSERT INTO tb_user (name, phone, email, profession, age, gender, status, createtime) VALUES ('露娜', '17799990006', 'luna_love@sina.com', '应用数学', 24, '2', '0', '2001-02-08 00:00:00');
INSERT INTO tb_user (name, phone, email, profession, age, gender, status, createtime) VALUES ('程咬金', '17799990007', 'chengyaojin@163.com', '化工', 38, '1', '5', '2001-05-23 00:00:00');
INSERT INTO tb_user (name, phone, email, profession, age, gender, status, createtime) VALUES ('项羽', '17799990008', 'xiaoyu666@qq.com', '金属材料', 43, '1', '0', '2001-09-18 00:00:00');
INSERT INTO tb_user (name, phone, email, profession, age, gender, status, createtime) VALUES ('白起', '17799990009', 'baiqi666@sina.com', '机械工程及其自动化', 27, '1', '2', '2001-08-16 00:00:00');
INSERT INTO tb_user (name, phone, email, profession, age, gender, status, createtime) VALUES ('韩信', '17799990010', 'hanxin520@163.com', '无机非金属材料工程', 27, '1', '0', '2001-06-12 00:00:00');
INSERT INTO tb_user (name, phone, email, profession, age, gender, status, createtime) VALUES ('荆轲', '17799990011', 'jingke123@163.com', '会计', 29, '1', '0', '2001-05-11 00:00:00');
INSERT INTO tb_user (name, phone, email, profession, age, gender, status, createtime) VALUES ('兰陵王', '17799990012', 'lanlinwang666@126.com', '工程造价', 44, '1', '1', '2001-04-09 00:00:00');
INSERT INTO tb_user (name, phone, email, profession, age, gender, status, createtime) VALUES ('狂铁', '17799990013', 'kuangtie@sina.com', '应用数学', 43, '1', '2', '2001-04-10 00:00:00');
INSERT INTO tb_user (name, phone, email, profession, age, gender, status, createtime) VALUES ('貂蝉', '17799990014', '84958948374@qq.com', '软件工程', 40, '2', '3', '2001-02-12 00:00:00');
INSERT INTO tb_user (name, phone, email, profession, age, gender, status, createtime) VALUES ('妲己', '17799990015', '2783238293@qq.com', '软件工程', 31, '2', '0', '2001-01-30 00:00:00');
INSERT INTO tb_user (name, phone, email, profession, age, gender, status, createtime) VALUES ('芈月', '17799990016', 'xiaomin2001@sina.com', '工业经济', 35, '2', '0', '2000-05-03 00:00:00');
INSERT INTO tb_user (name, phone, email, profession, age, gender, status, createtime) VALUES ('嬴政', '17799990017', '8839434342@qq.com', '化工', 38, '1', '1', '2001-08-08 00:00:00');
INSERT INTO tb_user (name, phone, email, profession, age, gender, status, createtime) VALUES ('狄仁杰', '17799990018', 'jujiamlm8166@163.com', '国际贸易', 30, '1', '0', '2007-03-12 00:00:00');
INSERT INTO tb_user (name, phone, email, profession, age, gender, status, createtime) VALUES ('安琪拉', '17799990019', 'jdodm1h@126.com', '城市规划', 51, '2', '0', '2001-08-15 00:00:00');
INSERT INTO tb_user (name, phone, email, profession, age, gender, status, createtime) VALUES ('典韦', '17799990020', 'ycaunanjian@163.com', '城市规划', 52, '1', '2', '2000-04-12 00:00:00');
INSERT INTO tb_user (name, phone, email, profession, age, gender, status, createtime) VALUES ('廉颇', '17799990021', 'lianpo321@126.com', '土木工程', 19, '1', '3', '2002-07-18 00:00:00');
INSERT INTO tb_user (name, phone, email, profession, age, gender, status, createtime) VALUES ('后羿', '17799990022', 'altycj2000@139.com', '城市园林', 20, '1', '0', '2002-03-10 00:00:00');
INSERT INTO tb_user (name, phone, email, profession, age, gender, status, createtime) VALUES ('姜子牙', '17799990023', '37483844@qq.com', '工程造价', 29, '1', '4', '2003-05-26 00:00:00');
```



### 3.4.1:创建索引

```sql
CREATE [ UNIQUE | FULLTEXT ] INDEX index_name ON table_name (index_col_name,... )
```

**创建唯一索引，全文索引，不写就是常规索引**



### 3.4.2:查看索引

```sql
SHOW INDEX FROM table_name
SHOW INDEX FROM table_name\G  转换为列
```



### 3.4.3:删除索引

```sql
DROP INDEX index_name ON table_name 
```



### 3.4.4:语法练习



先查看索引，发现只有主键索引

![image-20220902104533868](.\assets\image-20220902104533868.png)



需求：

- name字段为姓名字段，该字段的值可能会重复，为该字段创建索引。

```sql
create index idx_user_name on tb_user(name)  idx_表_字段
```

再次查看索引信息

![image-20220902104757721](.\assets\image-20220902104757721.png)

- phone手机号字段的值，是非空，且唯一的，为该字段创建唯一索引。

```sql
create unique index idx_user_phone on tb_user(phone);
```

![image-20220902104937153](.\assets\image-20220902104937153.png)

- 为profession、age、status创建联合索引。

```sql
create index idx_user_pro_age_sta on tb_user(profession,age,status);
```

![image-20220902105120972](.\assets\image-20220902105120972.png)

- 为email建立合适的索引来提升查询效率。

```sql
create index idx_user_email on tb_user(email);
```

![image-20220902105216200](.\assets\image-20220902105216200.png)



## 3.5:性能分析



### 3.5.1:SQL执行频率

MySQL 客户端连接成功后，通过 show [session|global] status 命令可以提供服务器状态信 息。通过如下指令，可以查看当前数据库的INSERT、UPDATE、DELETE、SELECT的访问频次：

知道了执行频率，我们才知道怎么在线上对哪些sql来进行优化

```sql
-- session 是查看当前会话 ;
-- global 是查询全局数据 ;
SHOW GLOBAL STATUS LIKE 'Com_______';  一个_代表匹配一个字符
```



![](.\assets\image-20220902105546373.png)



我们执行一次select * from tb_user

再执行SHOW GLOBAL STATUS LIKE 'Com_______'

发现数量增加了1



那么通过查询SQL的执行频次，我们就能够知道当前数据库到底是增删改为主，还是查询为主

那假 如说是以查询为主，我们又该如何定位针对于那些查询语句进行优化呢？ 

我们可以借助于慢查询日志。



### 3.5.2:慢查询日志

```sql
先导入一千万的数据到数据库
一次性需要插入大批量数据，使用insert语句插入性能较低
此时可以使用MysQL数据库提供的load指令进行插入。
准备1000W的数据在tb_sku.sql下

连接数据库的时候：mysql --local-infile -u root -p

load data local infile '/root/sql/tb_sku.sql' into table `tb_sku` fields terminated by ',' lines terminated by '\n';
```



慢查询日志记录了所有执行时间超过指定参数（long_query_time，单位：秒，默认10秒）的所有 SQL语句的日志。 

MySQL的慢查询日志默认没有开启，我们可以查看一下系统变量 slow_query_log。

```sql
#查看是否开启慢查询日志，OFF代表每页，默认为OFF
show variables like 'slow_query_log'  
```



如果要开启慢查询日志，需要在Linux的MySQL的配置文件（/etc/my.cnf）中配置如下信息：

```apl
在[Mysqld]下加入
# 开启MySQL慢日志查询开关
slow_query_log=1
# 设置慢日志的时间为2秒，SQL语句执行时间超过2秒，就会视为慢查询，记录慢查询日志
long_query_time=2
# 慢查询日志文件位置
slow_query_log_file = /var/log/mysqld-slow.log
```



在/var/log/下使用  tail -f mysqld-slow.log可以查看文件的追加情况



执行 select count(*) from tb_sku 明显大于2S

这边输出了慢查询日志 用户，ip，花费时间，执行的语句等

![image-20220902154344483](.\assets\image-20220902154344483.png)



最终我们发现，在慢查询日志中，只会记录执行时间超多我们预设时间（2s）的SQL，执行较快的SQL 是不会记录的

那这样，通过慢查询日志，就可以定位出执行效率比较低的SQL，从而有针对性的进行优化。



### 3.5.3:profile详情

show profiles 能够在做SQL优化时帮助我们了解时间都耗费到哪里去了

通过have_profiling 参数，能够看到当前MySQL是否支持profile操作

```sql
SELECT @@have_profiling
```

返回Yes即代表支持，支持但不是代表开启了

```sql
SELECT @@profiling
```

返回0，代表没有开启

可以通过set语句在 session/global级别开启profiling

```sql
SET profiling = 1
```

开关已经打开了，接下来，我们所执行的SQL语句，都会被MySQL记录，并记录执行时间消耗到哪儿去 了。 我们直接执行如下的SQL语句

```sql
select * from tb_user;
select * from tb_user where id = 1;
select * from tb_user where name = '白起';
select count(*) from tb_sku;
```



执行后执行 show profiles

![image-20220902155017002](.\assets\image-20220902155017002.png)



假如我想知道select count(*) from tb_sku耗费时间到哪里去了

我们就可以执行一个命令查看当前SQL时间都耗费到哪里去了

-- 查看指定query_id的SQL语句各个阶段的耗时情况
show profile for query query_id;



以上的query_id也就是5

```sql
show profile for query 5;
```

![image-20220902155347911](.\assets\image-20220902155347911.png)

我们发现耗费时间主要是在executing【执行】



-- 查看指定query_id的SQL语句CPU的使用情况 

show profile cpu for query query_id;

```sql
show profile cpu for query 5;
```

![image-20220902155529427](.\assets\image-20220902155529427.png)



### 3.5.4:explain执行计划

上面的时间只是初略判断，对于开发人员来讲，了解即可，这个执行计划对开发人员是比较重要的

EXPLAIN 或者 DESC命令获取 MySQL 如何执行 SELECT 语句的信息，包括在 SELECT 语句执行 过程中表如何连接和连接的顺序

```sql
-- 直接在select语句之前加上关键字 explain / desc【描述】
EXPLAIN SELECT 字段列表 FROM 表名 WHERE 条件 ;
```

例如 explain select * from tb_user where id=1

![image-20220902155808862](.\assets\image-20220902155808862.png)



Explain 执行计划中各个字段的含义：



#### 3.5.4.1:id

select 查询的序列号，表示查询中执行 select 子句或者操作表的顺序（id相同，执行顺序从上到下；id不同，值越大越先执行）多表查询的时候才有用

这里使用一个多表联查

![image-20220902160817232](.\assets\image-20220902160817232.png)

id相同的话，表结构的执行顺序就是从上到下

嵌套子查询的时候，最内层的表的id最大，最内层最先执行



#### 3.5.4.2:select_type

表示 SELECT 的类型，常见取值有 SIMPLE（简单表，即不适用表连接或者子查询）、PRIMARY（主查询，即外层的查询）、UNION（UNION中的第二个或者后面的查询语句）、SUBQUERY（SELECT/WHERE之后包含了子查询）等



#### 3.5.4.3:type

表示连接类型，性能由好到差的连接类型为 NULL、system、const、eq_ref、ref、range、index、all，不可能优化到null，除非是 类似 select '10' 直接输出的

访问系统表，可能会出现system

根据主键，唯一索引的，就会出现const

使用非唯一性的索引，会出现 ref 

出现all，代表全表扫面

出现index，代表使用了索引，但是还是做了全表扫描



#### 3.5.4.4:possible_key

可能应用在这张表上的索引，一个或多个



#### 3.5.4.5:key

实际使用的索引，如果为 NULL，则没有使用索引



#### 3.5.4.6:key_len

表示索引中使用的字节数，该值为索引字段最大可能长度，并非实际使用长度，在不损失精确性的前提下，长度越短越好



#### 3.5.4.7:rows

MySQL认为必须要执行的行数，在InnoDB引擎的表中，是一个估计值，可能并不总是准确的



#### 3.5.4.8:filtered

表示返回结果的行数占需读取行数的百分比，filtered的值越大越好



#### 3.5.4.9:我们最关注的是

- type
- possible_key
- key
- key_len





## 3.6:索引使用

### 3.6.1:验证索引效率

我们还是使用之前准备的一张表 tb_sku , 在这张表中准备了1000w 的记录

![image-20220902163421389](.\assets\image-20220902163421389.png)



我们在建表的时候添加了主键，就默认添加了主键索引，根据id主键查询，注意观察执行时间

![image-20220902163521135](.\assets\image-20220902163521135.png)



根据sn查询，我们没设置索引，也就是不走索引，查看时间

![image-20220902163647065](.\assets\image-20220902163647065.png)



验证：我们给sn创建一个索引

create index idx_sku_sn on tb_sku(sn)

注意：由于我们的数据量太大，创建索引的时间也会很长，原因是因为要对数据创建B+树的数据结构来维护数据，所以耗时也是较长的



这个时候，我们在执行一次刚刚的根据sn查询的sql语句

![image-20220902164216593](.\assets\image-20220902164216593.png)

发现时间大大的缩短，即验证了索引对于效率的提升



查看执行计划，有没有使用索引

![image-20220902164337468](.\assets\image-20220902164337468.png)





### 3.6.2:最左前缀法则

如果索引了多列（联合索引），要遵守最左前缀法则。最左前缀法则指的是查询从索引的最左列开始，并且不跳过索引中的列。如果跳跃某一列，索引将会部分失效(后面的字段索引失效)。



添加联合索引

![image-20220902164556641](.\assets\image-20220902164556641.png)



意思是：要使用联合索引的话，必须保证最左边存在【profession】

中间跳过【age】的话，则后面的索引【status】都会失效



```sql
explain select * from tb_user where profession = '软件工程' and age = 31 and status
= '0';
```



分析：是从最左侧profession开始，没跳过中间任何的联合索引字段，不存在索引失效



执行计划使用的索引信息

![image-20220902164957283](.\assets\image-20220902164957283.png)





```sql
explain select * from tb_user where profession = '软件工程' and age = 31;
```



分析：是从最左侧profession开始，没跳过中间任何的联合索引字段，不存在索引失效



执行计划使用的索引信息

![image-20220902165058647](.\assets\image-20220902165058647.png)

注意到key_len少了4，证明status的索引字段长度为4



```sql
explain select * from tb_user where profession = '软件工程';
```



分析：是从最左侧profession开始，没跳过中间任何的联合索引字段，不存在索引失效



执行计划使用的索引信息

![image-20220902165336358](.\assets\image-20220902165336358.png)

注意到key_len少了2，证明age的索引字段长度为2



得到索引长度  

- profession=36 
- age=2 
- status=4



```sql
explain select * from tb_user where age = 31 and status = '0';
explain select * from tb_user where status = '0';
```

分析：不是从最左侧profession开始，索引失效



执行计划使用的索引信息

![image-20220902165522253](.\assets\image-20220902165522253.png)





```sql
explain select * from tb_user where profession = '软件工程' and status = '0';
```

分析：从最左侧profession开始，但是跳过了age，则从age后面的索引都失效，key_len=36



![image-20220902165726777](.\assets\image-20220902165726777.png)



思考:

当执行SQL语句: explain select * from tb_user where age = 31 and status = '0' and profession = '软件工程'； 时，是否满足最左前缀法则，走不走 上述的联合索引，索引长度？



```apl
会走索引，实际上他并不是要求你的profession必须放在第一个，他的意思是存在即可
```



### 3.6.3:范围查询

联合索引中，出现范围查询(>,<)，范围查询右侧的列索引失效

```sql
explain select * from tb_user where profession = '软件工程' and 
age > 30 and status = '0';
```



![image-20220902170200666](.\assets\image-20220902170200666.png)

key_len的长度为38，说明只有profession和age走了索引，status索引失效



解决方式：

在业务允许的情况下尽量使用>=和<=进行范围查询

```sql
explain select * from tb_user where profession = '软件工程' and 
age >= 30 and status = '0';
```

![image-20220902170403226](.\assets\image-20220902170403226.png)



### 3.6.4:索引失效情况



#### 3.6.4.1:索引列运算

不要在索引列上进行运算操作， 索引将失效。

我们的表tb_user存在一个索引 idx_user_phone

![image-20220902170642422](.\assets\image-20220902170642422.png)

当根据phone字段进行函数运算操作之后，索引失效。

```sql
#查询电话号码结尾两位数为15的信息
explain select * from tb_user where substring(phone,10,2) = '15';
```



查看执行计划，发现不走索引

![image-20220902170844253](.\assets\image-20220902170844253.png)







#### 3.6.4.2:字符串不加引号

接下来，我们通过两组示例，来看看对于字符串类型的字段，加单引号与不加单引号的区别



phone字段为varchar类型

```sql
explain select * from tb_user where phone = '17799990015';
explain select * from tb_user where phone = 17799990015;
```



字符串类型加上了引号

![image-20220902171125416](.\assets\image-20220902171125416.png)



不加引号

![image-20220902171200930](.\assets\image-20220902171200930.png)



经过上面两组示例，我们会明显的发现，如果字符串不加单引号，对于查询结果，没什么影响，但是数据库存在隐式类型转换，索引将失效



#### 3.6.4.3:模糊查询

如果仅仅是尾部模糊匹配，索引不会失效。如果是头部模糊匹配，索引失效。

接下来，我们来看一下这三条SQL语句的执行效果，查看一下其执行计划： 

由于下面查询语句中，都是根据profession字段查询，符合最左前缀法则，联合索引是可以生效的

我们主要看一下，模糊查询时，%加在关键字之前，和加在关键字之后的影响

```sql
explain select * from tb_user where profession like '软件%';
explain select * from tb_user where profession like '%工程';
explain select * from tb_user where profession like '%工%';
```



第一条走了索引

![image-20220902171805041](.\assets\image-20220902171805041.png)



第二条不走

![image-20220902171859268](.\assets\image-20220902171859268.png)





第三条不走

![image-20220902171909756](.\assets\image-20220902171909756.png)



经过上述的测试，我们发现，在like模糊查询中，在关键字后面加%，索引可以生效

而如果在关键字 前面加了%，索引将会失效



#### 3.6.4.4:or连接条件

用or分割开的条件， 如果or前的条件中的列有索引，而后面的列中没有索引，或者or前的没有，or后的有，那么涉及的索引都不会被用到，只有两边都有的，才走索引

```sql
#id有索引，age只存在联合索引，没使用profession，不符合最左前缀，所以age相当于没有索引
explain select * from tb_user where id = 10 or age = 23;
explain select * from tb_user where phone = '17799990017' or age = 23;
```



第一条执行计划：

![image-20220902172253891](.\assets\image-20220902172253891.png)



第二条执行计划：

![image-20220902172349381](.\assets\image-20220902172349381.png)



由于age没有索引，所以即使id、phone有索引，索引也会失效。所以需要针对于age也要建立索引。

```sql
create index idx_user_age on tb_user(age);
```



这一下再执行上面的sql，查看执行计划，发现索引成功了



#### 3.6.4.5:数据分布影响

如果MySQL评估使用索引比全表更慢，则不使用索引

phone现在存在索引，phone的最小值目前是17799990000，最大是17799990023

```sql
explain select * from tb_user where phone >= '17799990023';
explain select * from tb_user where phone >= '17799990000';
```



第一条走了索引：

![image-20220902172943023](.\assets\image-20220902172943023.png)



第二条没走索引：

![image-20220902173023388](.\assets\image-20220902173023388.png)



MySQL在查询时，会评估使用索引的效率与走全表扫描的效率，如果走全表扫描更快，则放弃 索引，走全表扫描。 因为索引是用来索引少量数据的，如果通过索引查询返回大批量的数据，则还不 如走全表扫描来的快，此时索引就会失效



我的表数据全部都是有数据的，不存在profession为Null的数据

同理：

```sql
# 这个查询结果一条数据都没有，走索引更快，所以mysql会评估，最终走索引
explain select * from tb_user where profession is null;

# 这个查询结果是全部的数据，也就是全表，所以mysql会评估，最终不走索引，直接全表扫描
explain select * from tb_user where profession is not null;
```

注意：is null 和 is not null和走不走索引跟关键字没关系，凭数据的分布决定，由mysql决定



### 3.6.5:为SQL添加提示

目前tb_user只存在这下面几个索引

![image-20220902210311246](.\assets\image-20220902210311246.png)

执行  explain select * from tb_user where profession = '软件工程'  会走索引



我们假如再添加一个索引

```sql
create index idx_user_profession on tb_user(profession);
```



这个时候查看sql执行计划

![image-20220902210539399](.\assets\image-20220902210539399.png)

存在多个索引的时候，可能用到的索引有两个，但是用到的是联合索引，这个是Mysql自己评判的



那么，我们能不能在查询的时候，自己来指定使用哪个索引呢？ 

答案是肯定的，此时就可以借助于 MySQL的SQL提示来完成。 接下来，介绍一下SQL提示。

SQL提示，是优化数据库的一个重要手段，简单来说

就是在SQL语句中加入一些人为的提示来达到优化操作的目的。



- use index：建议MySQL使用哪一个索引完成查询(仅仅是建议,内部还会再次进行评估)
- ignore index：略指定的索引
- force index：强制使用索引



我们假如想使用我们刚刚创建的单列索引，就可以加上提示

```sql
explain select * from tb_user use index(idx_user_pro) where profession = '软件工
程';
```

![image-20220902211030600](.\assets\image-20220902211030600.png)



忽略单列索引

```sql
explain select * from tb_user ignore index(idx_user_profession) where profession = '软件工程';
```

![image-20220902211116171](.\assets\image-20220902211116171.png)



强制使用

```sql
explain select * from tb_user force index(idx_user_profession) where profession = '软件工程';
```

![image-20220902211348612](.\assets\image-20220902211348612.png)



### 3.6.6:覆盖索引

尽量使用覆盖索引，减少select *

那么什么是覆盖索引呢？ 覆盖索引是指查询使用了索引，并且需要返回的列，在该索引中已经全部能够找到，关注的不是where之后的，而是select的字段都能在索引找到，这类查询就叫做覆盖索引

接下来，我们来看一组SQL的执行计划，看看执行计划的差别，然后再来具体做一个解析



目前存在的索引是 profession,age, status



```sql
explain select id, profession from tb_user where profession = '软件工程' and age =
31 and status = '0' ;

explain select id,profession,age, status from tb_user where profession = '软件工程'
and age = 31 and status = '0' ;

explain select id,profession,age, status, name from tb_user where profession = '软
件工程' and age = 31 and status = '0' ;

explain select * from tb_user where profession = '软件工程' and age = 31 and status
= '0';
```



从上述的执行计划我们可以看到，这四条SQL语句的执行计划前面所有的指标都是一样的，看不出来差异。但是此时，我们主要关注的是后面的Extra，前面两条SQL的结果为 Using where; Using Index ; 而后面两条SQL的结果为: Null或Using index condition



- Using where & Using index

查找使用了索引，但是需要的数据都在索引列中能找到，所以不需要回表查询数据

- Null & Using index condition

查找使用了索引，但是需要回表查询数据



```apl
在tb_user表中有一个联合索引 idx_user_pro_age_sta，该索引关联了三个字段
profession、age、status，而这个索引也是一个二级索引，因为我们这个表存在主键
所以聚集索引是PRIMARY，所以叶子节点下面挂的是这一行的主键id
所以当我们查询返回的数据在 id、profession、age、status 之中，则直接走二级索引
直接返回数据了。 如果超出这个范围，就需要拿到主键id，再去扫描聚集索引，再获取额外的数据了
这个过程就是回表。 而我们如果一直使用select * 查询返回所有字段值，很容易就会造成回表
查询（除非是根据主键查询，此时只会扫描聚集索引）。
```



我们一起再来看下面的这组SQL的执行过程

![image-20220902213612918](.\assets\image-20220902213612918.png)

- id是主键，是一个聚集索引
- name字段建立了普通索引，是一个二级索引（辅助索引）

![image-20220902213648294](.\assets\image-20220902213648294.png)

- 聚集索引挂的是这一行数据
- 二级索引节点挂的是主键id
- 

执行SQL : select * from tb_user where id = 2;

直接走聚集索引直接拿到行数据就可以了



执行SQL：selet id,name from tb_user where name = 'Arm';

先走二级索引，拿到Arm和Arm下悬挂的id，不涉及回表

涉及的字段在这个索引的结构下都能找到，这个就叫做覆盖索引



执行SQL：selet id,name,gender from tb_user where name = 'Arm';

![image-20220902214046544](.\assets\image-20220902214046544.png)



发现在这个索引的数据结构只能找到部分，还差一个gender字段，这个就不是覆盖索引

还需要拿到节点下面的id，再进行回表查询

所以为什么不建议select *

正是因为根据除聚集索引以外的索引或者是全部字段的联合索引，都会涉及到回表查询



思考题：

一张表, 有四个字段(id, username, password, status) 由于数据量大

需要对以下SQL语句进行优化, 该如何进行才是最优方案: 

select id,username,password from tb_user where username = 'zhangsan';



问题解决：

- 性能优化，首先是加索引
- 字段查询的比较多，容易回表查询，所以，我们可以针对username,password建立联合索引
- 这样就不会执行回表查询，直接执行了覆盖索引



### 3.6.7:前缀索引

当字段类型为字符串（varchar，text，longtext等）时，有时候需要索引很长的字符串

这会让 索引变得很大，查询时，浪费大量的磁盘IO， 影响查询效率

此时可以只将字符串的一部分前缀，建立索引，这样可以大大节约索引空间，从而提高索引效率



语法：

```sql
create index idx_xxxx on table_name(column(n)) ;
```

n代表索引的字符串长度

事实上，我们使用一个很大的数据的前缀就可以很好地拿到这个数据



前缀长度选择：

可以根据索引的选择性来决定，选择性是指不重复的索引值(基数)和数据表的记录总数的比值， 索引选择性越高则查询效率越高，唯一索引的选择性是1，这是最好的索引选择性，性能也是最好的



假设tb_user的email是一个特别大的字段，维护数据结构消耗巨大，我们就可以这样来确定长度

两个参数：

- 表的总记录数

```sql
select count(*) from tb_user
```

- 不重复的记录数

```sql
select count(distinct email) from tb_user
```

- 计算

```sql
select count(distinct email) / count(*) from tb_user
```

- 当然我们不需要全部，只需要截取部分【假设截取前10个数据作为索引】

```sql
select count(distinct substring(email,1,10)) / count(*) from tb_user
```

- 假设截取前八个还是为1，我们还可以继续截取前九个

```sql
select count(distinct substring(email,1,10)) / count(*) from tb_user
```

- 我们的目的就是尽可能的让索引的选择性为1
- 我们最后加入选择了5个作为前缀



优点

- 减少了大数据量的建立索引或者维护索引的成本

缺点

- 如果在索引的选择性小于一的时候，回表的概率会增加

举例：

email为 536509593@qq.com 和 536509673@qq.com的两个邮箱

我们假如建立的索引只取了前5个字段，则我们查询的索引都是 53650

所以还是需要拿到数据下面的主键id，进行回表查询



前缀索引的查询流程

![image-20220902221603964](.\assets\image-20220902221603964.png)



### 3.6.8:单列索引与联合索引



- 单列索引：即一个索引只包含单个列。 

- 联合索引：即一个索引包含了多个列。



我们的phone和name都是有索引的，存在单列的phone和name索引

```sql
explain select * from tb_user where phone='17799990010' and name='韩信'
```



查看执行计划

![image-20220902231301604](.\assets\image-20220902231301604.png)

发现只走了phone的索引，没有走name的索引，phone的所有并不包含name

所以虽然加入了两个索引，但是不满足覆盖索引，还是会执行回表索引



通过上述执行计划我们可以看出来，在and连接的两个字段 phone、name上都是有单列索引的，但是 最终mysql只会选择一个索引，也就是说，and连接的两个索引，只能走一个字段的索引，此时是会回表查询的。



我们再来创建一个phone和name字段的联合索引来查询一下执行计划。

```sql
create unique index idx_user_phone_name on tb_user(phone,name);
```



执行sql语句

```sql
explain select * from tb_user where phone='17799990010' and name='韩信';
```

![image-20220902231620595](.\assets\image-20220902231620595.png)

发现三个索引，只执行了一个索引，走了联合索引，是mysql自己选择的，满足覆盖索引



在业务场景中，如果存在多个查询条件，考虑针对于查询字段建立索引时，建议建立联合索引，而非单列索引。因为涉及到多个and的单列索引，但是mysql最后还是只执行了一个索引，这样就不满足了覆盖索引，最后还是会走回表查询



如果查询使用的是联合索引，联合索引的具体结构示意图如下：

![image-20220902232119422](.\assets\image-20220902232119422.png)

会先进行phone排序，phone相同的时候再根据name排序，同样属于二级索引，悬挂的依然是主键ID

当根据联合索引查询联合索引的字段的时候，就满足覆盖查询，不会进行回表查询

建立联合字段的时候，字段的顺序是有讲究的，而且还有 最左前缀法则 等约束



### 3.6.9:索引设计原则

- 针对于数据量较大，且查询比较频繁的表建立索引【超过百万一般就算比较大了】
- 针对于常作为查询条件(where)排序(order by)分组(group by)操作的字段建立索引
- 尽量选择区分度高的列作为索引，尽量建立唯一索引，区分度越高，使用索引的效率越高
- 如果是字符串类型的字段，字段的长度较长，可以针对于字段的特点，建立前缀索引
- 尽量使用联合索引，减少单列索引，查询字段较多的时候，联合索引很多时候可以满足覆盖索引，节省存储空间，避免回表，提高查询效率
- 要控制索引的数量，索引并不是多多益善，索引越多，维护索引结构的代价也就越大，会影响增 删改的效率，因为涉及数据改变的时候，会维护索引的数据结构
- 如果索引列不能存储NULL值，请在创建表时使用NOT NULL约束它。当优化器知道每列是否包含 NULL值时，它可以更好地确定哪个索引最有效地用于查询



# 4:SQL优化



## 4.1:插入数据优化

### 4.1.1:insert

如果我们需要一次性往数据库表中插入多条记录，可以从以下三个方面进行优化

```sql
insert into tb_test values(1,'tom');
insert into tb_test values(2,'cat');
insert into tb_test values(3,'jerry');
```



优化方案一：批量插入数据

```sql
Insert into tb_test values(1,'Tom'),(2,'Cat'),(3,'Jerry');
```

优化原理：上面三条语句就会跟mysql进行建立连接， 构建网络传输，都会经过

- 连接层
- 服务层
- 引擎层
- 存储层

相当于进行三次



使用values的话，就只经历一次

但是values的话是不建议超过1000的



优化方案二：手动控制事务

大数量的插入，默认mysql自动提交回滚事务，大数据量的话，就会频繁的开启事务，提交事务

我们可以在插入大规模数据的时候设置手动开启事务，插入数据之后手动提交事务

```sql
start transaction;
insert into tb_test values(1,'Tom'),(2,'Cat'),(3,'Jerry');
insert into tb_test values(4,'Tom'),(5,'Cat'),(6,'Jerry');
insert into tb_test values(7,'Tom'),(8,'Cat'),(9,'Jerry');
commit;
```



优化方案三：主键顺序插入，性能要高于乱序插入

```sql
主键乱序插入 : 8 1 9 21 88 2 4 15 89 5 7 3
主键顺序插入 : 1 2 3 4 5 7 8 9 15 21 88 89
```



#### 4.1.2:大批量插入数据

如果一次性需要插入大批量数据(比如: 几百万的记录)，使用insert语句插入性能较低，此时可以使 用MySQL数据库提供的load指令进行插入。操作如下：



文件里面的数据，注意：只存在数据，不是sql语句

![image-20220903094434991](.\assets\image-20220903094434991.png)



使用load加载之后

![image-20220903093759359](.\assets\image-20220903093759359.png)



使用步骤：

- 客户端连接服务端时，加上参数 -–local-infile 

```sql
mysql –-local-infile -uroot -p
```

- 设置全局参数local_infile为1，开启从本地加载文件导入数据的开关 

```sql
set global local_infile = 1;
```

- 先把表的结构建立好
- 再执行load指令将准备好的数据，加载到表结构中

```sql
load data local infile '/root/sql1.log' into table tb_user fields terminated by ',·' lines terminated by '\n' 
```

- 在load时，主键顺序插入性能依然高于乱序插入



## 4.2:主键优化

在上一小节，我们提到，主键顺序插入的性能是要高于乱序插入的



数据组织方式：

在InnoDB存储引擎中，表数据都是根据主键顺序组织存放的，这种存储方式的表称为索引组织表   

![image-20220903094928348](.\assets\image-20220903094928348.png)

数据都是在叶子节点里面的，非叶子节点仅仅起到索引，判断范围的作用

所有的数据，叶子和非叶子节点的数据都是存放在页里面的，大小恒定16KB

回顾InnoDB逻辑存储结构

![image-20220901175441089](.\assets\image-20220901175441089.png)

页是InnoDB管理的最小单元

- 一个区含有64个页
- 页用来存放数据row
- row里面存放各个字段的值

数据行row是记录在逻辑结构 page 页中的，而每一个页的大小是固定的，默认16K。 那也就意味着，一个页中所存储的行也是有限的，如果插入的数据行row在该页存储不下，将会存储到下一个页中，页与页之间会通过指针连接



页分裂：

页可以为空，也可以填充一半，也可以填充100%。每个页包含了2-N行数据(如果一行数据过大，会行 溢出)，根据主键排列



### 4.2.1:顺序插入



- 主键顺序插入效果一

从磁盘中申请页， 主键顺序插入

![image-20220903095633686](.\assets\image-20220903095633686.png)



- 第一个页没有满，继续往第一页插入

![image-20220903095704760](.\assets\image-20220903095704760.png)



- 当第一个也写满之后，再写入第二个页，页与页之间会通过指针连接

![image-20220903095732727](.\assets\image-20220903095732727.png)



- 当第二页写满了，再往第三页写入

![image-20220903095754324](.\assets\image-20220903095754324.png)



### 4.2.2:乱序插入



- 假如1,2页都已经写满了，存放了如图所示的数据

![image-20220903100012562](.\assets\image-20220903100012562.png)



- 此时再插入id为50的记录，我们来看看会发生什么现象，会再次开启一个页，写入新的页中吗？

![image-20220903100041814](.\assets\image-20220903100041814.png)

不会。因为，索引结构的叶子节点是有顺序的。按照顺序，应该存储在47之后。



![image-20220903100108294](.\assets\image-20220903100108294.png)



- 但是47所在的1页，已经写满了，存储不了50对应的数据了。 那么此时会开辟一个新的页 3

![image-20220903100157362](.\assets\image-20220903100157362.png)



- 但是并不会直接将50存入3页，因为1页的数据量很大了【达到阈值】会先将1页后一半的数据，移动到3页

![image-20220903100245013](.\assets\image-20220903100245013.png)



- 然后在3页，插入50

![image-20220903100309284](.\assets\image-20220903100309284.png)



- 移动数据，并插入id为50的数据之后，那么此时，这三个页之间的数据顺序是有问题的。 1页的下一个页，应该是3， 3的下一个页是2。 所以，此时，需要重新设置链表指针。

![image-20220903100420103](.\assets\image-20220903100420103.png)



上述的这种现象，称之为 "页分裂"，是比较耗费性能的操作

所有乱序插入的时候，很容易产生页分裂，性能较差



### 4.2.3:页合并



- 目前表中已有数据的索引结构(叶子节点)如下：

![image-20220903100639575](.\assets\image-20220903100639575.png)



当我们对已有数据进行删除时，具体的效果如下: 

- 当删除一行记录时，实际上记录并没有被物理删除，只是记录被标记（flaged）为删除并且它的空间 变得允许被其他记录声明使用。

![image-20220903100716731](.\assets\image-20220903100716731.png)

- 当我们继续删除2页的数据记录

![image-20220903100813433](.\assets\image-20220903100813433.png)

- 当页中删除的记录达到阈值 MERGE_THRESHOLD（默认为页的50%），InnoDB会开始寻找最靠近

  页（前 或后）看看是否可以将两个页合并以优化空间使用

![image-20220903100856476](.\assets\image-20220903100856476.png)

- 发现3页可以合并

![image-20220903100924537](.\assets\image-20220903100924537.png)

- 删除数据，并将页合并之后，再次插入新的数据20，则直接插入3页

![image-20220903101000017](.\assets\image-20220903101000017.png)

这个里面所发生的合并页的这个现象，就称之为 "页合并"

MERGE_THRESHOLD：合并页的阈值，可以自己设置，在创建表或者创建索引时指定。



### 4.2.4:设计原则

- 满足业务需求的情况下，尽量降低主键的长度

```apl
因为二级索引很多，二级索引悬挂的是主键，维护这样的数据结构，就会大量消耗性能
```

- 插入数据时，尽量选择顺序插入，选择使用AUTO_INCREMENT自增主键

```apl
这样就肯定是顺序插入，不会导致页分裂，而且主键长度低，维护索引简单
```

- 尽量不要使用UUID做主键或者是其他自然主键，如身份证号

```apl
这样的长度很长，检索性能相对差，而且无序，容易页分裂
```

- 业务操作时，避免对主键的修改

```apl
修改主键，会改变各类索引的结构，代价很大
```





## 4.3:order by优化

MySQL的排序，有两种方式：



Using filesort：通过表的索引或全表扫描，读取满足条件的数据行，然后在排序缓冲区sort buffer中完成排序操作，所有不是通过索引直接返回排序结果的排序都叫 FileSort 排序。



Using index : 通过有序索引顺序扫描直接返回有序数据，这种情况即为 using index，不需要 额外排序，操作效率高。



对于以上的两种排序方式，Using index的性能高，而Using filesort的性能低

我们在优化排序 操作时，尽量要优化为 Using index。



- tb_user目前的索引情况

![image-20220903102100347](.\assets\image-20220903102100347.png)



- age没有单独的索引

```sql
explain select id,age,phone from tb_user order by age ;
```

- 查看执行计划的extra



![image-20220903102221377](.\assets\image-20220903102221377.png)



```sql
explain select id,age,phone from tb_user order by age, phone ;
```

- 由于 age, phone 都没有索引，所以此时再排序时，出现Using filesort， 排序性能较低。

![image-20220903102221377](.\assets\image-20220903102221377.png)



- 创建索引

```sql
create index idx_user_age_phone on tb_user(age,phone);
```

- 创建联合索引后，根据 age 或者 age, phone 进行升序排序

```sql
explain select id,age,phone from tb_user order by age;
explain select id,age,phone from tb_user order by age,phone;
```

![image-20220903102517746](.\assets\image-20220903102517746.png)

说明直接使用索引就返回了排序的数据

注意：排序时,也需要满足最左前缀法则,否则也会出现 filesort。因为在创建索引的时候， age是第一个字段，phone是第二个字段，所以排序时，也就该按照这个顺序来，否则就会出现 Using filesort，比如这个就会出现filesort

```sql
explain select id,age,phone from tb_user order by phone,age
```





- 创建索引后，根据age, phone进行降序排序

```sql
explain select id,age,phone from tb_user order by age desc , phone desc ;
```

![image-20220903103226199](.\assets\image-20220903103226199.png)

也出现 Using index， 但是此时Extra中出现了 Backward index scan，这个代表反向扫描索 引，因为在MySQL中我们创建的索引，默认索引的叶子节点是从小到大排序的，而此时我们查询排序 时，是从大到小，所以，在扫描时，就是反向扫描，就会出现 Backward index scan。 在 MySQL8版本中，支持降序索引，我们也可以创建降序索引



- 根据age, phone进行降序一个升序，一个降序

```apl
explain select id,age,phone from tb_user order by age asc , phone desc ;
```

![image-20220903103000148](.\assets\image-20220903103000148.png)

因为创建索引时，如果未指定顺序，默认都是按照升序排序的，而查询时

一个升序，一个降序，此时就会出现Using filesort。

![image-20220903103840467](.\assets\image-20220903103840467.png)

默认维护的数据结构都是ASC  A代表ASC  D代表DESC



为了解决上述的问题，我们可以创建一个索引，这个联合索引中 age 升序排序，phone 倒序排序



```sql
创建联合索引(age 升序排序，phone 倒序排序)
create index idx_user_age_phone_ad on tb_user(age asc ,phone desc);
```



这个时候再执行

```sql
explain select id,age,phone from tb_user order by age asc , phone desc ;
```

![image-20220903103951366](.\assets\image-20220903103951366.png)



总结

- 都升序或者降序，会走索引
- 一升一降默认不走索引
- 可以自己创建对应字段的升降序索引



默认创建的age,phone索引,先排age，再排phone

![image-20220903104433746](.\assets\image-20220903104433746.png)



我们创建的age asc,phone desc索引，先排age，再按降序排phone

![image-20220903104410451](.\assets\image-20220903104410451.png)



前提：使用了覆盖索引，因为我们建立的索引能拿到的值只有 id,age,phone

select的字段在 [id,age,phone]的子集才满足覆盖索引，如果查询其他的，还是会filesort



order by优化原则: 

- 根据排序字段建立合适的索引，多字段排序时，也遵循最左前缀法则
- 尽量使用覆盖索引
- 多字段排序, 一个升序一个降序，此时需要注意联合索引在创建时的规则（ASC/DESC）
- 如果不可避免的出现filesort，大数据量排序时，可以适当增大排序缓冲区大小sort_buffer_size(默认256k)。





## 4.4:group by优化

分组操作，我们主要来看看索引对于分组操作的影响。

目前表没有任何二级索引，只有聚集索引【主键】



Using temporary代表没有用到索引，使用了临时表

Using index代表使用了索引



- 没使用索引直接分组

```sql
explain select profession , count(*) from tb_user group by profession ;
```



![image-20220903105328102](.\assets\image-20220903105328102.png)



- 创建一个聚合索引 profession,age,status

```sql
create index idx_user_pro_age_sta on tb_user(profession,age,status);
explain select profession , count(*) from tb_user group by profession ;
```

![image-20220903105524178](.\assets\image-20220903105524178.png)



- 根据年龄分组

```sql
explain select age , count(*) from tb_user group by age ;
```

![image-20220903105721252](.\assets\image-20220903105721252.png)



- 根据profession,age分组

```sql
explain select age , count(*) from tb_user group by profession,age ;
```

![image-20220903105832934](.\assets\image-20220903105832934.png)

我们发现，如果仅仅根据age分组，就会出现 Using temporary ；而如果是根据 profession,age两个字段同时分组，则不会出现 Using temporary。原因是因为对于分组操作， 在联合索引中，也是符合最左前缀法则的。



所以，在分组操作中，我们需要通过以下两点进行优化，以提升性能： 

- 在分组操作时，可以通过索引来提高效率
- 分组操作时，索引的使用也是满足最左前缀法则的



## 4.5:limit优化

在数据量比较大时，如果进行limit分页查询，在查询时，越往后，分页查询效率越低

先查看表的数据量，100W

![image-20220903110135958](.\assets\image-20220903110135958.png)



分页测试：

```sql
select * from tb_sku limit 0,10; 			#65ms
select * from tb_sku limit 10,10;	 		#143ms
select * from tb_sku limit 1000000,10; 		#1242ms
select * from tb_sku limit 5000000,10; 		#5437ms
```

测试后：分页查询时，越往后，分页查询效率越低



原因：在进行分页查询时，如果执行 limit 2000000,10 ，此时需要MySQL排序前2000010 记 录，仅仅返回 2000000 - 2000010 的记录，其他记录丢弃，查询排序的代价非常大



官方建议的优化方式

一般分页查询时，通过创建覆盖索引 能够比较好地提高性能，可以通过覆盖索引加子查询形式进行优化。



需求：查询500W开始后的10条数据



优化测试：

- 首先满足覆盖索引

```sql
select * from tb_sku limit 9000000,10; 					#9437ms

select id from tb_sku order by id limit 9000000,10; 	 #5746ms
```

- 再子查询

```sql
select * from tb_sku where id in(select id from tb_sku limit 9000000,10);
```



```ABAP
This version of MySQL doesn't yet support 'LIMIT & IN/ALL/ANY/SOME subquery'
不支持在in里面使用 limit 关键字
```



- 可以使用多表联查

```sql
select * from tb_sku t , (select id from tb_sku order by id
limit 2000000,10) a where t.id = a.id;   #5644ms
相比于之前的9437ms，优化了将近4S
```



优化思路: 一般分页查询时，通过创建 覆盖索引 能够比较好地提高性能，可以通过覆盖索引加子查 询形式进行优化



## 4.6:count优化

![image-20220903110135958](.\assets\image-20220903110135958.png)

耗时9834ms

原因：取决于InnoDB的存储方式

MyISAM 引擎把一个表的总行数存在了磁盘上，因此执行 `count(*)` 的时候会直接返回这个数，效率很高； 但是如果是带条件的count，MyISAM也慢。 

InnoDB 引擎就麻烦了，它执行 `count(*)` 的时候，需要把数据一行一行地从引擎里面读出来，然后累积计数。

如果说要大幅度提升InnoDB表的count效率，

主要的优化思路：

- 自己计数(可以借助于redis这样的数据库进行,但是如果是带条件的count又比较麻烦了)
- 自己计数的时候，就直接计数到redis，获取的时候直接通过get key来拿到总数



count() 是一个聚合函数，对于返回的结果集，一行行地判断，如果 count 函数的参数不是 NULL，累计值就加 1，否则不加，最后返回累计值



用法：

- count（*）
- count（主键）
- count（字段）
- count（数字）

![image-20220903113325355](.\assets\image-20220903113325355.png)

注意：

- count(主键)就算返回整个表数量，因为主键不可能为null，取值id

- count(字段)返回的是不为null的字段的总数，要判定字段，效率最低
- count(1)就是遍历整张表，不取值，给返回的记录添加一个标识，读取到这个标识，就把计数加一，注意，并不是累计加上标识的数据，我这里的数字只是标识，不作为计算，写0也可以
- count(*)并不是我们想的那么差，mysql内部做了优化，并不会取任何值，直接累加



按照效率排序:

count(字段) < count(主键 id) < count(1) ≈ count(*)，所以尽量使用 `count(*)`





## 4.7:update优化

我们主要需要注意一下update语句执行时的注意事项。

```sql
update course set name = 'javaEE' where id = 1 ;
```

对于InnoDB当前的存储引擎，和默认的事务隔离级别下，会先把id=1的数据锁住，我们这里是根据id进行修改，则默认是行级锁，只要事务还没提交，id=1这一行数据的行锁就不会释放



查看原数据

![image-20220903115408976](.\assets\image-20220903115408976.png)

我们执行的时候，只要事务还没有提交，则id=1的数据就无法被其他的事务修改

但是我们是可以修改id=2 id=3这些数据的，因为我们这里是根据id进行修改，则默认是行级锁



另外的情况

我们先开启一个事务，执行下面的SQL

```sql
update course set name = 'javaEE' where name='Java' ;
```

再开启一个事务，执行下面的SQL

```sql
update course set name = 'Redis' where name='MYSQL' ;
```

按理说，可以修改name为MYSQL的数据，可是实际上却停在了这里

原因：行锁升级为了表锁，锁住了整张表，对其他行的数据无法执行修改



解决办法：

给name字段添加索引

这个时候根据name更新，默认就是行锁而不是表锁了



总结：

- InnoDB的行锁是针对索引加的锁，不是针对记录加的锁 

- 并且该索引不能失效，否则会从行锁升级为表锁





# 5:视图



## 5.1:介绍

视图（View）是一种虚拟存在的表,操作视图就像操作表一样

视图中的数据并不在数据库中实际存在，行和列数据来自定义视图的查询中使用的表

并且是在使用视图时动态生成的。 通俗的讲，视图只保存了查询的SQL逻辑，不保存查询结果。

所以我们在创建视图的时候，主要的工作就落在创建这条SQL查询语句上

视图的数据都来自于定义这个视图的表，视图只是封装了SQL的逻辑





## 5.2:语法

创建视图

```sql
CREATE [OR REPLACE] VIEW 视图名称[(列名列表)] AS SELECT语句 [ WITH [
CASCADED | LOCAL ] CHECK OPTION ]
```

```sql
create or replace view stu_v_1 as select student.id,name from student where id<=10;
```



查询视图

```sql
查看创建视图语句：SHOW CREATE VIEW 视图名称;
查看视图数据：SELECT * FROM 视图名称 ...... ;
```

```sql
-- 查看创建视图语句
show create view stu_v_1;
-- 查看视图数据
select * from stu_v_1;
-- 查看视图数据，加条件
select * from stu_v_1 where id<10;
```



修改视图

```sql
方式一：CREATE [OR REPLACE] VIEW 视图名称[(列名列表)] AS SELECT语句 [ WITH
[ CASCADED | LOCAL ] CHECK OPTION ]

方式二：ALTER VIEW 视图名称[(列名列表)] AS SELECT语句 [ WITH [ CASCADED |
LOCAL ] CHECK OPTION ]
```

```sql
-- 修改视图
create or replace view stu_v_1 as select student.id,name,no from student where id<=10;

-- 修改视图
alter view stu_v_1 as select student.id,name from student where id<=10;
```



删除视图

```sql
DROP VIEW [IF EXISTS] 视图名称 [,视图名称] ...
```

```sql
-- 删除视图
drop view if exists stu_v_1;
```



## 5.3:检查选项

上述我们演示了视图应该如何创建、查询、修改、删除

那么我们能不能通过视图来插入、更新数据呢？ 接下来，做一个测试



```sql
-- 创建视图
create or replace view stu_v_1 as select id,name from student where id<=20;

-- 查询视图
select * from stu_v_1;
```

![image-20220903153821060](.\assets\image-20220903153821060.png)



添加数据

```sql
-- 给视图添加数据
insert into stu_v_1 values(6,'张三');
insert into stu_v_1 values(7,'李四');
-- 查询视图
select * from stu_v_1;
```

![image-20220903154105072](.\assets\image-20220903154105072.png)

注意：视图是虚拟的，并不真实存储数据，实际上是插入到了这个视图的基表里

```sql
select * from student;
```

![image-20220903154130089](.\assets\image-20220903154130089.png)



我们定义视图的时候，创建视图写了id<=20

我们这个时候往视图插入id=30的数据

```sql
insert into stu_v_1 values(30,'周志雄');

-- 查询视图
select * from stu_v_1;
```

![image-20220903154130089](.\assets\image-20220903154130089.png)

发现并没有数据，因为我们定义的视图是id<=20，我们插入一个id为30的视图数据，是查不到的



当使用WITH CHECK OPTION子句创建视图时，MySQL会通过视图检查正在更改的每个行，例如插入、更新，删除，以使其符合视图的定义。如果不符合定义，就会报错，不允许操作。 MySQL允许基于另一个视图创建视图，它还会检查依赖视 图中的规则以保持一致性。为了确定检查的范围，mysql提供了两个选项： CASCADED 和 LOCAL ，默认值为 CASCADED 



```sql
create or replace view stu_v_1 as select student.id,name from student where id<=10;
```

上面这个就不会对插入视图的数据做任何的检查



```sql
create or replace view stu_v_1 as select student.id,name from student where id<=10 with cascaded check option 


create or replace view stu_v_1 as select student.id,name from student where id<=10 with local check option
```

以上两条就会做检查



### 5.3.1:CASCADED级联

比如，v2视图是基于v1视图的，如果在v2视图创建的时候指定了检查选项为 cascaded，但是v1视图 创建时未指定检查选项。 则在执行检查时，不仅会检查v2，还会级联检查v2的关联视图v1。

![image-20220903155405723](.\assets\image-20220903155405723.png)

相当于黄色的是一个无形的，是mysql自动添加的

```sql
-- 创建视图
create or replace view stu_v_1 as select student.id,name from student where id<=20;

-- 符合
insert into stu_v_1 values(20,'张三');

-- 符合，但是不检查
insert into stu_v_1 values(30,'李四');

-- 基于视图创建视图
create or replace view stu_v_2 as select id,name from stu_v_1 where id>=10 with cascaded check option;

-- 加入了with cascaded check option 会检查选项>=10 报错
insert into stu_v_2 values(7,'王五');

-- 加入了with cascaded check option
-- 会检查选项>=10成功，但是还会检查stu_v_1，发现30>20不符合，报错
insert into stu_v_2 values(30,'王五');

-- 再创建一个视图stu_v_3。基于stu_v_2
create or replace view stu_v_3 as select id,name from stu_v_2 where id<=15;

-- 成功，不管是否检查，条件都满足
insert into stu_v_3 values(11,'王五');

-- 成功，因为没有添加with cascaded check option，就不会对stu_v_3做检查
-- 直接去检查stu_v_2,再检查stu_v_1，都满足
insert into stu_v_2 values(17,'王五');

-- 成功，因为没有添加with cascaded check option，就不会对stu_v_3做检查
-- 直接去检查stu_v_2，成功--再检查stu_v_1，不满足，失败
insert into stu_v_2 values(30,'王五');
```



CASCADED：级联就是会检查相关联的视图





### 5.3.2:LOCAL本地

比如，v2视图是基于v1视图的，v2视图创建的时候指定了检查选项为 local 

- v1视图创建时未指定检查选项，则在执行检查时，只会检查v2，不会检查v2的关联视图v1
- v1指定了检查选项，检查v2之后，还会检查v1

```sql
-- 创建视图
create or replace view stu_v_4 as select student.id,name from student where id<=15;

-- 符合
insert into stu_v_4 values(15,'张三');

-- 符合，但是不检查
insert into stu_v_4 values(30,'李四');

-- 基于视图创建视图
create or replace view stu_v_5 as select id,name from stu_v_4 where id>=10 with local check option;

-- 加入了with local check option 会检查选项>=10 报错
insert into stu_v_5 values(7,'王五');

-- 加入了with local check option
-- 会检查选项>=10成功，由于是local 而且 由于stu_v_4没有加入检查选项，就直接成功了
insert into stu_v_5 values(20,'王五');
```



## 5.4:视图的更新

要使视图可更新，视图中的行与基础表中的行之间必须存在一对一的关系，也就是必须做好检查选项

重点是要满足一对一



如果视图包含以下任何一项，则该视图不可更新

- 聚合函数或窗口函数（SUM()、 MIN()、 MAX()、 COUNT()等） 
- DISTINCT
- GROUP BY
- HAVING 
- UNION 或者 UNION ALL



```sql
create or replace view stu_v_6 as select count(*) from student;

select * from stu_v_6;

这个视图代表的就是一个计数量
```



视图

![image-20220903162148306](.\assets\image-20220903162148306.png)



基表数据

![image-20220903162205721](.\assets\image-20220903162205721.png)



这个时候明显不是一对一，所有不能执行视图的更新操作





## 5.5:视图作用

简单：视图不仅可以简化用户对数据的理解，也可以简化他们的操作。那些被经常使用的查询，或者比较负载的sql，可以被定义为视图，从而使得用户不必为以后的操作每次指定全部的条件。



安全：数据库可以授权，但不能授权到数据库特定行和特定的列上。只能授权到表级别，不能授权到字段级别，通过视图可以授权到字段级别。用户只能查询和修改他们所能见到的数据。



数据独立：视图可帮助用户屏蔽真实表结构变化带来的影响。





## 5.6:案例

案例一

为了保证数据库表的安全性，开发人员在操作tb_user表时

只能看到的用户的基本字段，屏蔽 手机号和邮箱两个字段



```sql
-- 定义视图，屏蔽掉手机邮箱
create view tb_user_view as select id,name,profession,age,gender,status,createtime
from tb_user;

-- 使用的时候，直接使用视图
select * from tb_user_view;
```





案例二

查询每个学生所选修的课程（三张表联查），这个功能在很多的业务中都有使用到

为了简化操作，定义一个视图

```sql
-- 定义视图，封装很长的sql
create view tb_stu_course_view as select s.name student_name , s.no student_no ,
c.name course_name from student s, student_course sc , course c where s.id =
sc.studentid and sc.courseid = c.id;

-- 使用视图来完成多表联查
select * from tb_stu_course_view;
```





# 6:存储过程



## 6.1:介绍



问题引入：在一个业务里，一个业务可能需要先执行select，再update，再....

很多个SQL，就会涉及到多次的网络请求，跟数据库打多次交道，所以我们可以将常用的一些SQL

进行封装，如下的P1就是多个SQL的封装



![image-20220904161834759](.\assets\image-20220904161834759.png)



特点

- 封装，复用 -----------------------> 可以把某一业务SQL封装在存储过程中，需要用到的时候直接调用即可
- 可以接收参数，也可以返回数据 --------> 再存储过程中，可以传递参数，也可以接收返回值
- 减少网络交互，效率提升 -------------> 如果涉及到多条SQL，每执行一次都是一次网络传 输。 而如果封装在存储过程中，我们只需要网络交互一次可能就可以了



## 6.2:语法



### 6.2.1:创建

```sql
CREATE PROCEDURE 存储过程名称 ([ 参数列表 ])
BEGIN
-- SQL语句
END ;
```



### 6.2.2:调用

```sql
CALL 名称 ([ 参数 ]);
```



### 6.2.3:查看

```sql
-- 查询指定数据库的存储过程及状态信息
SELECT * FROM INFORMATION_SCHEMA.ROUTINES WHERE ROUTINE_SCHEMA = 'xxx';

-- 查询某个存储过程的定义
SHOW CREATE PROCEDURE 存储过程名称 ; 
```



### 6.2.4:删除

```sql
DROP PROCEDURE [ IF EXISTS ] 存储过程名称 ；
```



### 6.2.5:测试

```sql
--  创建储存过程，会生成routines
create  procedure  p1()
begin
    select count(*) from student;
end;

-- 调用存储过程
call p1();

-- 查看某个数据库的存储过程
select * from information_schema.ROUTINES where ROUTINE_SCHEMA='study';

-- 查看创建存储过程的SQL语句
show create procedure p1;

-- 删除存储过程
drop procedure if exists p1;
```

在命令行中，执行创建存储过程的SQL时，需要通过关键字 delimiter 指定SQL语句的结束符

默认是以;结尾，我们可以设置为 delimiter @@ 以两个@@结尾



## 6.3:变量

在MySQL中变量分为三种类型: 系统变量、用户定义变量、局部变量。



### 6.3.1:系统变量

系统变量 系统变量 是MySQL服务器提供，不是用户定义的，属于服务器层面。分为

- 全局变量（GLOBAL）
- 会话变量（SESSION）

开三个命令行就是三个会话，只是在当前黑窗口有效



- 查看系统变量

```sql
-- 查看所有系统变量
SHOW [ SESSION | GLOBAL ] VARIABLES ; 

-- 可以通过LIKE模糊匹配方式查找变量
SHOW [ SESSION | GLOBAL ] VARIABLES LIKE '......'; 

-- 查看指定变量的值
SELECT @@[SESSION | GLOBAL] 系统变量名; 
```

不指定的话。默认是session级别



```sql
-- 查看全部系统变量，不写session.global的话就是session
show variables ;

-- 查看全部系统变量，global级别
show global variables ;

-- 查看事务是否字段提交，假设我们只知道是auto开头
show variables like 'auto%';

-- 准确知道系统变量的值的话，使用@@变量查看系统变量
select @@autocommit;
```



- 设置系统变量

```sql
SET [ SESSION | GLOBAL ] 系统变量名 = 值 ;

SET @@[SESSION | GLOBAL] 系统变量名 = 值 ;
```



```sql
-- 设置session的事务不自动提交
set autocommit = 0;

-- 设置global的事务不自动提交【mysql重启之后，还是会恢复】
set global @@autocommit = 0;
```



### 6.3.2:自定义变量

@@代表系统变量

@代表用户变量，其作用域为当前连接，也就是session



- 赋值方式一，可以使用 = ，也可以使用 :=   推荐使用:=

```sql
SET @var_name = expr [, @var_name = expr] ... ;

SET @var_name := expr [, @var_name := expr] ... ;
```



- 赋值方式二【select只能使用 :=】

```sql
SELECT @var_name := expr [, @var_name := expr] ... ;

SELECT 字段名 INTO @var_name FROM 表名;
```



用户定义的变量无需对其进行声明或初始化，只不过获取到的值为NULL



- 使用

```sql
SELECT @var_name ;
```





```sql
-- 设置用户变量
set @name='zzx';

set @age:=20;

set @gender='男',@hobby='rap';

select @height:=173;

select count(*) into @count from tb_user;

-- 使用
select @name;
select @age;
select @gender;
select @hobby;
select @height;
select @count;
```



### 6.3.3:局部变量

局部变量是根据需要定义的在局部生效的变量，访问之前，需要DECLARE声明

可用作存储过程内的局部变量和输入参数，局部变量的范围是在其内声明的BEGIN ... END块

所以要想使用局部变量，要在创建一个存储过程的begin...end内使用



- 声明

```sql
DECLARE 变量名 变量类型 [DEFAULT ... ] ;
```

变量类型就是数据库字段类型：INT、BIGINT、CHAR、VARCHAR、DATE、TIME等



- 赋值

```sql
SET 变量名 = 值 ;
SET 变量名 := 值 ;
SELECT 字段名 INTO 变量名 FROM 表名 ... ;
```

- 查看

```sql
SELECT 变量名
```



```sql
-- 创建存储过程
create procedure p2()
begin
    -- 局部变量在存储过程内才能使用

    -- 声明局部变量
    declare stu_count int default 1;

    -- 给局部变量赋值
    set stu_count := 100;

    -- 使用局部变量
    select stu_count;
end


-- 使用存储过程
call p2();
```



## 6.4:if判断

if 用于做条件判断，具体的语法结构为：

```sql
IF 条件1 THEN
.....
ELSEIF 条件2 THEN -- 可选
.....
ELSE -- 可选
.....
END IF;
```

在if条件判断的结构中，ELSE IF 结构可以有多个，也可以没有。 ELSE结构可以有，也可以没有



案例：

根据定义的分数score变量，判定当前分数对应的分数等级

- score >= 85分，等级为优秀
- score >= 60分 且 score < 85分，等级为及格
- score < 60分，等级为不及格。



```sql
-- 创建存储过程
create procedure p3()
begin
    declare score int default 98;
    declare result varchar(10) default '及格';

    if score>=85 then
        set result:='优秀';

    elseif score>=60 then
        set result:='及格';

    else
        set result='不及格';
    end if;

    select result;
end;

-- 使用存储过程
call p3();
```

上述的需求我们虽然已经实现了，但是也存在一些问题，比如：score 分数我们是在存储过程中定义 死的，而且最终计算出来的分数等级，我们也仅仅是最终查询展示出来而已。那么我们能不能，把score分数动态的传递进来，计算出来的分数等级是否可以作为返回值返回呢？可以



## 6.5:参数

参数的类型，主要分为以下三种：IN、OUT、INOUT。 具体的含义如下：

![image-20220904205450322](.\assets\image-20220904205450322.png)



用法：

```sql
CREATE PROCEDURE 存储过程名称 ([ IN/OUT/INOUT 参数名 参数类型 ])
BEGIN
-- SQL语句
END ;
```



根据传入参数score，判定当前分数对应的分数等级，并返回

- score >= 85分，等级为优秀
- score >= 60分 且 score < 85分，等级为及格
- score < 60分，等级为不及格。



```sql
-- 创建存储过程
create procedure p4(in score int,out result varchar(10))
begin
    if score>=85 then
        set result:='优秀';

    elseif score>=60 then
        set result:='及格';

    else
        set result='不及格';
    end if;
end;

-- 使用存储过程，使用用户自定义变量来接收存储函数的输出
call p4(90,@result);
-- 拿到结果
select @result;
```



案例二：

将传入的200分制的分数，进行换算，换算成百分制【除以二】然后返回

分析：分数即是传入的，又是传出的，所以可以使用inout

```sql
-- 创建存储过程
create procedure p5(inout score double)
begin
    -- 重新赋值
    set score:=score/2;
end;

-- 使用存储过程，使用用户自定义变量来接收存储函数的输出
set @score:=188;
call p5(@score);
-- 拿到结果
select @score;
```



## 6.6:case



语法一：

```sql
-- 含义:当case_value的值为 when_value1时，执行statement_list1
-- 当值为 when_value2时，执行statement_list2
-- 否则就执行 statement_list
CASE case_value
WHEN when_value1 THEN statement_list1
[ WHEN when_value2 THEN statement_list2] ...
[ ELSE statement_list ]
END CASE;
```



语法二：

```sql
-- 含义:当条件search_condition1成立时，执行statement_list1
-- 当条件search_condition2成立时，执行statement_list2
-- 否则就执行 statement_list
CASE
WHEN search_condition1 THEN statement_list1
[WHEN search_condition2 THEN statement_list2] ...
[ELSE statement_list]
END CASE;

```



案例 

根据传入的月份，判定月份所属的季节（要求采用case结构）

- 1-3月份，为第一季度 
- 4-6月份，为第二季度 
- 7-9月份，为第三季度 
- 10-12月，为第四季度



## 6.7:while

while 循环是有条件的循环控制语句。满足条件后，再执行循环体中的SQL语句。具体语法为：

```sql
-- 先判定条件，如果条件为true，则执行逻辑，否则，不执行逻辑
WHILE 条件 DO
SQL逻辑...
END WHILE;
```



求1+2+3+....+n的值

```sql
-- 创建存储过程
create procedure p7(in n int,out result int)
begin

    set result:=0;
    while n>0 do
        set result:=result+n;
        set n:=n-1;
        end while;
        
end;

-- 调用
call p7(100,@result);

-- 查看结果
select @result;
```



## 6.8:repeat

repeat是有条件的循环控制语句, 当满足until声明的条件的时候，则退出循环,具体语法为：

```sql
-- 先执行一次逻辑，然后判定UNTIL条件是否满足，如果满足，则退出。如果不满足，则继续下一次循环
REPEAT
SQL逻辑...
UNTIL 条件
END REPEAT;
```

类似于do while

相同：都会先执行一次逻辑

区别：do while是条件成立的话，就执行，但是repeat是条件成立，退出



```sql
-- 创建存储过程
create procedure p8(in n int,out result int)
begin
    set result:=0;
    repeat
        set result:=result+n;
        set n:=n-1;
    until
        n<=0
        end repeat;
end;

-- 调用存储过程
call p8(100,@result);

select @result;
```



## 6.9:loop

LOOP 实现简单的循环，如果不在SQL逻辑中增加退出循环的条件，可以用其来实现简单的死循环。 LOOP可以配合一下两个语句使用：

- LEAVE ：配合循环使用，退出循环
- ITERATE：必须用在循环中，作用是跳过当前循环剩下的语句，直接进入下一次循环。



```sql
[begin_label:] LOOP
SQL逻辑...
END LOOP [end_label];

LEAVE label; -- 退出指定标记的循环体
ITERATE label; -- 直接进入下一次循环
```

上述语法中出现的 begin_label，end_label，label 指的都是我们所自定义的标记



计算从1累加到n的值，n为传入的参数值

```sql
-- 创建存储过程
create procedure p9(in n int,out result int)
begin
    set result:=0;

    sum:loop
        if n<=0 then
            leave sum;
        end if ;

        set result:=result+n;
        set n:=n-1;
    end loop;

end;

call p9(100,@result);

select @result;
```



计算从1到n之间的偶数累加的值，n为传入的参数值。

```sql
-- 创建存储过程
create procedure p10(in n int,out result int)
begin
    set result:=0;

    sum:loop
        if n<=0 then
            leave sum;
        elseif n%2=1 then
            set n:=n-1;
            iterate sum;
        end if ;

        set result:=result+n;
        set n:=n-1;
    end loop;
end;

call p10(100,@result);

select @result;
```



## 6.10:游标

游标（CURSOR）是用来存储查询结果集的数据类型 , 在存储过程和函数中可以使用游标对结果集进 行循环的处理。游标的使用包括游标的声明、OPEN、FETCH 和 CLOSE，其语法分别如下。



问题引入

```sql
create procedure p11()
begin
    declare stu_count int default 0;
    select count(*) into stu_count from student;
    select stu_count;
end;

call p11();
```



上面的可以成功，下面的不可以成功，下面是把表的数据赋值给stu_count【int类型】

```sql
create procedure p11()
begin
    declare stu_count int default 0;
    select * into stu_count from student;
    select stu_count;
end;

call p11();
```

报错：The used SELECT statements have a different number of columns

回忆之前的，我们写的所有变量都不能被表的数据进行赋值，所以出现了游标



- 声明游标

```sql
-- 把某个查询语句的结果封装到游标
DECLARE 游标名称 CURSOR FOR 查询语句 ;
```



- 打开游标

```sql
-- 使用之前必须先打开
OPEN 游标名称 ;
```



- 获取游标数据

```sql
FETCH 游标名称 INTO 变量 [, 变量 ] ;
```



- 关闭游标

```sql
CLOSE 游标名称 ;
```





案例 

根据传入的参数uage，来查询用户表tb_user中，所有的用户年龄小于等于uage的用户姓名和专业

并将用户的姓名和专业插入到所创建的一张新表(id,name,profession)

```sql
-- 逻辑:
-- A. 声明游标, 存储查询结果集
-- B. 准备: 创建表结构
-- C. 开启游标
-- D. 获取游标中的记录
-- E. 插入数据到新表中
-- F. 关闭游标


-- 声明存储过程
create procedure p1(in uage int)
begin

    -- 声明普通变量必须在声明游标之前
    declare uname varchar(50);
    declare upro varchar(50);

    -- 声明游标
    declare u_cursor cursor for select name,profession from tb_user where age<= uage;

    -- 创建新的表结构
    create table if not exists tb_user_pro(
        id int primary key auto_increment,
        name varchar(50) not null,
        profession varchar(50) not null
    );

    -- 开启游标
    open u_cursor;

    -- 获取游标数据，其实就可以理解为集合
    while true do

        -- 取出来的数据是name和profession，按照顺序赋值给uname,upro
        fetch u_cursor into uname,upro;

        -- 插入到表结构里面
        insert tb_user_pro values (null,uname,upro);

        end while;

    -- 关闭游标
    close u_cursor;
end;

call p1(40);
```

上述的功能，我们实现了，但是逻辑并不完善，而且程序执行完毕，获取不到数据，数据库还报错

但是新的表和数据是全部成功了，因为我们使用了while true

因为游标没有数据了，但是还在执行，就报错了，但是前面执行成功的不会撤销，新的表已经生成

数据也已经插入，该怎么修改呢？使用条件处理程序



## 6.11:条件处理程序

条件处理程序（Handler）可以用来定义在流程控制结构执行过程中遇到问题时相应的处理步骤

具体语法为：

```sql
DECLARE handler_action HANDLER FOR condition_value [, condition_value]... statement ;

handler_action 的取值：
	CONTINUE: 继续执行当前程序
	EXIT: 终止执行当前程序
	
condition_value 的取值：
	SQLSTATE sqlstate_value: 状态码，如 02000
	SQLWARNING: 所有以01开头的SQLSTATE代码的简写
	NOT FOUND: 所有以02开头的SQLSTATE代码的简写
	SQLEXCEPTION: 所有没有被SQLWARNING 或 NOT FOUND捕获的SQLSTATE代码的简写
```

实际上就类似java的异常处理机制，捕获到什么样的异常或者状态码就执行对应的SQL语句



```sql
-- 根据传入的参数uage，来查询用户表tb_user中，所有的用户年龄小于等于uage的用户姓名
-- name和专业profession，并将用户的姓名和专业插入到所创建的一张新表
-- (id,name,profession)中

-- 逻辑:
-- A. 声明游标, 存储查询结果集
-- B. 准备: 创建表结构
-- C. 开启游标
-- D. 获取游标中的记录
-- E. 插入数据到新表中
-- F. 关闭游标


-- 声明存储过程
create procedure p1(in uage int)
begin

    -- 声明普通变量必须在声明游标之前
    declare uname varchar(50);
    declare upro varchar(50);

    -- 声明游标
    declare u_cursor cursor for select name,profession from tb_user where age<= uage;

    -- 声明条件处理程序
    -- 游标没数据了，再取数据，状态码就会为02000，这个时候就会关闭游标并执行 close u_cursor
    declare exit handler for SQLSTATE '02000' close u_cursor;
    
    -- 也可以使用NOT FOUND: 所有以02开头的SQLSTATE代码的简写
    declare exit handler for not found close u_cursor;

    -- 创建新的表结构
    create table if not exists tb_user_pro(
        id int primary key auto_increment,
        name varchar(50) not null,
        profession varchar(50) not null
    );

    -- 开启游标
    open u_cursor;

    -- 获取游标数据，其实就可以理解为集合
    while true do

        -- 取出来的数据是name和profession，按照顺序赋值给uname,upro
        fetch u_cursor into uname,upro;

        -- 插入到表结构里面
        insert tb_user_pro values (null,uname,upro);

        end while;

    -- 关闭游标
    close u_cursor;
end;

call p1(40);
```



# 7:存储函数

存储函数是必须有返回值的存储过程，存储函数的参数只能是IN类型的。具体语法如下：

```sql
CREATE FUNCTION 存储函数名称 ([ 参数列表 ])
RETURNS type [characteristic ...]
BEGIN
	-- SQL语句
RETURN ...;
END ;
```



characteristic说明：

- DETERMINISTIC:相同的输入参数总是产生相同的结果
- NO SQL:不包含 SQL 语句。 
- READS SQL DATA:包含读取数据的语句，但不包含写入数据的语句。



计算从1累加到n的值，n为传入的参数值

```sql
-- 参数可以不写in 因为只能是in 不写也是in
create function sum1(n int)
returns int deterministic
begin
declare result int default 0;

while n>0 do
    set result:=result+n;
    set n:=n-1;
end while;

return result;
end;

select sum1(100);
```



注意：存储函数并不常用，因为必须有返回值，但是存储过程也可以实现，如果想执行一段没有返回值的SQL，那么存储函数就不行了



# 8:触发器



## 8.1:介绍

触发器是与表有关的数据库对象，指在insert/update/delete之前(BEFORE)或之后(AFTER)，触 发并执行触发器中定义的SQL语句集合。触发器的这种特性可以协助应用在数据库端确保数据的完整性 , 日志记录 , 数据校验等操作。类似Spring的AOP

使用别名OLD和NEW来引用触发器中发生变化的记录内容，这与其他的数据库是相似的。现在触发器还 只支持行级触发，不支持语句级触发



![image-20220905103354372](.\assets\image-20220905103354372.png)



- 行级触发

```apl
比如一条Update语句SQL影响了5条SQL，触发器触发5次
```

- 语句级触发器

```apl
比如一条Update语句SQL影响了5条SQL，触发器触发1次
```



## 8.2:语法

- 创建触发器

```sql
CREATE TRIGGER trigger_name
BEFORE/AFTER  INSERT/UPDATE/DELETE
ON tbl_name FOR EACH ROW -- 行级触发器，只支持行级触发器
BEGIN
trigger_stmt ;
END;
```



- 查看触发器

```sql
SHOW TRIGGERS ;
```



- 删除指定数据库的触发器

```sql
DROP TRIGGER [schema_name.]trigger_name ; -- 如果没有指定 schema_name，默认为当前数据库 
```



## 8.3:案例

通过触发器记录 tb_user 表的数据变更日志

将变更日志插入到日志表user_logs中, 包含增加, 修改 , 删除

表结构准备:

```sql
-- 准备工作 : 日志表 user_logs
create table user_logs(
	id int(11) not null auto_increment,
	operation varchar(20) not null comment '操作类型, insert/update/delete',
	operate_time datetime not null comment '操作时间',
	operate_id int(11) not null comment '操作的ID',
	operate_params varchar(500) comment '操作参数',
	primary key(`id`)
)engine=innodb default charset=utf8;，
```





## 8.4:案例插入

```sql
-- 给tb_user增加insert触发器
create trigger tb_user_insert_trigger
    after insert on tb_user for each row

    begin
        -- 编写触发器具体逻辑，插入数据到日志表
        insert into user_logs(id, operation, operate_time, operate_id,operate_params) 	  values
        (null, 'insert', now(), new.id, concat('插入的数据内容为:
        id=',new.id,',name=',new.name,', phone=', NEW.phone,', email=', NEW.email,',
        profession=', NEW.profession));
    end;



-- 查看当前数据库触发器
show triggers ;



-- 删除触发器
drop trigger tb_user_insert_trigger;



-- 插入数据到tb_user
insert into tb_user(id, name, phone, email, profession, age, gender, status,
createtime) VALUES (26,'三皇子','18809091212','erhuangzi@163.com','软件工程',23,'1','1',now());
```



## 8.5:案例修改

```sql
-- 给tb_user增加update的触发器
create trigger tb_user_update_trigger
    after update on tb_user for each row

begin
    insert into user_logs(id, operation, operate_time, operate_id, operate_params)
    VALUES
    (null, 'update', now(), new.id,
    concat('更新之前的数据: id=',old.id,',name=',old.name, ', phone=',
    old.phone, ', email=', old.email, ', profession=', old.profession,
    ' | 更新之后的数据: id=',new.id,',name=',new.name, ', phone=',
    NEW.phone, ', email=', NEW.email, ', profession=', NEW.profession));
end;

-- 查看
show triggers ;

-- 删除触发器
drop trigger tb_user_insert_trigger;

-- 更新
update tb_user set profession = '会计' where id = 23;
update tb_user set profession = '会计' where id <= 5;
```



## 8.6:案例删除

```sql
-- 给tb_user增加delete的触发器
create trigger tb_user_delete_trigger
    after delete on tb_user for each row
begin
    insert into user_logs(id, operation, operate_time, operate_id, operate_params)
    VALUES
    (null, 'delete', now(), old.id,
    concat('删除之前的数据: id=',old.id,',name=',old.name, ', phone=',
    old.phone, ', email=', old.email, ', profession=', old.profession));
end;

-- 查看
show triggers ;

-- 删除数据
delete from tb_user where id = 26;
```



总结就是 

- insert的时候，拿到的new是新插入的数据
- update的时候，拿到的old是之前的数据，new是更新后的数据
- delete的时候，拿到的old是删除之前的数据



# 9:mysql锁



## 9.1:概述

锁是计算机协调多个进程或线程并发访问某一资源的机制。在数据库中，除传统的计算资源（CPU、 RAM、I/O）的争用以外，数据也是一种供许多用户共享的资源。如何保证数据并发访问的一致性、有 效性是所有数据库必须解决的一个问题，锁冲突也是影响数据库并发访问性能的一个重要因素。从这个角度来说，锁对数据库而言显得尤其重要，也更加复杂

MySQL中的锁，按照锁的粒度分，分为以下三类： 

- 全局锁：锁定数据库中的所有表
- 表级锁：每次操作锁住整张表
- 行级锁：每次操作锁住对应的行数据。



## 9.2:全局锁

全局锁就是对整个数据库实例加锁，加锁后整个实例就处于只读状态，后续的DML的写语句，DDL语 句，已经更新操作的事务提交语句都将被阻塞。 其典型的使用场景是做全库的逻辑备份，对所有的表进行锁定，从而获取一致性视图，保证数据的完整性。 

为什么全库逻辑备份，就需要加全局锁呢？



我们一起先来分析一下不加全局锁，可能存在的问题

假设在数据库中存在这样三张表: 

- tb_stock 库存表
- tb_order 订单表
- tb_orderlog 订单日志表

![image-20220905170943467](.\assets\image-20220905170943467.png)

- 在进行数据备份时，先备份了tb_stock库存表
- 然后接下来，在业务系统中，执行了下单操作，扣减库存，生成订单
- 更新tb_stock表，插入 tb_order表
- 然后再执行备份 tb_order表的逻辑。 业务中执行插入订单日志操作
- 最后，又备份了tb_orderlog表

此时备份出来的数据，是存在问题的。因为备份出来的数据

tb_stock表与tb_order表的数据不一 致(有最新操作的订单信息,但是库存数没减)



再来分析一下加了全局锁后的情况

![image-20220905171325876](.\assets\image-20220905171325876.png)

对数据库进行进行逻辑备份之前，先对整个数据库加上全局锁，一旦加了全局锁之后，其他的DDL、 DML全部都处于阻塞状态，但是可以执行DQL语句，也就是处于只读状态，而数据备份就是查询操作。 那么数据在进行逻辑备份的过程中，数据库中的数据就是不会发生变化的，这样就保证了数据的一致性和完整性



- 加全局锁

```sql
flush tables with read lock ;
```



- 数据备份

```apl
-- 把某个数据库的信息备份到bak.sql里
mysqldump -h192.168.61.131 -uroot –pJXLZZX79 数据库名 > bak.sql
```



- 释放全局锁

```sql
unlock tables ;
```



数据库中加全局锁，是一个比较重的操作，存在以下问题： 

- 如果在主库上备份，那么在备份期间都不能执行更新，业务基本上就得停摆
- 如果在从库上备份，那么在备份期间从库不能执行主库同步过来的二进制日志（binlog），会导 致主从延迟。

在InnoDB引擎中，我们可以在备份时加上参数 --single-transaction 参数来完成不加锁的一致 性数据备份

```sql
mysqldump --single-transaction -h192.168.61.131 -uroot –p123456 数据库名 > bak.sql
```





## 9.3:表级锁



表级锁，每次操作锁住整张表。锁定粒度大，发生锁冲突的概率最高，并发度最低

应用在MyISAM、 InnoDB、BDB等存储引擎中

讲解的是InnoDB引擎下的的表级锁

对于表级锁，主要分为以下三类： 

- 表锁 
- 元数据锁（meta data lock，MDL） 
- 意向锁



### 9.3.1:表锁

对于表锁，分为两类： 

- 表共享读锁（read lock） 
- 表独占写锁（write lock）



语法 

加锁：

- lock tables 表名... read/write。

释放锁：

- unlock tables / 客户端断开连接 。





- 读锁情况

![image-20220905174814336](.\assets\image-20220905174814336.png)



终端一：

![image-20220905174423769](.\assets\image-20220905174423769.png)



终端二：

![image-20220905174609778](.\assets\image-20220905174609778.png)



- 写锁情况

![image-20220905174915620](.\assets\image-20220905174915620.png)



终端一：

![image-20220905175205584](.\assets\image-20220905175205584.png)



终端二：

![image-20220905175316927](.\assets\image-20220905175316927.png)

读写都会被阻塞



### 9.3.2:元数据锁

meta data lock , 元数据锁，简写MDL

MDL加锁过程是系统自动控制，无需显式使用，在访问一张表的时候会自动加上。MDL锁主要作用是维护表中元数据的数据一致性，在表上有活动事务的时候，不可以对元数据进行写入操作。

主要是为了避免DML与 DDL冲突，保证读写的正确性。



元数据：简单来说就是表结构

元数据锁：就是为了维护表结构的数据一致性



在MySQL5.5中引入了MDL，当对一张表进行增删改查的时候，加MDL读锁(共享)

当对表结构进行变更操作的时候，加MDL写锁(排他)。



常见的SQL操作时，所添加的元数据锁：

![image-20220905180109465](.\assets\image-20220905180109465.png)



终端一：

![image-20220905180853370](.\assets\image-20220905180853370.png)



终端二：

![image-20220905181011840](.\assets\image-20220905181011840.png)

阻塞



元数据锁实际上就是当我们存在还未提交的的增删改查事务的时候

是不能执行表或者数据库这些涉及结构的修改的



- 也就是数据定义语言【涉及结构修改的】和涉及到表结构数据的变化和展现的时候的锁



查看元数据锁

```sql
select object_type,object_schema,object_name,lock_type,lock_duration from
performance_schema.metadata_locks ;
```



我们可以验证

- 终端一开启事务之后，执行select之后不提交
- 终端二查看元数据锁，发现多了一个shared_read
- 终端二开启事务，执行update，不提交
- 终端二查看元数据锁，发现多了一个shared_write
- 由于shared_read和shared_write不会冲突，所以正常执行了
- 终端二执行alter，发现阻塞了
- 因为exclusive锁和shared_read和shared_write都会冲突
- 由于我们没提交事务，shared_read和shared_write都没释放
- 所以exclusive锁加入不进来



### 9.3.3:意向锁

为了避免DML在执行时，加的行锁与表锁的冲突，在InnoDB中引入了意向锁，使得表锁不用检查每行 数据是否加锁，使用意向锁来减少表锁的检查。

假如没有意向锁，客户端一对表加了行锁后，客户端二如何给表加表锁呢，来通过示意图简单分析一 下：



首先客户端一，开启一个事务，然后执行DML【根据id更新】操作，在执行DML语句时，会对涉及到的行加行锁

![image-20220905210125153](.\assets\image-20220905210125153.png)



当客户端二，想执行根据dept_id更新的时候，dept_id没有索引，那么就会对这张表加表锁，会检查当前表是的每一行否有对应的行锁，如果没有，则添加表锁，此时就 会从第一行数据，检查到最后一行数据，效率较低。

![image-20220905210234969](.\assets\image-20220905210234969.png)

所以引入了意向锁



存在意向锁之后：

客户端一，在执行DML操作时，会对涉及的行加行锁，同时也会对该整张表加上意向锁。

![image-20220905210340632](.\assets\image-20220905210340632.png)





而其他客户端，在对这张表加表锁的时候，会根据该表上所加的意向锁来判定是否兼容，从而判断是否可以成功加表锁，会直接判断意向锁是否兼容，而不用逐行判断行锁情况了。

![image-20220905210410447](.\assets\image-20220905210410447.png)



- 意向共享锁(IS): 由语句select ... lock in share mode添加 。 与表锁共享锁 (read)兼容，与表锁排他锁(write)互斥
- 意向排他锁(IX): 由insert、update、delete、select...for update添加 。与表锁共享锁(read)及排他锁(write)都互斥，意向锁之间不会互斥

一旦事务提交了，意向共享锁、意向排他锁，都会自动释放



查看意向锁及行锁的加锁情况

```sql
select object_schema,object_name,index_name,lock_type,lock_mode,lock_data from
performance_schema.data_locks;
```



- 开启事务，终端一添加意向共享锁
- 终端二添加表共享读锁，成功
- 终端二添加表独占写锁，失败
- 则意向共享锁和读锁兼容，和写锁不兼容





- 开启事务，终端一添加意向排它锁
- 终端二添加表共享读锁，阻塞
- 终端二添加表独占写锁，阻塞
- 则意向排他锁跟读锁写锁都不兼容





## 9.4:行级锁



### 9.4.1:介绍

行级锁，每次操作锁住对应的行数据。锁定粒度最小，发生锁冲突的概率最低，并发度最高。应用在 InnoDB存储引擎中。 InnoDB的数据是基于索引组织的，行锁是通过对索引上的索引项加锁来实现的，而不是对记录加的 锁。对于行级锁，主要分为以下三类：



行锁（Record Lock）：锁定单个行记录的锁，防止其他事务对此行进行update和delete。在 RC、RR隔离级别下都支持。

- RC：Read committed(Oracle默认)
- RR：Repeatable Read(Mysql默认)

![image-20220905224307725](.\assets\image-20220905224307725.png)

比如上面锁住34这一行下面悬挂的数据行【聚集索引】或者id【二级索引】





间隙锁（Gap Lock）：锁定索引记录间隙（不含该记录），确保索引记录间隙不变，防止其他事务在这个间隙进行insert，产生幻读。在RR隔离级别下都支持。

![image-20220905224546765](.\assets\image-20220905224546765.png)

- RR：Repeatable Read(Mysql默认)



回顾幻读

![image-20220901165211229](.\assets\image-20220901165211229.png)



临键锁（Next-Key Lock）：行锁和间隙锁组合，同时锁住数据，并锁住数据前面的间隙Gap。 在RR隔离级别下支持。

- RR：Repeatable Read(Mysql默认)

![image-20220905224813366](.\assets\image-20220905224813366.png)



### 9.4.1:行锁

InnoDB实现了以下两种类型的行锁： 

共享锁（S）：允许一个事务去读一行，阻止其他事务获得相同数据集的排它锁

排他锁（X）：允许获取排他锁的事务更新数据，阻止其他事务获得相同数据集的共享锁和排它锁。



![image-20220905225110310](.\assets\image-20220905225110310.png)



常见的SQL语句，在执行时，所加的行锁如下：

![image-20220905225213550](.\assets\image-20220905225213550.png)



默认情况下，InnoDB在 REPEATABLE READ事务隔离级别运行，InnoDB使用 next-key 锁进行搜索和索引扫描，以防止幻读。 

1：针对唯一索引进行检索时，对已存在的记录进行等值匹配时，将会自动优化为行锁。 

2：InnoDB的行锁是针对于索引加的锁，不通过索引条件检索数据，那么InnoDB将对表中的所有记 录加锁，此时就会升级为表锁。



查看加锁情况

```sql
select object_schema,object_name,index_name,lock_type,lock_mode,lock_data from
performance_schema.data_locks;
```



演示：

数据准备

```sql
CREATE TABLE `stu` (
    `id` int NOT NULL PRIMARY KEY AUTO_INCREMENT,
    `name` varchar(255) DEFAULT NULL,
    `age` int NOT NULL
) ENGINE = InnoDB CHARACTER SET = utf8mb4;

INSERT INTO `stu` VALUES (1, 'tom', 1);
INSERT INTO `stu` VALUES (3, 'cat', 3);
INSERT INTO `stu` VALUES (8, 'rose', 8);
INSERT INTO `stu` VALUES (11, 'jetty', 11);
INSERT INTO `stu` VALUES (19, 'lily', 19);
INSERT INTO `stu` VALUES (25, 'luci', 25);
```



验证行级锁的共享锁和 共享锁和排它锁的兼容性

- 终端一开启事务，先执行 select * from stu where id=1
- 终端二查看锁情况，空
- 终端一select * from stu where id=1 lock in share mode【生成共享锁】
- 终端二查看锁情况，多了个S，REC_NOT_GAP【行锁的共享锁，无间隙】
- 终端二开启事务，select * from stu where id=1 lock in share mode【成功，兼容】
- 终端二查看锁情况，又多了个S，REC_NOT_GAP
- 终端二对id为3的数据进行更新，成功，因为是行锁，只锁住了id为1的
- 终端二对id为1的数据进行更新，失败，行锁的共享锁和排它锁不兼容



验证行级锁的排他锁和 共享锁和排它锁的兼容性

- 终端一开启事务，先执行 update stu set name='ZZX' where id=1【生成排它锁】
- 终端二查看锁情况，X,REC_NOT_GAP【行锁的排它锁，无间隙】
- 终端二update stu set name='ZZX' where id=1，阻塞
- 终端二执行select * from stu where id=1 lock in share mode，阻塞



验证无索引行锁升级为表锁

- 终端一开启事务，先执行 update stu set name='ZZX' where name='JXL'【不使用索引就会生成表锁，这个数据是id为1的行数据】
- 终端二update stu set name='ZZX' where id=2，阻塞
- 给name增加索引
- 重复上述1，2操作，不会阻塞了，只会阻塞id为1的那条数据，表明为行锁了





### 9.4.2:间隙锁临键锁

默认情况下，InnoDB在 REPEATABLE READ事务隔离级别运行，InnoDB使用 next-key 锁进行搜 索和索引扫描，以防止幻读。



- 索引上的等值查询(唯一索引)，给不存在的记录加锁时, 优化为间隙锁 。
- 索引上的等值查询(非唯一普通索引)，向右遍历时最后一个值不满足查询需求时，next-key lock 退化为间隙锁。 
- 索引上的范围查询(唯一索引)--会访问到不满足条件的第一个值为止。



演示

1：索引上的等值查询(唯一索引)，给不存在的记录加锁时, 优化为间隙锁 

- 终端一开启事务，执行update stu set age=10 where id=5【id=5不存在】
- 终端二查看锁【发现有X,GAP】间隙锁
- 终端二执行间隙锁以内的数据的id的insert，阻塞，插入不进去
- 这就是RR解决幻读的原理



2：索引上的等值查询(非唯一普通索引，二级索引)，向右遍历时最后一个值不满足查询需求时

next-key lock 退化为间隙锁



我们知道InnoDB的B+树索引，叶子节点是有序的双向链表。 假如，我们要根据这个二级索引查询值 为18的数据，并加上共享锁，我们是只锁定18这一行就可以了吗？ 并不是，因为是非唯一索引，这个 结构中可能有多个18的存在，所以，在加锁时会继续往后找，找到一个不满足条件的值（当前案例中也 就是29）。此时会对18加临键锁，并对29之前的间隙加锁。



![image-20220905232824624](.\assets\image-20220905232824624.png)

会对18和18~29之间的间隙加锁



假如age存在二级索引，而且年龄排序分别为  1，3，8，10，23

- 终端一执行 select * from stu where age=3 lock in share mode
- 终端二查看锁，发现对3存在行级锁，对3~8存在间隙锁，为了防止有事务插入导致幻读





3：索引上的范围查询(唯一索引)--会访问到不满足条件的第一个值为止。



id存在聚集索引，而且id排序分别为  1，3，4，7，8，11，19，25

- 终端一执行 select * from stu where id>=19 lock in share mode
- 终端二查看锁，发现对19存在行级锁，对19~25存在间隙锁
- 对25到正无穷存在间隙锁
- 为了防止有事务插入导致幻读



# 10:InnoDB



## 10.1:逻辑存储结构



![image-20220901175441089](.\assets\image-20220901175441089.png)



表->段->区->页->行



表空间（Tablespace）

表空间是InnoDB存储引擎逻辑结构的最高层，则每张表都会有一个表空间（xxx.ibd），一个mysql实例可以对应多个表空间，用于存储记录、索引等数据



段（Segment）

分为数据段（Leaf node segment）、索引段（Non-leaf node segment）、回滚段 （Rollback segment），InnoDB是索引组织表，数据段就是B+树的叶子节点， 索引段即为B+树的非叶子节点。段用来管理多个Extent（区）



区 （Extent）

表空间的单元结构，每个区的大小为1M。 默认情况下， InnoDB存储引擎页大小为16K， 即一个区中一共有64个连续的页



页（Page）

是InnoDB 存储引擎磁盘管理的最小单元，每个页的大小默认为 16KB。为了保证页的连续性， InnoDB 存储引擎每次从磁盘申请 4-5 个区，表数据和索引等都在这里存储



行（Row）

InnoDB 存储引擎数据是按行进行存放的

在行中，默认有两个隐藏字段： 

- Trx_id：每次对某条记录进行改动时，都会把对应的事务id赋值给trx_id隐藏列
- Roll_pointer：每次对某条引记录进行改动时，都会把旧的版本写入到undo日志中，然后这个 隐藏列就相当于一个指针，可以通过它来找到该记录修改前的信息。



## 10.2:架构

MySQL5.5 版本开始，默认使用InnoDB存储引擎，它擅长事务处理，具有崩溃恢复特性，在日常开发 中使用非常广泛。下面是InnoDB架构图，左侧为内存结构，右侧为磁盘结构



![image-20220905235539835](.\assets\image-20220905235539835.png)





### 10.2.1:内存结构

![image-20220905235604899](.\assets\image-20220905235604899.png)

在左侧的内存结构中，主要分为这么四大块儿： 

Buffer Pool、Change Buffer、Adaptive Hash Index、Log Buffer。 接下来介绍一下这四个部分



- Buffer Pool 

InnoDB存储引擎基于磁盘文件存储，访问物理硬盘和在内存中进行访问，速度相差很大，为了尽可能 弥补这两者之间的I/O效率的差值，就需要把经常使用的数据加载到缓冲池中，避免每次访问都进行磁 盘I/O

在InnoDB的缓冲池中不仅缓存了索引页和数据页，还包含了undo页、插入缓存、自适应哈希索引以及 InnoDB的锁信息等等

缓冲池 Buffer Pool，是主内存中的一个区域，里面可以缓存磁盘上经常操作的真实数据，在执行增 删改查操作时，先操作缓冲池中的数据（若缓冲池没有数据，则从磁盘加载并缓存），然后再以一定频 率刷新到磁盘，从而减少磁盘IO，加快处理速度。



缓冲池以Page页为单位，底层采用链表数据结构管理Page。根据状态，将Page分为三种类型： 

• free page：空闲page，未被使用。 

• clean page：被使用page，数据没有被修改过。 

• dirty page：脏页，被使用page，数据被修改过，也中数据与磁盘的数据产生了不一致。



在专用服务器上，通常将多达80％的物理内存分配给缓冲池 

参数设置： show variables like 'innodb_buffer_pool_size';





- Change Buffer Change Buffer更改缓冲区（针对于非唯一二级索引页）

在执行DML语句时，如果这些数据Page 没有在Buffer Pool中，不会直接操作磁盘，而会将数据变更存在更改缓冲区 Change Buffer 中，在未来数据被读取时，再将数据合并恢复到Buffer Pool中，再将合并后的数据刷新到磁盘中。 Change Buffer的意义是什么呢?

先来看一幅图，这个是二级索引的结构图：

![image-20220906000110947](.\assets\image-20220906000110947.png)





-  Adaptive Hash Index自适应哈希

自适应hash索引，用于优化对Buffer Pool数据的查询。MySQL的innoDB引擎中虽然没有直接支持 hash索引，但是给我们提供了一个功能就是这个自适应hash索引。因为前面我们讲到过，hash索引在 进行等值匹配时，一般性能是要高于B+树的，因为hash索引一般只需要一次IO即可，而B+树，可能需 要几次匹配，所以hash索引的效率要高，但是hash索引又不适合做范围查询、模糊匹配等。 InnoDB存储引擎会监控对表上各索引页的查询，如果观察到在特定的条件下hash索引可以提升速度， 则建立hash索引，称之为自适应hash索引。

自适应哈希索引，无需人工干预，是系统根据情况自动完成

参数： adaptive_hash_index



-  Log Buffer

Log Buffer：日志缓冲区，用来保存要写入到磁盘中的log日志数据（redo log 、undo log）， 默认大小为 16MB，日志缓冲区的日志会定期刷新到磁盘中。如果需要更新、插入或删除许多行的事 务，增加日志缓冲区的大小可以节省磁盘 I/O

参数: 

innodb_log_buffer_size：缓冲区大小 

innodb_flush_log_at_trx_commit：日志刷新到磁盘时机，取值主要包含以下三个：



0: 每秒将日志写入并刷新到磁盘一次

1: 日志在每次事务提交时写入并刷新到磁盘，默认值

2: 日志在每次事务提交后写入，并每秒刷新到磁盘一次





### 10.2.2:磁盘结构



![image-20220906095445023](.\assets\image-20220906095445023.png)



- System Tablespace 

系统表空间是更改缓冲区的存储区域。如果表是在系统表空间而不是每个表文件或通用表空间中创建 的，它也可能包含表和索引数据。(在MySQL5.x版本中还包含InnoDB数据字典、undolog等)

参数：innodb_data_file_path

系统表空间，默认的文件名叫 ibdata1



-  File-Per-Table Tablespaces

如果开启了innodb_file_per_table开关 ，则每个表的文件表空间包含单个InnoDB表的数据和索引 ，并存储在文件系统上的单个数据文件中。

开关参数：innodb_file_per_table ，该参数默认开启

也就是每张表单独对应一个表空间

那也就是说，我们没创建一个表，都会产生一个表空间文件，如图：

![image-20220906095859343](.\assets\image-20220906095859343.png)



- General Tablespaces

通用表空间，需要通过 CREATE TABLESPACE 语法创建通用表空间，在创建表时，可以指定该表空间



A. 创建表空间

```sql
CREATE TABLESPACE ts_name ADD DATAFILE 'file_name' ENGINE = engine_name;
```

 B.创建表时指定数据存放到哪个表空间

```sql
CREATE TABLE xxx ... TABLESPACE ts_name;
```



```sql
create tablespace ts_zzx add datafile 'my.ibd' engine='innodb'

create table student(
    id int primary key auto_increment,
	name varchar(10)
)engine=innodb tablespace ts_zzx;
```



-  Undo Tablespaces

撤销表空间，MySQL实例在初始化时会自动创建两个默认的undo表空间【undo_001,undo_002】（初始大小16M），用于存储 undo log日志



- Temporary Tablespaces

InnoDB 使用会话临时表空间和全局临时表空间。存储用户创建的临时表等数据。



-  Doublewrite Buffer Files

双写缓冲区，innoDB引擎将数据页从Buffer Pool刷新到磁盘前，先将数据页写入双写缓冲区文件 中，便于系统异常时恢复数据

![image-20220906100938785](.\assets\image-20220906100938785.png)



- Redo Log

重做日志，是用来实现事务的持久性。该日志文件由两部分组成：重做日志缓冲（redo log buffer）以及重做日志文件（redo log）,前者是在内存中，后者在磁盘中。当事务提交之后会把所 有修改信息都会存到该日志中, 用于在刷新脏页到磁盘时,发生错误时, 进行数据恢复使用。 以循环方式写入重做日志文件，涉及两个文件：

![image-20220906100955051](.\assets\image-20220906100955051.png)





### 10.2.3:后台线程

后台线程就是将InnoDB的数据在合适的时机刷新到磁盘文件

![image-20220906101221099](.\assets\image-20220906101221099.png)

在InnoDB的后台线程中，分为4类，

分别是：Master Thread 、IO Thread、Purge Thread、 Page Cleaner Thread。



- Master Thread 

核心后台线程，负责调度其他线程，还负责将缓冲池中的数据异步刷新到磁盘中, 保持数据的一致性， 还包括脏页的刷新、合并插入缓存、undo页的回收 



- IO Thread

在InnoDB存储引擎中大量使用了AIO【异步非阻塞IO】来处理IO请求, 这样可以极大地提高数据库的性能，而IO Thread主要负责这些IO请求的回调

![image-20220906101357988](.\assets\image-20220906101357988.png)

我们可以通过以下的这条指令，查看到InnoDB的状态信息，其中就包含IO Thread信息

```sql
show engine innodb status \G;
```



- Purge Thread

主要用于回收事务已经提交了的undo log，在事务提交之后，undo log可能不用了，就用它来回收



- Page Cleaner Thread。

协助 Master Thread 刷新脏页到磁盘的线程，它可以减轻 Master Thread 的工作压力，减少阻塞





总结

- 业务操作的时候，会直接操作缓冲区
- 缓冲区没有数据，先把磁盘的加载到缓冲区，然后存储到缓冲区
- 增删改的时候，都是操作的缓冲区
- 缓冲区会以一定的频率刷新到磁盘中
- 在磁盘以永久化的保留





## 10.2:事务原理



### 10.2.1:介绍

InnoDB最大的特点就是支持事务

事务 是一组操作的集合，它是一个不可分割的工作单位，事务会把所有的操作作为一个整体一起向系 统提交或撤销操作请求，即这些操作要么同时成功，要么同时失败



特性

ACID

- 原子性（Atomicity）：事务是不可分割的最小操作单元，要么全部成功，要么全部失败

- 一致性（Consistency）：事务完成时，必须使所有的数据都保持一致状态

- 隔离性（Isolation）：数据库系统提供的隔离机制，保证事务在不受外部并发操作影响的独立环 境下运行

- 持久性（Durability）：事务一旦提交或回滚，它对数据库中的数据的改变就是永久的



那实际上，我们研究事务的原理，就是研究MySQL的InnoDB引擎是如何保证事务的这四大特性的



而对于这四大特性，实际上分为两个部分

其中的原子性、一致性、持久化，实际上是由InnoDB中的 两份日志来保证的，一份是redo log日志，一份是undo log日志

而隔离性是通过数据库的锁， 加上MVCC来保证的

![image-20220906102712418](.\assets\image-20220906102712418.png)



### 10.2.2:redo log

重做日志，记录的是事务提交时数据页的物理修改，是用来实现事务的持久性

事务持久性的实现就是依靠 redo log

该日志文件由两部分组成：

- 重做日志缓冲（redo log buffer）
- 重做日志文件（redo log file）
- 前者是在内存中，后者在磁盘中
- 当事务提交之后会把所有修改信息都存到该日志文件中,
- 用于在刷新脏页到磁盘,发生错误时, 进行数据恢复使用



如果没有redolog，可能会存在什么问题的？ 我们一起来分析一下



我们知道，在InnoDB引擎中的内存结构中，主要的内存区域就是缓冲池，在缓冲池中缓存了很多的数 据页。 当我们在一个事务中，执行多个增删改的操作时，InnoDB引擎会先操作缓冲池中的数据，如果 缓冲区没有对应的数据，会通过后台线程将磁盘中的数据加载出来，存放在缓冲区中，然后将缓冲池中 的数据修改，修改后的数据页我们称为脏页。 而脏页则会在一定的时机，通过后台线程刷新到磁盘 中，从而保证缓冲区与磁盘的数据一致。 而缓冲区的脏页数据并不是实时刷新的，而是一段时间之后 将缓冲区的数据刷新到磁盘中，假如刷新到磁盘的过程出错了，而提示给用户事务提交成功，而数据却没有持久化下来，这就出现问题了，没有保证事务的持久性



![image-20220906103547685](.\assets\image-20220906103547685.png)



那么，如何解决上述的问题呢？ 在InnoDB中提供了一份日志 redo log，接下来我们再来分析一 下，通过redolog如何解决这个问题

![image-20220906103733774](.\assets\image-20220906103733774.png)

注意：写入Buffer Pool和写入Redolog Buffer是原子性的

写入Buffer Pool的时候，一定会写入Redolog Buffer

这样就可以保证持久性



有了redolog之后，当对缓冲区的数据进行增删改之后，会首先将操作的数据页的变化，记录在redo log buffer中。在事务提交时，会将redo log buffer中的数据刷新到redo log磁盘文件中。 过一段时间之后，如果刷新缓冲区的脏页到磁盘时，发生错误，此时就可以借助于redo log进行数据 恢复，这样就保证了事务的持久性。 而如果脏页成功刷新到磁盘 或 或者涉及到的数据已经落盘，此时redolog就没有作用了，就可以删除了，所以存在的两个redolog文件是循环写的



那为什么每一次提交事务，要刷新redo log 到磁盘中呢，而不是直接将buffer pool变更的数据页刷新到磁盘呢



因为在业务操作中，我们操作数据一般都是随机读写磁盘的，而不是顺序读写磁盘。 而redo log在往磁盘文件中写入数据，由于是日志文件，所以都是顺序写的。顺序写的效率，要远大于随机写。 这 种先写日志的方式，称之为 WAL（Write-Ahead Logging）



WAL-先写日志



### 10.2.3:undo log

回滚日志，用于记录数据被修改前的信息 , 作用包含两个 :

- 提供回滚(保证事务的原子性) 
- MVCC(多版本并发控制) 



undo log和redo log记录物理日志不一样，它是逻辑日志。可以认为当delete一条记录时，undo log中会记录一条对应的insert记录，反之亦然，当update一条记录时，它记录一条对应相反的 update记录。当执行rollback时，就可以从undo log中的逻辑记录读取到相应的内容并进行回滚



也就是undo log记录的是能恢复到之前的状态的逻辑



Undo log销毁：undo log在事务执行时产生，事务提交时，并不会立即删除undo log，因为这些日志可能还用于MVCC



Undo log存储：undo log采用段的方式进行管理和记录，存放在前面介绍的 rollback segment 回滚段中，内部包含1024个undo log segment



### 10.2.4:MVCC



#### 10.2.4.1:基本概念



-  当前读

读取的是记录的最新版本，读取时还要保证其他并发事务不能修改当前记录，会对读取的记录进行加 锁。对于我们日常的操作

如：

- select ... lock in share mode(共享锁)
- select ... for update、update、insert、delete(排他锁)都是一种当前读



测试：

- 终端一二都开启事务

- 终端一 select * from stu
- 终端二 更新一条数据
- 终端一 select * from stu，查看不到
- 终端二提交事务
- 终端一 select * from stu，查看不到，默认是可重复读
- 不满足当前读
- 终端一 select * from stu lock in share mode
- 查看到了更新的数据
- 总结当前读就是能够立刻读取到最新的数据





- 快照读

简单的select（不加锁）就是快照读，快照读，读取的是记录数据的可见版本，有可能是历史数据， 不加锁，是非阻塞读，以下是不同隔离级别的情况

- Read Committed：每次select，都生成一个快照读
- Repeatable Read：开启事务后第一个select语句才是快照读的地方，后面select实际上就是查询的快照，不是每次都生成快照读
- Serializable：快照读会退化为当前读





- MVCC

全称 Multi-Version Concurrency Control，多版本并发控制。指维护一个数据的多个版本， 使得读写操作没有冲突，快照读为MySQL实现MVCC提供了一个非阻塞读功能。MVCC的具体实现，还需 要依赖于数据库记录中的 `三个隐式字段`、`undo log日志`、`readView`

接下来，我们再来介绍一下InnoDB引擎的表中涉及到的隐藏字段 、undolog 以及 readview，从 而来介绍一下MVCC的原理





#### 10.2.4.2:MVVC隐藏字段

![image-20220906110851990](.\assets\image-20220906110851990.png)

当我们创建了上面的这张表，我们在查看表结构的时候，就可以显式的看到这三个字段。 实际上除了 这三个字段以外，InnoDB还会自动的给我们添加三个隐藏字段及其含义分别是

![image-20220906110930000](.\assets\image-20220906110930000.png)

而上述的前两个字段是肯定会添加的， 是否添加最后一个字段DB_ROW_ID，得看当前表有没有主键， 如果有主键，则不会添加该隐藏字段



查看idb文件的指令  ibd2sdi xxx.ibd



测试：

- 查看有主键的表 stu

```sql
ibd2sdi stu.ibd
```

发现了 "name": "DB_TRX_ID"  和  "name": "DB_ROLL_PTR"字段



-  查看没有主键的表 employee
- create table employee (id int , name varchar(10))
- ibd2sdi employee.ibd

我们会看到处理我们建表时指定的字段以外，还有 额外的三个字段 分别是：DB_TRX_ID 、 DB_ROLL_PTR 、DB_ROW_ID，因为employee表是没有 指定主键的





#### 10.2.4.3:undo log

回滚日志，在insert、update、delete的时候产生的便于数据回滚的日志

当insert的时候，产生的undo log日志只在回滚时需要，在事务提交后，可被立即删除

而update或者delete的时候，产生的undo log日志不仅在回滚时需要

在快照读时也需要，不会立即被删除



undo log版本链：

有一张表原始数据为：

![image-20220906112204706](.\assets\image-20220906112204706.png)



然后，有四个并发事务同时在访问这张表



-  第一步

![image-20220906112441061](.\assets\image-20220906112441061.png)

当事务2执行第一条修改语句时，会记录undo log日志，记录数据变更之前的样子; 然后更新记录， 并且记录本次操作的事务ID，回滚指针，回滚指针用来指定如果发生回滚，回滚到哪一个版本。

![image-20220906112852930](.\assets\image-20220906112852930.png)

回滚指针就指向了undo log日志，就可以根据回滚指针的地址指向来恢复回滚数据



- 第二步

![image-20220906113027752](.\assets\image-20220906113027752.png)

当事务3执行第一条修改语句时，也会记录undo log日志，记录数据变更之前的样子; 然后更新记 录，并且记录本次操作的事务ID，回滚指针，回滚指针用来指定如果发生回滚，回滚到哪一个版本。

![image-20220906113102312](.\assets\image-20220906113102312.png)

但是第一条undo log日志不会删除，因为还有活动事务正在使用



-  第三步

![image-20220906113433029](.\assets\image-20220906113433029.png)



当事务4执行第一条修改语句时，也会记录undo log日志，记录数据变更之前的样子; 然后更新记 录，并且记录本次操作的事务ID，回滚指针，回滚指针用来指定如果发生回滚，回滚到哪一个版本。

![image-20220906113505021](.\assets\image-20220906113505021.png)



最终我们发现，不同事务或相同事务对同一条记录进行修改，会导致该记录的undolog生成一条 记录版本链表，链表的头部是最新的旧记录，链表尾部是最早的旧记录，



### 10.2.5:readview

ReadView(读视图)是快照读 SQL执行时MVCC提取数据的依据，记录并维护系统当前活跃的事务 未提交的id，RR隔离级别下，仅仅在事务第一次执行快照读的时候生成ReadView，后续复用ReadView



![image-20220906114603289](.\assets\image-20220906114603289.png)

所以RR每次读都是一样的数据，因为是同一个ReadView



ReadView中包含了四个核心字段：

![image-20220906114352152](.\assets\image-20220906114352152.png)



不同的隔离级别，生成ReadView的时机不同： 

- READ COMMITTED ：在事务中每一次执行快照读时生成ReadView
- REPEATABLE READ：仅在事务中第一次执行快照读时生成ReadView，后续复用该ReadView



![image-20220906114820723](.\assets\image-20220906114820723.png)
