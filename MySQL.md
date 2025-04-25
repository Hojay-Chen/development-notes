# 一、MySQL架构篇

## 1. 逻辑架构

### 1.1 架构总览

MySQL的逻辑架构总体分为两层，分别为Server层和引擎层，如下图所示：

<img src=".\images\mysql_architecture_opanfk.jpg" />

**Server层：**主要包括**连接器**、**查询缓存**、**分析器**、**优化器**、**执行器**等，可以说此层涵盖了MySQL的**大多数核心服务功能**，以及**所有的内置函数**（如日期、时间、数学和加密函数等），**所有跨存储引擎的功能**都在这一层实现，比如存储过程、触发器、视图等。

**存储引擎层**：存储引擎层主要是负责数据的存储和提取。其架构模式是插件式的，支持 InnoDB、MyISAM、Memory等多个存储引擎。目前最常用的存储引擎是InnoDB，也是MySQL（5.5.5版本及以后版本）默认存储引擎。也就是说我们在创建表时如果不指定表的存储引擎类型，则默认设置为InnoDB。

### 1.2 连接器

负责客户端与服务端进行连接的工作。客户端（比如：Navicat、MySQL front、JDBC、SQLyog以及各种编程语言实现的客户端连接程序等）要向MySQL发起通信都必须先跟Server端建立通信连接，而建立连接的工作就是有连接器完成的。连接过程一般如下：

客户端会先连接到数据库上，这时候首先遇到的就是连接器。连接器负责跟客户端建立连接、获取权限、维持和管理连接。

连接命令如下：

```
mysql ‐h host[数据库地址] ‐u root[用户] ‐p root[密码] ‐P 3306；
```

连接时，在完成经典的**TCP握手**后，连接器就要开始**认证客户端的身份**，通过输入的用户名和密码来认证身份：

- 如果用户名或密码不对，就会返回"Access denied for user"的错误，然后客户端程序结束执行。
- 如果用户名密码认证通过，连接器会到权限表里面查出客户端拥有的权限。之后，这个连接里面的权限判断逻辑，都将依赖于此时读到的权限。这就意味着，一个用户成功建立连接后，即使你用管理员账号对这个用户的权限做了修改，也不会影响已经存在连接的权限。修改完成后，只有再新建的连接才会使用新的权限设置。

### 1.3 查询缓存

MySQL收到一个查询请求后，会先到查询缓存里检查之前是否执行过该语句。之前执行过的语句及其结果可能会以key-value对的形式被直接缓存在内存中，key是查询的语句，value是查询的结果。

如果你的查询能够直接在这个缓存中命中，那么结果就会被直接返回给客户端。如果语句不在查询缓存中，就会继续后面的执行阶段。执行完成后，执行结果会被存入查询缓存中。

查看当前MySQL实例是否开启缓存机制，可采用命令：

```
show global variables like "%query_cache_type%”;
```

query_cache_type有3个值：

- 0(默认)：代表关闭查询缓存OFF；

- 1：代表开启ON；

- 2：（DEMAND）代表当SQL语句中有SQL_CACHE关键词时才缓存，这样对于默认的SQL语句都不使用查询缓存。而对于你确定要使用查询缓存的语句，可以用SQL_CACHE显式指定，如下：

  ```
  select SQL_CACHE * from test where id=5;
  ```

一般建议大家在静态表（即极少更新的表，如：系统配置表、字典表等）里使用查询缓存，原因如下：

> **为什么大多数情况查询缓存就是个鸡肋？**
>
> 因为缓存失效非常频繁，只要有对一个表的更新，这个表上所有的查询缓存都会被清空。因此很可能存起来的结果，还没使用，就被一个更新全清空了。对于更新频繁的数据库来说，查询缓存的命中率会非常低。

**注意**：MySQL8.0已经移除了查询缓存功能。



### 1.4 解析器

**词法分析**：你输入的是由多个字符串和空格组成的一条 SQL 语句，MySQL 需要识别出里面的字符串分别是什么，代表什么。 MySQL 从你输入的"select"这个关键字识别出来，这是一个查询语句。它也要把字符串“T”识别成“表名 T”，把字符串“ID”识别成“列 ID”。
**语法分析**：根据词法分析的结果，语法分析器（比如：Bison）会根据语法规则，判断你输入的这个 SQL 语句是否 满足 MySQL 语法 。
如果SQL语句正确，则会生成一个这样的语法树：

![mysql_architecture_xsnao](.\images\mysql_architecture_xsnao.png)

### 1.5 优化器

- 表里面有多个索引的时候，决定使用哪个索引。
- 在一个语句有多表关联（join）的时候，决定各个表的连接顺序。
- 进行MySQL自己内部的优化机制。

### 1.6 执行器

- 先判断一下客户端对这个表T有没有执行查询的权限，如果没有，就会返回没有权限的错误；如果有权限，就打开表继续执行。
- 打开表的时候，执行器就会根据表的引擎定义，去使用这个引擎提供的接口来完成SQL语句的执行。



## 2. 引擎

### 2.1 引擎介绍

#### 2.1.2 InnoDB 引擎

**版本**

- ~~MySQL从3.23.34a开始就包含InnoDB存储引擎。~~ MySQL5.5之后，默认采用InnoDB引擎。

**结构**

- 数据文件结构：
  - 表名.frm 存储表结构（MySQL8.0时，合并在表名.ibd中）
  - 表名.ibd 存储数据和索引

**特点**

- InnoDB是MySQL的**默认事务型引擎**，它被设计用来处理大量的短期(short-lived)事务。可以确保事务的完整提交(Commit)和回滚(Rollback)。
- 支持**外键**功能，除了支持表级锁，还支持行级锁。

**使用**

- 除了增加和查询外，还需要更新、删除操作，那么应优先选择InnoDB存储引擎。


- 除非有非常特别的原因需要使用其他的存储引擎，否则应该优先考虑InnoDB引擎。


- InnoDB是为处理巨大数据量的最大性能设计。

**对比**

- 对比MyISAM的存储引擎，InnoDB写的处理效率差一些 ，并且会占用更多的磁盘空间以保存数据和索引。

- MyISAM只缓存索引，不缓存真实数据；InnoDB不仅缓存索引还要缓存真实数据， 对内存要求较高，而且内存大小对性能有决定性的影响。

#### 2.1.2 MyISAM 引擎

**版本**

- 5.5之前默认的存储引擎。

**特点**

- 主要的非事务处理存储引擎。

- MyISAM提供了大量的特性，包括全文索引、压缩、空间函数(GIS)等，但MyISAM **不支持事务、行级锁、外键**，只支持表级锁，且有一个毫无疑问的缺陷就是**崩溃后无法安全恢复** 。
- 优势是访问的速度快 ，对事务完整性没有要求或者以SELECT、INSERT为主的应用。
- 针对数据统计有额外的常数存储。故而 count(*) 的查询效率很高。

**结构**

- 数据文件结构：
  - 表名.frm 存储表结构
  - 表名.MYD 存储数据 (MYData)
  - 表名.MYI 存储索引 (MYIndex)

**使用**

- **适用于只读应用或者以读和插入为主的业务**，如作为数据仓库。



#### 2.1.3 Memory 引擎

**特点**

- Memory采用的逻辑介质是内存 ， 响应速度很快 ，但是当mysql的守护进程崩溃的时候数据会丢失 。

- 要求存储的数据是**数据长度不变的格式**，比如，Blob和Text类型的数据不可用(长度不固定的)。

- Memory同时支持**哈希索引** 和 **B+树索引** 。
- Memory表至少比MyISAM表要快一个数量级 。

- MEMORY**表的大小是受到限制的**。表的大小主要取决于两个参数，分别是max_rows和 max_heap_table_size。其中，max_rows可以在创建表时指定；max_heap_table_size的大小默认为16MB，可以按需要进行扩大。
- 缺点：其数据易丢失，生命周期短。基于这个缺陷，选择MEMORY存储引擎时需要特别小心。

**结构**

- 数据文件与索引文件分开存储。

**使用**

- 目标数据比较小 ，而且非常频繁的进行访问 ，在内存中存放数据，如果太大的数据会造成内存溢出 。可以通过参数max_heap_table_size 控制Memory表的大小，限制Memory表的最大的大小。
- 如果数据是临时的 ，而且**必须立即可用**得到，那么就可以放在内存中。

```
| 1 | record one |
| 2 | record two |
+---+------------+
2 rows in set (0.00 sec)
"1"，"record one"
"2"，"record two"存储在Memory表中的数据如果突然间丢失的话也没有太大的关系 。
```


#### ~~2.1.4 Archive 引擎~~

- ~~用于数据存档~~

- ~~archive是归档的意思，仅仅支持插入和查询 两种功能(行被插入后不能再修改)。~~

- ~~在MySQL5.5以后 支持索引 功能。~~


- ~~拥有很好的压缩机制，使用zlib压缩库，在记录请求的时候实时的进行压缩，经常被用来作为仓库使用。~~

- ~~创建ARCHIVE表时，存储引擎会创建名称以表名开头的文件。数据文件的扩展名为**.ARZ**。~~

- ~~根据英文的测试结论来看，同样数据量下，Archive表比MyISAM表要小大约75%，比支持事务处理的InnoDB表小大约83%。~~

- ~~ARCHIVE存储引擎采用了 行级锁。该ARCHIVE引擎支持 AUTO_INCREMENT列属性。AUTO_INCREMENT列可以~~

- ~~具有唯一索引或非唯一索引。尝试在任何其他列上创建索引会导致错误。~~

- ~~Archive表适合日志和数据采集(档案)类应用;适合存储大量的独立的作为历史记录的数据。拥有很高的插入速度，但是对查询的支持较差。~~

- ~~下表展示了ARCHIVE 存储引擎功能~~



#### ~~2.1.5 Blackhole 引擎~~

- ~~丢弃写操作，读操作会返回空内容~~

- ~~Blackhole引擎没有实现任何存储机制，它会丢弃所有插入的数据不做任何保存。~~
- ~~但服务器会记录Blackhole表的日志，所以可以用于复制数据到备库，或者简单地记录到日志。但这种应用方式会碰到很多问题，因此并不推荐。~~



#### ~~2.1.6 CSV 引擎~~

- ~~存储数据时，以逗号分隔各个数据项~~

- ~~CSV引擎可以将普通的CSV文件作为MySQL的表来处理，但不支持索引。~~

- ~~CSV引擎可以作为一种数据交换的机制，非常有用。~~

- ~~CSV存储的数据直接可以在操作系统里，用文本编辑器，或者excel读取。~~

- ~~对于数据的快速导入、导出是有明显优势的。~~

- ~~创建CSV表时，服务器会创建一个纯文本数据文件，其名称以表名开头并带有.CSV扩展名。当你将数据存储到表中时，存储引擎将其以逗号分隔值格式保存到数据文件中。~~


~~案例如下~~

```
mysql> CREATE TABLE test (i INT NOT NULL， c CHAR(10) NOT NULL) ENGINE = CSV;
Query OK， 0 rows affected (0.06 sec)

mysql> INSERT INTO test VALUES(1，'record one')，(2，'record two');
Query OK， 2 rows affected (0.05 sec)
Records: 2 Duplicates: 0  Warnings: 0

mysql> SELECT * FROM test;
+---+------------+
| i | c     |
+---+------------+
| 1 | record one |
| 2 | record two |
+---+------------+
2 rows in set (0.00 sec)
```

~~创建CSV表还会创建相应的 元文件 ，用于存储表的状态和 表中存在的行数。此文件的名称与表的名称相同，后缀为CSM 。如图所示~~

~~如果检查test.CSV通过执行上述语句创建的数据库目录中的文件，其内容使用Notepad++打开如下：~~

```
"1"，"record one"
"2"，"record two"
```

~~这种格式可以被Microsoft Excel 等电子表格应用程序读取，甚至写入。使用Microsoft Excel打开如图所示~~



#### ~~2.1.7 Federated 引擎~~

~~Federated引擎是访问其他MySQL服务器的一个 代理，尽管该引擎看起来提供了一种很好的 跨服务器的灵活性，但也经常带来问题，因此 默认是禁用的 。~~



#### ~~2.1.8 Merge引擎~~

~~管理多个MyISAM表构成的表集合~~



#### ~~2.1.9 NDB引擎~~

~~MySQL集群专用存储引擎~~

~~也叫做 NDB Cluster 存储引擎，主要用于MySQL Cluster 分布式集群环境，类似于 Oracle 的 RAC 集群。~~



#### 2.1.10 引擎对比

MySQL中同一个数据库，不同的表可以选择不同的存储引擎。如下表对常用存储引擎做出了对比。

| 特点           | MyISAM                                                     | InnoDB                                                       | MEMORY | MERGE | NDB  |
| -------------- | ---------------------------------------------------------- | ------------------------------------------------------------ | ------ | ----- | ---- |
| 存储限制       | 有                                                         | 64TB                                                         | 有     | 没有  | 有   |
| 事务           |                                                            | 支持                                                         |        |       |      |
| 锁机制         | `表锁`，即使操作一条记录也会锁住整个表，不适合高并发的操作 | `行锁`，操作时只锁某一行，不对其它行有影响，适合高并发的操作 | 表锁   | 表锁  | 行锁 |
| B树索引        | 支持                                                       | 支持                                                         | 支持   | 支持  | 支持 |
| 哈希索引       |                                                            |                                                              | 支持   |       | 支持 |
| 全文索引       | 支持                                                       |                                                              |        |       |      |
| 集群索引       |                                                            | 支持                                                         |        |       |      |
| 数据缓存       |                                                            | 支持                                                         | 支持   |       | 支持 |
| 索引缓存       | 只缓存索引，不缓存真实数据                                 | 不仅缓存索引还要缓存真实数据，对内存要求较高，而且内存大小对性能有决定性的影响 | 支持   | 支持  | 支持 |
| 数据可压缩     | 支持                                                       |                                                              |        |       |      |
| 空间使用       | 低                                                         | 高                                                           | N/A    | 低    | 低   |
| 内存使用       | 低                                                         | 高                                                           | 中等   | 低    | 高   |
| 批量插入的速度 | 高                                                         | 低                                                           | 高     | 高    | 高   |
| 支持外键       |                                                            | 支持                                                         |        |       |      |



### 2.2 引擎操作

#### 2.2.1 查看存储引擎

```
show engines;
```

#### 2.2.2 查看默认的存储引擎

```
show variables like '%storage_engine%';
#或
SELECT @@default_storage_engine;
```

#### 2.2.3 修改默认的存储引擎

命令行修改：

```
SET DEFAULT_STORAGE_ENGINE=MyISAM;
```

修改`my.cnf` 文件：

```
default-storage-engine=MyISAM
# 重启服务
systemctl restart mysqld.service
```



# 二、MySQL调优篇

## 1. 索引介绍

### 1.1 概述

​	索引（Index）是帮助MySQL高效获取数据的数据结构。

### 1.2 优点

（1）提高数据检索的效率，降低数据库的IO成本 ，这也是创建索引最主要的原因。

（2）通过创建唯一索引，可以保证数据库表中每一行数据的唯一性 。

（3）在实现数据的参考完整性方面，可以加速表和表之间的连接。换句话说，对于有依赖关系的子表和父表联合查询时，可以提高查询速度。

（4）在使用分组和排序子句进行数据查询时，可以显著减少查询中分组和排序的时间 ，降低了CPU的消耗。

### 1.3 缺点

（1）创建索引和维护索引要耗费时间 ，并且随着数据量的增加，所耗费的时间也会增加。

（2）索引需要占磁盘空间 ，除了数据表占数据空间之外，每一个索引还要占一定的物理空间，存储在磁盘上 ，如果有大量的索引，索引文件就可能比数据文件更快达到最大文件尺寸。

（3）虽然索引大大提高了查询速度，同时却会降低更新表的速度 。当对表中的数据进行增加、删除和修改的时候，索引也要动态地维护，这样就降低了数据的维护速度。

## 2. 索引分类

### 2.1 按索引性质分类

#### 2.1.1 普通索引

​	在创建普通索引时，不附加任何限制条件，只是用于提高查询效率。**这类索引可以创建在任何数据类型中，其值是否唯一和非空，要由字段本身的完整性约束条件决定**。建立索以后，可以通过索引进行查询。·

#### 2.1.2 唯一性索引

​	使用`UNIQUE`参数可以设置索引为唯一性索引，在创建唯一性索引时，**限制该索引的值必须是唯一的，但允许有 空值。在一张数据表里可以有多个唯一索引**。

#### 2.1.3 主键索引

​	**主键索引就是一种特殊的唯一性索引**，在唯一索引的基础上增加了**不为空**的约束，也就是`NOT NULL`+`UNIQUE`，一张表里最多只有一个主键索引。

​	这是由主键索引的物理实现方式决定的，因为数据存储在文件中只能按照一种顺序进行存储。

#### 2.1.4 单列索引

​	在表中的单个字段上创建索引。单列索引只根据该字段进行索引。单列索引可以是普通索引，也可以是唯一性索 引，还可以是全文索引。只要保证该索引只对应一个字段即可。**一个表可以有多个单列索引**。

#### 2.1.5 联合索引

​	多列索引是在表的`多个字段组合`上创建一个索引。该索引指向创建时对应的多个字段，可以通过这几个字段进行 查询，但是只有查询条件中使用了这些字段中的第一个字段时才会被使用。

​	例如，在表中的字段`id`、`name`和`gender`上建立一个多列索引`idx_id_name_gender`，只有在查询条件中使用了字段`id`时该索引才会被使用。使用组合索引时遵循**最左前缀原则**。

> **最左前缀匹配原则**
>
> ​	最左前缀匹配原则指的是在使用联合索引时， 查询条件必须从的最左侧开始匹配。如果一个联合索引包含多个列，查询条件必须包含第一个列的条件，然后是第二个列，以此类推。
> ​	**底层原理**:因为联合索引在B+树中的排列方式遵循”从左到右”的顺序，只有在左侧索引值相等时才会用下一个索引值进行排列，这就导致了左侧索引值不相等的所有数据的右侧索引值是无序的，例如联合引(first_ name， last_ name， age)按照(first_ name， last_ name， age)的顺在B+树中进行排序。MySQL在查找时会优先使用first_ name作为匹配依据，然后依次使用last_ name 和age 。因此，组合索引能够从左到右依次高效匹配，跳过最左侧字段会导致无法利用该索引。

#### 2.1.6 全文检索

​	全文索引（也称全文检索）是目前`搜索引擎`使用的一种关键技术。它能够利用**分词技术**等多种算法智能分析出文本文字中关键词的频率和重要性，然后按照一定的算法规则智能地筛选出我们想要的搜索结果。全文索引非常适合大型数据集，对于小的数据集，它的用处比较小。

​	使用参数`FULL TEXT`可以设置索引为全文索引。在定义索引的列上支持值的全文查找，允许在这些索引列中插入重复值和空值。全文索引只能创建在`CHAR`、`VARCHAR`或`TEXT`类型及其系列类型的字段上，**查询数据量较大的字符串类型的字段时，使用全文索引可以提高查询速度**。

​	全文索引典型的有两种类型：自然语言的全文索引和布尔全文索引。

​	自然语言搜索擎将计算每一个文档对象和查询的相关度。这里，相关度是基于匹配的关键词的个数，以及关键词在文档中出现的次数。**在整个索引中出现次数越少的词语，匹配时的相关度就越高。**相反，非常常见的单词将不会被搜索，如果一个词语在超过50%的记录中都出现了，那么自然语言的搜索将不会搜索这类词语。

​	MySQL数据库从3.23.23版开始支持全文索引，但**MySQL5.6.4以前只有Myisam支持，5.6.4版本以后innodb才支** **持**，**但是官方版本不支持中文分词，需要第三方分词插件**。在5.7.6版本，MySQL内置了`ngram全文解析器`，用来支持亚洲语种的分词。

​	随着大数据时代的到来，关系型数据库应对全文索引的需求已力不从心，逐渐被`solr`、`ElasticSearch`等专门 的搜索引擎所替代。

#### 2.1.7 空间索引

​	使用参数`SPATIAL`可以设置索引为`空间索引`。空间索引只能建立在空间数据类型上，这样可以提高系统获取空间 数据的效率。MySQL中的空间数据类型包括`GEOMETRY`、`POINT`、`LINESTRING`和`POLYGON`等。目前只有MyISAM 存储引擎支持空间检索，而且索引的字段不能为空值。对于初学者来说，这类索引很少会用到。



### 2.2 按数据结构分类

#### 2.2.1 B+树索引

B+树中，每一个记录（包括数据记录和目录项记录）结构如下：

![mysql_btree_xapnf](.\images\mysql_btree_xapnf.png)


我们只在示意图里展示记录的这几个部分:

- record_type ：记录头信息的一项属性，表示记录的类型， 0 表示普通记录、 2 表示最小记录、 3 表示最大记录、 1 暂时还没用过，下面讲。
- next_record：记录头信息的一项属性，表示下一条地址相对于本条记录的地址偏移量，我们用箭头来表明下一条记录是谁。
- 各个列的值 ：这里只记录在 index_demo 表中的三个列，分别是 c1 、 c2 和 c3 。
- 其他信息 ：除了上述3种信息以外的所有信息，包括其他隐藏列的值以及记录的额外信息。

将记录格式示意图的其他信息项暂时去掉并把它竖起来的效果就是这样：

<img src=".\images\mysql_btree_cnsla.png" alt="mysql_btree_cnsla" style="zoom:80%;" />

把一些记录放到页里的示意图就是：

<img src=".\images\mysql_btree_csanklv.png" alt="mysql_btree_csanklv" style="zoom: 50%;" />

**①一个简单的索引设计方案**
我们在根据某个搜索条件查找一些记录时为什么要遍历所有的数据页呢？因为各个页中的记录并没有规律，我们并不知道我们的搜索条件匹配哪些页中的记录，所以不得不依次遍历所有的数据页。所以如果我们想快速的定位到需要查找的记录在哪些数据页中该咋办？我们可以为快速定位记录所在的数据页而建立一个目录 ，建这个目录必须完成下边这些事：

- 下一个数据页中数据项记录的主键值必须大于上一个页中数据项记录的主键值。

- 给所有的页建立一个目录项。

所以我们为上边几个页做好的目录就像这样子：

![mysql_btree_ndslvd](.\images\mysql_btree_ndslvd.png)

以 页28 为例，它对应 目录项2 ，这个目录项中包含着该页的页号 28 以及该页中数据项记录的最小主键值 5 。我们只需要把几个目录项在物理存储器上连续存储（比如：数组），就可以实现根据主键值快速查找某条记录的功能了。比如：查找主键值为 20 的记录，具体查找过程分两步：

- 先从目录项中根据 二分法 快速确定出主键值为 20 的记录在 目录项3 中（因为 12 < 20 < 209 ），它对应的页是 页9 。

- 再根据前边说的在页中查找记录的方式去 页9 中定位具体的记录。

至此，针对数据页做的简易目录就搞定了。这个目录有一个别名，称为 索引 。

**②InnoDB中的索引方案**
**迭代1次：目录项纪录的页**

我们把前边使用到的目录项放到数据页中的样子就是这样：

![mysql_btree_knclads](.\images\mysql_btree_knclads.png)

从图中可以看出来，我们新分配了一个编号为30的页来专门存储目录项记录。

这里再次强调`目录项记录`和`普通的数据项记录`的不同点：

- `目录项记录`的 record_type 值是1，而`数据项记录`的 record_type 值是0。
- 目录项记录只有主键值和页的编号两个列，而普通的数据项记录的列是用户自己定义的，可能包含很多列 ，另外还有InnoDB自己添加的隐藏列。

相同点：两者用的是一样的数据页，都会为主键值生成**Page Directory（页目录）**，从而在按照键值进行查找时可以使用二分法来加快查询速度。

现在以查找主键为 20 的记录为例，根据某个主键值去查找记录的步骤就可以大致拆分成下边两步：

- 先到存储目录项记录的页，也就是页30中通过 二分法 快速定位到对应目录项，因为 12 < 20 < 209 ，所以定位到对应的记录所在的页就是页9。
- 再到存数据项记录的页9中根据二分法快速定位到主键值为 20 的数据项记录。

**迭代2次：多个目录项纪录的页**

​	虽然说 目录项记录 中只存储主键值和对应的页号，比数据项记录需要的存储空间小多了，但是不论怎么说一个页只
​	有16KB大小，能存放的目录项记录也是有限的，那如果表中的数据太多，以至于一个数据页不足以存放所有的
目录项记录，如何处理呢?

​	这里我们假设一个存储目录项记录的页最多只能存放4条目录项记录，所以如果此时我们再向上图中插入一条主键值为320的数据项记录的话，那就需要分配一个新的存储目录项记录的页:

![mysql_btree_mkldvnc](.\images\mysql_btree_mkldvnc.png)


​	从图中可以看出，我们插入了一条主键值为320的数据项记录之后需要两个新的数据页：

​	为存储该数据项记录而新生成了 页31 。
​	因为原先存储目录项记录的 页30的容量已满 （我们前边假设只能存储4条目录项记录)，所以不得不需要一个新的页32 来存放 页31 对应的目录项。
​	现在因为存储目录项记录的页不止一个，所以如果我们想根据主键值查找一条数据项记录大致需要3个步骤，以查找主键值为 20 的记录为例：

- 确定 目录项记录页

  我们现在的存储目录项记录的页有两个，即 页30和页32 ，又因为页30表示的目录项的主键值的范围是 [1， 320)，页32表示的目录项的主键值不小于 320，所以主键值为 20 的记录对应的目录项记录在 页30 中。

- 通过目录项记录页 确定数据项记录真实所在的页 。

  在一个存储 目录项记录 的页中通过主键值定位一条目录项记录的方式说过了。

- 在真实存储数据项记录的页中定位到具体的记录。

**迭代3次：目录项记录页的目录页**

​	问题来了，在这个查询步骤的第1步中我们需要定位存储目录项记录的页，但是这些页是不连续的，如果我们表中的数据非常多则会产生很多存储目录项记录的页，那我们怎么根据主键值快速定位一个存储目录项记录的页呢?那就为这些存储目录项记录的页再生成一个更高级的目录，就像是一个多级目录一样，大目录里嵌套小目录，小目录里才是实际的数据，所以现在各个页的示意图就是这样子:

​	![mysql_btree_cndksfs](.\images\mysql_btree_cndksfs.png)

​	如图，我们生成了一个存储更高级目录项的页33 ，这个页中的两条记录分别代表页30和页32，如果数据项记录的主键值在 [1， 320)之间，则到页30中查找更详细的目录项记录，如果主键值不小于320 的话，就到页32中查找更详细的目录项记录。

​	我们可以用下边这个图来描述它：

![mysql_btree_csanivd](.\images\mysql_btree_csanivd.png)

​	这个数据结构，它的名称是 B+树 。

**B+Tree索引**

​	不论是存放数据项记录的数据页，还是存放目录项记录的数据页，我们都把它们存放到B+树这个数据结构中了，所以我们也称这些数据页为节点。从图中可以看出，我们的实际数据项记录其实都存放在B+树的最底层的节点上，这些节点也被称为叶子节点，其余用来存放目录项的节点称为非叶子节点或者内节点，其中B+树最上边的那个节点也称为根节点。

​	一个B+树的节点其实可以分成好多层，规定最下边的那层，也就是存放我们数据项记录的那层为第 0层，之后依次往上加。之前我们做了一个非常极端的假设：存放数据项记录的页最多存放3条记录 ，存放目录项记录的页最多存放4条记录 。其实真实环境中一个页存放的记录数量是非常大的，假设所有存放数据项记录的叶子节点代表的数据页可以存放100条数据项记录，所有存放目录项记录的内节点代表的数据页可以存放1000条目录项记录 ，那么：

- 如果B+树只有1层，也就是只有1个用于存放数据项记录的节点，最多能存放 100 条记录。

- 如果B+树有2层，最多能存放 1000×100=10，0000 条记录。
- 如果B+树有3层，最多能存放 1000×1000×100=1，0000，0000 条记录。
- 如果B+树有4层，最多能存放 1000×1000×1000×100=1000，0000，0000 条记录。相当多的记录！！！

​	你的表里能存放 100000000000 条记录吗？所以一般情况下，我们用到的B+树都不会超过4层 ，那我们通过主键值去查找某条记录最多只需要做4个页面内的查找（查找3个目录项页和一个数据项记录页），又因为在每个页面内有所谓的 **Page Directory（页目录）**，所以在页面内也可以通过`二分法`实现快速定位记录。


#### 2.2.2 哈希索引



#### 2.2.3 倒排索引（全文索引Full-Text）



#### 2.2.4 R-树索引（多维空间索引）



### 2.3 按物理实现方法分类

#### 2.3.1 聚簇索引

聚簇索引并不是一种单独的索引类型，而是一种数据存储方式，要求所有的用户记录都存储在叶子节点，这就使得索引和数据存储在一个B+树当中，也就是所谓的索引即数据，数据即索引。

**特点**

- 使用记录主键值的大小进行记录和页的排序，这包括三个方面的含义：

  - 页内的记录是按照主键的大小顺序排成一个单向链表。
  - 各个存放 用户记录的页也是根据页中用户记录的主键大小顺序排成一个双向链表 。
  - 存放目录项记录的页分为不同的层次，在同一层次中的页也是根据页中目录项记录的主键大小顺序排成一个 双向链表。
- B+树的叶子节点存储的是完整的用户记录。
- 所谓完整的用户记录，就是指这个记录中存储了所有列的值（包括隐藏列）。

**优点**

- 数据访问更快，因为聚簇索引将索引和数据保存在同一个B+树中，因此从聚簇索引中获取数据比非聚簇索引更快。

- 聚簇索引对于主键的**排序查找**和**范围查找**速度非常快。
- 按照聚簇索引排列顺序，查询显示一定范围数据的时候，由于数据都是紧密相连，数据库不用从多个数据块中提取数据，所以节省了大量的io操作。

**缺点**

- 插入速度严重依赖于插入顺序 ，按照主键的顺序插入是最快的方式，否则将会出现页分裂，严重影响性能。因此，对于InnoDB表，我们一般都会定义一个自增的ID列为主键。
- 更新主键的代价很高 ，因为将会导致被更新的行移动。因此，对于InnoDB表，我们一般定义主键为不可更新。
- 二级索引访问需要两次索引查找 ，第一次找到主键值，第二次根据主键值找到行数据。

**限制**

- 对于MySQL数据库目前只有InnoDB数据引擎支持聚簇索引，而MylSAM并不支持聚簇索弘。

- 由于数据物理存储排序方式只能有一种，所以每个MySQL的表只能有一个聚簇索引。一般情况下就是该表的主键。

- 如果没有定义主键，Innodb会选择非空的唯一索引代替。如果没有这样的索引，Innodb会隐式的定义一个主键来作为聚簇索引。

- 为了充分利用聚簇索引的聚簇的特性，所以innodb表的主键列尽量选用有序的顺序id，而不建议用无序的id，
  比如UUID、MD5、HASH、字符串列作为主键无法保证数据的顺序增长。



#### 2.3.2 非聚簇索引

![mysql_btree_xanoad](.\images\mysql_btree_xanoad.png)

**概念**

**非聚簇索引**（二级索引、辅助索引）：按照某列建立的B+树，且需要一次回表操作才可以定位到完整的用户记录的B+树叫二级索引。由于我们使用的是c2列的大小作为B+树的排序规则，所以我们也称这个B+树是为c2列建立的索引。

**特点**

- 使用记录c2列的大小进行记录和页的排序，这包括三个方面的含义:

  - 页内的记录是按照c2列的大小顺序排成一个单向链表。

  - 各个存放用户记录的页 也是根据页中记录的c2列大小顺序排成一个 双向链表。

  - 存放目录项记录的页分为不同的层次，在同一层次中的页也是根据页中目录项记录的c2列大小顺序排成一个双向链表。

- B+树的叶子节点存储的并不是完整的用户记录，而只是c2列和数据项记录（这里数据项记录可能是用户记录，也可能是）。

**使用**

所以如果想通过c2列的值查找某些记录的话就可以使用刚刚建好的这个B+树了。查找过程如下:

> 以查找c2列的值为4的记录为例

- 确定目录项记录页

  > 根据根页面，也就是页44，可以快速定位到目录项记录所在的页为页42(因为2<4<9)。

- 通过目录项记录页确定用户记录真实所在的页。

  > 在页42中可以快速定位到实际存储用户记录的页，但是由于c2列并没有唯一性约束，所以c2列值为4的记录可能分布在多个数据页中，又因为2<4≤4，所以确定实际存储用户记录的页在页34和页35中。

- 在真实存储用户记录的页中定位到具体的记录。

  > 到页34和页35中定位到具体的记录。

- 但是这个B+树的叶子节点中的记录只存储了c2和c1(也就是主键)两个列，所以我们必须再根据获取到的主键值（Innodb）或数据文件偏移量（MyISAM）进行一次回表。


> **回表**
>
> 根据c2列的值在以该列建立的非聚簇索引只能确定我们要查找记录的主键值，所以如果我们想根据c2列的值查找到完整的用户记录的话，仍然需要到聚簇索引中再查一遍，这个过程称为回表 。也就是根据c2列的值查询一条完整的用户记录需要使用到 2 棵B+树！
>
> 问题：为什么我们还需要一次`回表`操作呢？直接把完整的用户记录放到叶子节点不OK吗？
>
> 如果每一个都完整存放用户记录，那倘若100w的用户数据，每个都要完整记录，那不是有几个二级索引，就需要翻几倍去储存，加大了储存空间的开销

**对比**

聚簇索引与非聚簇索引的原理不同，在使用上也有一些区别:

- 聚簇索引的叶子节点存储的就是我们的数据记录，非聚簇索引的叶子节点存储的是数据位置。非聚簇索引不会影响数据表的物理存储顺序。

- 一个表只能有一个聚簇索引，因为只能有一种排序存储的方式，但可以有多个非聚簇索引，也就是多个索引目录提供数据检索。

- 使用聚簇索引的时候，数据的**查询效率高**，但如果对数据进行插入，删除，更新等操作，效率会比非聚簇索引低。



## 3. 引擎对索引支持情况

### 3.1 InnoDB

​	支持 B-tree、Full-text 等索引，不支持 Hash 索引；

### 3.2 MyISAM

​	支持 B-tree、Full-text 等索引，不支持 Hash 索引；

### 3.3 Memory

​	支持 B-tree、Hash 等 索引，不支持 Full-text 索引；

### ~~3.4 NDB~~

​	~~支持 Hash 索引，不支持 B-tree、Full-text 等索引；~~

### ~~3.5 Archive~~

​	~~不支持 B-tree、Hash、Full-text 等索引；~~



## 4. 常用引擎的索引结构

### 4.1 Innodb

Innodb中，**主键**构建的索引结合了**B+树索引**和**聚簇索引**，该结构将索引和数据聚合在一起，B+树的子节点存储用户记录，非子节点存储目录项记录。

而**非主键列**的索引则均为**非聚簇索引**，即索引结构只把主键值作为数据记录，通过该索引结构来查找出目标用户记录的主键值，再通过主键值进行回表找到目标用户记录。

### 4.2 MyISAM

MyISAM引擎使用 B+Tree 作为索引结构，叶子节点的data域存放的是数据记录的地址 。

我们知道InnoDB中索引即数据，也就是聚簇索引的那棵B+树的叶子节点中已经把所有完整的用户记录都包含了，而MyISAM的索引方案虽然也使用树形结构，但是却将索引和数据分开存储：

- 将表中的记录 按照记录的插入顺序单独存储在一个文件中，称之为**数据文件**。这个文件并不划分为若干个数据页，有多少记录就往这个文件中塞多少记录就成了。由于在插入数据的时候并没有**刻意按照主键大小排序，**所以我们并不能在这些数据上使用二分法进行查找。

- 使用MyISAM存储引擎的表会把索引信息另外存储到一个称为**索引文件**的另一个文件中。MyISAM会单独为表的主键创建一个索引，只不过在索引的**叶子节点中存储的不是完整的用户记录**，而是主键值 +数据记录地址的组合。

<img src=".\images\mysql_myisam_ncksan.png" alt="mysql_myisam_ncksan" style="zoom: 25%;" />

这里设表一共有三列，假设我们以C1为主键，上图是一个MylSAM表的主索引(Primary key)示意。可以看出MylSAM的索引文件仅仅保存数据记录的地址。**在MylSAM中，主键索引和二级索引(Secondary key)在结构上没有任何区别**，只是主键索引要求key是唯一的，而二级索引的key可以重复。

如果我们在C2上建立一个二级索引，则此索引的结构如下图所示：

<img src=".\images\mysql_myisam_xsanklos.png" alt="mysql_myisam_xsanklos" style="zoom: 25%;" />

同样也是一棵B+Tree，data域保存数据记录的地址。因此，MylSAM中索引检索的算法为：首先按照B+Tree搜索算法搜索索引，如果指定的Key存在，则取出其data域的值，然后以data域的值为地址，读取相应用户记录。

### 4.3 Innodb和MyISAM对比

- MyISAM的索引方式都是“非聚簇”的，与InnoDB包含1个聚簇索引是不同的。
- 在InnoDB存储引擎中，我们只需要根据主键值对 聚簇索引 进行一次查找就能找到对应的记录，而在MyISAM 中却需要进行一次 回表（找到地址后根据地址去找表这个操作叫回表） 操作，意味着MyISAM中建立的索引相当于全部都是 二级索引 。
- InnoDB的数据文件本身就是索引文件是 一起的，而MyISAM索引文件和数据文件是 分离的 ，索引文件仅保存数据记录的地址。
- InnoDB的非聚簇索引data域存储相应记录 主键的值 ，而MyISAM索引记录的是 地址。换句话说，InnoDB的所有非聚簇索引都引用主键作为data域。
- MyISAM的回表操作是十分快速的，因为是拿着地址偏移量直接到文件中取数据的，反观InnoDB是通过获取主键之后再去聚簇索引里找记录，虽然说也不慢，但还是比不上直接用地址去访问。
- InnoDB要求表 必须有主键 （ MyISAM可以没有 ）。如果没有显式指定，则MySQL系统会自动选择一个可以非空且唯一标识数据记录的列作为主键。如果不存在这种列，则MySQL自动为InnoDB表生成一个隐含字段作为主键，这个字段长度为6个字节，类型为长整型。



## 5. 数据库调优

### 5.1 调优分析

当我们遇到数据库调优问题的时候，该如何思考呢？这里把思考的流程整理成下面这张图。

整个流程划分成了 `观察（Show status）` 和 `行动（Action）` 两个部分。字母 S 的部分代表观察（会使 用相应的分析工具），字母 A 代表的部分是行动（对应分析可以采取的行动）。

![mysql_better_xmasnon](E:\各种资料\Java开发笔记\我的笔记\images\mysql_better_xmasnon.png)

![mysql_better_xapanf](E:\各种资料\Java开发笔记\我的笔记\images\mysql_better_xapanf.png)

**详细解释一下这张图：**

- 在S1部分，我们需要**观察服务器的状态是否存在周期性的波动**。

  - 如果存在周期性波动，有可能是周期性节点的原因，比如双十一、促销活动等。这样的话，我们可以通过A1这一步骤解决，也就是**加缓存**，或者**更改缓存失效策略**。

  - 如果缓存策略没有解决，或者不是周期性波动的原因，我们就需要进一步分析查询延迟和卡顿的原因。接下来进入S2这一步，我们需要**开启慢查询**。慢查询可以帮我们定位执行慢的SQL语句。我们可以通过设置`long_query_time`参数定义“慢”的阈值，如果SQL执行时间超过了`long_query_time，`则会认为是慢查询。当收集上来这些慢查询之后，我们就可以通过分析工具对慢查询日志进行分析。

  - 接着S3本步骤可以针对性地**用`EXPLAIN`查看对应SQL语句的执行计划**，或者**使用`show profile`**查看SQL中每一个步骤的时间成本。这样我们就可以了解SQL查询慢是因为执行时间长，还是等待时间长。

    - 如果是**SQL等待时间长**，我们进入A2步骤。在这一步骤中，我们可以**调优服务器的参数**，比如适当增加数据库缓冲池等。
    - 如果是**SQL执行时间长**，就进入A3步骤，这一步中我们需要考虑是**索引设计的问题**？还是**查询关联的数据表过多**？还是因为**数据表的字段设计问题**导致了这一现象。然后在这些维度上进行对应的调整。

    - 如果A2和A3都不能解决问题，我们需要考虑数据库自身的**SQL查询性能是否已经达到了瓶颈**。
      - 如果确认没有达到性能瓶颈，就需要重新检查，重复以上的步骤。
      - 如果已经达到了`性能瓶颈`，进入A4阶段，需要考虑**增加服务器**，**采用读写分离的架构，**或者考虑**对数据库进行分库分表**，比如垂直分库、垂直分表和水平分表等。

以上就是数据库调优的流程思路。如果我们发现执行$QL时存在不规则延迟或卡顿的时候，就可以采用分析工具 帮我们定位有问题的SQL，这三种分析工具你可以理解是SQL调优的三个步骤：`慢查询`、`EXPLAIN`和`SHOW PROFILING`.

![mysql_better_apkfs](E:\各种资料\Java开发笔记\我的笔记\images\mysql_better_apkfs.png)



### 5.2 查看系统性能参数

在MySQL中，可以使用 `SHOW STATUS` 语句查询一些MySQL数据库服务器的`性能参数、执行频率`。

SHOW STATUS语句语法如下：

```
SHOW [GLOBAL|SESSION] STATUS LIKE '参数';
```

> **`GLOBAL` 和 `SESSION` 的区别**
>
> 1. **`GLOBAL`**：
>    - **作用范围**：全局范围，适用于整个 MySQL 服务器。
>    - **设置方式**：通过 `SET GLOBAL` 或 `SET @@global` 设置。
>    - **查询方式**：通过 `SHOW GLOBAL STATUS` 查询。
>    - **生效范围**：对所有新创建的会话有效，但对当前已存在的会话无效。
> 2. **`SESSION`**：
>    - **作用范围**：会话范围，仅适用于当前会话。
>    - **设置方式**：通过 `SET SESSION` 或 `SET @@session` 设置。
>    - **查询方式**：通过 `SHOW SESSION STATUS` 查询。
>    - **生效范围**：仅对当前会话有效，不会影响其他会话。

一些常用的性能参数如下：

- Connections：连接MySQL服务器的次数。
- Uptime：MySQL服务器的上线时间。
- Slow_queries：慢查询的次数。
- Innodb_rows_read：Select查询返回的行数
- Innodb_rows_inserted：执行INSERT操作插入的行数
- Innodb_rows_updated：执行UPDATE操作更新的 行数
- Innodb_rows_deleted：执行DELETE操作删除的行数
- Com_select：查询操作的次数。
- Com_insert：插入操作的次数。对于批量插入的 INSERT 操作，只累加一次。
- Com_update：更新操作 的次数。
- Com_delete：删除操作的次数。

若查询MySQL服务器的连接次数，则可以执行如下语句:

```
SHOW STATUS LIKE 'Connections';
```

若查询服务器工作时间，则可以执行如下语句:

```
SHOW STATUS LIKE 'Uptime';
```

若查询MySQL服务器的慢查询次数，则可以执行如下语句:

```
SHOW STATUS LIKE 'Slow_queries';
```

慢查询次数参数可以结合慢查询日志找出慢查询语句，然后针对慢查询语句进行`表结构优化`或者`查询语句优化`。

再比如，如下的指令可以查看相关的指令情况：

```
SHOW STATUS LIKE 'Innodb_rows_%';
```



### 5.3 统计SQL的查询成本

一条SQL查询语句在执行前需要查询执行计划，如果存在多种执行计划的话，MySQL会计算每个执行计划所需要的成本，从中选择**成本最小**的一个作为最终执行的执行计划。

如果我们想要查看某条SQL语句的查询成本，可以在执行完这条SQL语句之后，通过**查看当前会话中的`last_query_cost`变量值**来得到当前查询的成本。它通常也是我们**评价一个查询的执行效率**的一个常用指标，通过如下指令进行查看：

```
SHOW STATUS LIKE 'last_query_cost';
```

这个查询成本对应的是**SQL 语句所需要读取的读页的数量**。

**使用场景：**它对于比较开销是非常有用的，特别是有好几种查询方式可选的时候。

> **举例**
>
> ```
> CREATE TABLE `student_info` (
>     `id` INT(11) NOT NULL AUTO_INCREMENT，
>     `student_id` INT NOT NULL ，
>     `name` VARCHAR(20) DEFAULT NULL，
>     `course_id` INT NOT NULL ，
>     `class_id` INT(11) DEFAULT NULL，
>     `create_time` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP，
>     PRIMARY KEY (`id`)
> ) ENGINE=INNODB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8;
> ```
>
> 如果我们想要查询 id=900001 的记录，然后看下查询成本，我们可以直接在聚簇索引上进行查找：
>
> ```
> SELECT student_id， class_id， NAME， create_time FROM student_info WHERE id = 900001;
> ```
>
> 运行结果（1 条记录，运行时间为 0.042s ）
>
> 然后再看下查询优化器的成本，实际上我们只需要检索一个页即可：
>
> ```
> mysql> SHOW STATUS LIKE 'last_query_cost';
> +-----------------+----------+
> | Variable_name   |   Value  |
> +-----------------+----------+
> | Last_query_cost | 1.000000 |
> +-----------------+----------+
> ```
>
> 如果我们想要查询 id 在 900001 到 9000100 之间的学生记录呢？
>
> ```
> SELECT student_id， class_id， NAME， create_time FROM student_info WHERE id BETWEEN 900001 AND 900100;
> ```
>
> 运行结果（100 条记录，运行时间为 0.046s ）：
>
> 然后再看下查询优化器的成本，这时我们大概需要进行 20 个页的查询。
>
> ```
> mysql> SHOW STATUS LIKE 'last_query_cost';
> +-----------------+-----------+
> | Variable_name   |   Value   |
> +-----------------+-----------+
> | Last_query_cost | 21.134453 |
> +-----------------+-----------+
> ```
>
> 你能看到页的数量是刚才的 20 倍，但是查询的效率并没有明显的变化，实际上这两个 SQL 查询的时间 基本上一样，就是因为采用了顺序读取的方式将页面一次性加载到缓冲池中，然后再进行查找。虽然 页 数量（last_query_cost）增加了不少 ，但是通过缓冲池的机制，并 没有增加多少查询时间 。
>
> 
>
> **结论**
>
> SQL查询是一个动态的过程，从页加载的角度来看，我们可以得到以下两点结论：
>
> 1. `位置决定效率`。如果页就在数据库 `缓冲池` 中，那么效率是最高的，否则还需要从 `内存` 或者 `磁盘` 中进行读取，当然针对单个页的读取来说，如果页存在于内存中，会比在磁盘中读取效率高很多。
> 2. `批量决定效率`。如果我们从磁盘中对单一页进行随机读，那么效率是很低的(差不多10ms)，而采用顺序读取的方式，批量对页进行读取，平均一页的读取效率就会提升很多，甚至要快于单个页面在内存中的随机读取。
>
> 所以说，遇到I/O并不用担心，方法找对了，效率还是很高的。我们首先要考虑数据存放的位置，如果是进程使用的数据就要尽量放到`缓冲池`中，其次我们可以充分利用磁盘的吞吐能力，一次性批量读取数据，这样单个页的读取效率也就得到了提升。



### 5.4 定位执行慢的 SQL：慢查询日志

MySQL的慢查询日志，用来**记录在MySQL中响应时间超过阀值的语句**，具体指运行时间超过`long_query_time`值的SQL，则会被记录到慢查询日志中。`long_query_time`的默认值为`10`，意思是运行10秒以上（不含10秒）的语句，认为是超出了我们的最大忍耐时间值。

它的主要作用是，帮助我们发现那些执行时间特别长的SQL查询，并且有针对性地进行优化，从而提高系统的整体效率。当我们的数据库服务器发生阻塞、运行变慢的时候，可以通过慢查询日志找到那些慢查询。

MySQL数据库**默认没有开启慢查询日志**，需要我们手动来设置这个参数。**如果不是调优需要的话，一般不建议启动该参数**，因为开启慢查询日志会或多或少带来一定的性能影响。 慢查询日志支持将日志记录写入文件。

#### 5.4.1 开启慢查询日志参数

**开启 slow_query_log**

在使用前，我们需要先查下慢查询是否已经开启，使用下面这条命令即可：

```
mysql > show variables like '%slow_query_log';
```

![image-20220628173525966](https://img.shiguangdev.cn/i/2024/09/22/66f01058201f6.png)

我们可以看到 `slow_query_log=OFF`，我们可以把慢查询日志打开，注意设置变量值的时候**需要使用 global**，否则会报错：

```
mysql > set global slow_query_log='ON';
```

然后我们再来查看下慢查询日志是否开启，以及慢查询日志文件的位置：

```
mysql > show variables like '%slow_query_log%';
```

![image-20220628175226812](https://img.shiguangdev.cn/i/2024/09/22/66f010583eeff.png)

你能看到这时慢查询分析已经开启，同时文件保存在 `/var/lib/mysql/atguigu02-slow.log` 文件 中。

**修改 long_query_time 阈值**

接下来我们来看下慢查询的时间阈值设置，使用如下命令：

```
mysql > show variables like '%long_query_time%';
```

![image-20220628175353233](https://img.shiguangdev.cn/i/2024/09/22/66f010584856a.png)

这里如果我们想把时间缩短，比如设置为 1 秒，可以这样设置：

```
mysql > set global long_query_time = 1;
mysql> show global variables like '%long_query_time%';
```

![image-20220628175425922](https://img.shiguangdev.cn/i/2024/09/22/66f010584ef2e.png)

**补充：配置文件中一并设置参数**

如下的方式相较于前面的命令行方式，可以看做是永久设置的方式。

修改 `my.cnf` 文件，`[mysqld]` 下增加或修改参数 `long_query_time、slow_query_log` 和 `slow_query_log_file` 后，然后重启 MySQL 服务器。

```
[mysqld]
slow_query_log=ON  # 开启慢查询日志开关
slow_query_log_file=/var/lib/mysql/atguigu-low.log  # 慢查询日志的目录和文件名信息
long_query_time=3  # 设置慢查询的阈值为3秒，超出此设定值的SQL即被记录到慢查询日志
log_output=FILE
```

如果不指定存储路径，慢查询日志默认存储到MySQL数据库的数据文件夹下。如果不指定文件名，默认文件名为`hostname_slow.log`。



#### 5.4.2 查看慢查询数目

查询当前系统中有多少条慢查询记录

```
SHOW GLOBAL STATUS LIKE '%Slow_queries%';
```

> **案例演示**
>
> **步骤1. 建表**
>
> ```
> CREATE TABLE `student` (
>     `id` INT(11) NOT NULL AUTO_INCREMENT，
>     `stuno` INT NOT NULL ，
>     `name` VARCHAR(20) DEFAULT NULL，
>     `age` INT(3) DEFAULT NULL，
>     `classId` INT(11) DEFAULT NULL，
>     PRIMARY KEY (`id`)
> ) ENGINE=INNODB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8;
> ```
>
> **步骤2：设置参数 `log_bin_trust_function_creators`**
>
> 创建函数，假如报错：
>
> ```
> This function has none of DETERMINISTIC......
> ```
>
> - 命令开启：允许创建函数设置：
>
> ```
> set global log_bin_trust_function_creators=1; # 不加global只是当前窗口有效。
> ```
>
> **步骤3：创建函数**
>
> 随机产生字符串：（同上一章）
>
> ```
> DELIMITER //
> CREATE FUNCTION rand_string(n INT)
> 	RETURNS VARCHAR(255) #该函数会返回一个字符串
> BEGIN
> 	DECLARE chars_str VARCHAR(100) DEFAULT
> 'abcdefghijklmnopqrstuvwxyzABCDEFJHIJKLMNOPQRSTUVWXYZ';
> 	DECLARE return_str VARCHAR(255) DEFAULT '';
>     DECLARE i INT DEFAULT 0;
>     WHILE i < n DO
>     	SET return_str =CONCAT(return_str，SUBSTRING(chars_str，FLOOR(1+RAND()*52)，1));
>     	SET i = i + 1;
>     END WHILE;
>     RETURN return_str;
> END //
> DELIMITER ;
> 
> # 测试
> SELECT rand_string(10);
> ```
>
> 产生随机数值：（同上一章）
>
> ```
> DELIMITER //
> CREATE FUNCTION rand_num (from_num INT ，to_num INT) RETURNS INT(11)
> BEGIN
>     DECLARE i INT DEFAULT 0;
>     SET i = FLOOR(from_num +RAND()*(to_num - from_num+1)) ;
>     RETURN i;
> END //
> DELIMITER ;
> 
> #测试：
> SELECT rand_num(10，100);
> ```
>
> **步骤4：创建存储过程**
>
> ```
> DELIMITER //
> CREATE PROCEDURE insert_stu1( START INT ， max_num INT )
> BEGIN
> DECLARE i INT DEFAULT 0;
>     SET autocommit = 0; #设置手动提交事务
>     REPEAT #循环
>     SET i = i + 1; #赋值
>     INSERT INTO student (stuno， NAME ，age ，classId ) VALUES
>     ((START+i)，rand_string(6)，rand_num(10，100)，rand_num(10，1000));
>     UNTIL i = max_num
>     END REPEAT;
>     COMMIT; #提交事务
> END //
> DELIMITER ;
> ```
>
> **步骤5：调用存储过程**
>
> ```
> #调用刚刚写好的函数， 4000000条记录，从100001号开始
> 
> CALL insert_stu1(100001，4000000);
> ```
>
> 
>
> **测试及分析**
>
> **1. 测试**
>
> ```
> mysql> SELECT * FROM student WHERE stuno = 3455655;
> +---------+---------+--------+------+---------+
> |   id    |  stuno  |  name  | age  | classId |
> +---------+---------+--------+------+---------+
> | 3523633 | 3455655 | oQmLUr |  19  |    39   |
> +---------+---------+--------+------+---------+
> 1 row in set (2.09 sec)
> 
> mysql> SELECT * FROM student WHERE name = 'oQmLUr';
> +---------+---------+--------+------+---------+
> |   id    |  stuno  |  name  |  age | classId |
> +---------+---------+--------+------+---------+
> | 1154002 | 1243200 | OQMlUR | 266  |   28    |
> | 1405708 | 1437740 | OQMlUR | 245  |   439   |
> | 1748070 | 1680092 | OQMlUR | 240  |   414   |
> | 2119892 | 2051914 | oQmLUr | 17   |   32    |
> | 2893154 | 2825176 | OQMlUR | 245  |   435   |
> | 3523633 | 3455655 | oQmLUr | 19   |   39    |
> +---------+---------+--------+------+---------+
> 6 rows in set (2.39 sec)
> ```
>
> 从上面的结果可以看出来，查询学生编号为“3455655”的学生信息花费时间为2.09秒。查询学生姓名为 “oQmLUr”的学生信息花费时间为2.39秒。已经达到了秒的数量级，说明目前查询效率是比较低的，下面 的小节我们分析一下原因。
>
> **2. 分析**
>
> ```
> show status like 'slow_queries';
> ```
>
> > **补充说明**： 除了上述变量，控制慢查询日志的还有一个系统变量：`min_examined_row_limit`。这个变量的意思是，查询扫描过的最少记录数。这个变量和查询执行时间，共同组成了判别一个查询是否是慢查询的条件。如果查询扫描过的记录数大于等于这个变量的值，并目查询执行时间超过`long_query_time`的值，那么，这个查询就被记录到慢查询日志中；反之，则不被记录到慢查询日志中。
> >
> > ![image-20240922205900112](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f01494c2328.png)
> >
> > 这个值默认是0。与`long_query_time=10`合在一起，表示只要查询的执行时间超过10秒钟，哪怕一个记录也没有扫描过，都要被记录到慢查询日志中。你也可以根据需要，通过修改`my.ini`文件，来修改查询时长，或者通过SET指令，用SQL语句修改`min_examined_row_limit`的值。

#### 5.4.3 慢查询日志分析工具：mysqldumpslow

在生产环境中，如果要手工分析日志，查找、分析SQL，显然是个体力活，MySQL提供了日志分析工具 `mysqldumpslow` 。

查看mysqldumpslow的帮助信息

```
mysqldumpslow --help
```

![image-20220628195821440](https://img.shiguangdev.cn/i/2024/09/22/66f0154f52b6d.png)

`mysqldumpslow `命令的具体参数如下：

- -a: 不将数字抽象成N，字符串抽象成S
- -s: 是表示按照何种方式排序：
  - c: 访问次数
  - l: 锁定时间
  - r: 返回记录
  - t: 查询时间
  - al:平均锁定时间
  - ar:平均返回记录数
  - at:平均查询时间 （默认方式）
  - ac:平均查询次数
- -t: 即为返回前面多少条的数据；
- -g: 后边搭配一个正则匹配模式，大小写不敏感的；

举例：我们想要按照查询时间排序，查看前五条 SQL 语句，这样写即可：

```
mysqldumpslow -s t -t 5 /var/lib/mysql/atguigu01-slow.log
[root@bogon ~]# mysqldumpslow -s t -t 5 /var/lib/mysql/atguigu01-slow.log

Reading mysql slow query log from /var/lib/mysql/atguigu01-slow.log
Count: 1 Time=2.39s (2s) Lock=0.00s (0s) Rows=13.0 (13)， root[root]@localhost
SELECT * FROM student WHERE name = 'S'

Count: 1 Time=2.09s (2s) Lock=0.00s (0s) Rows=2.0 (2)， root[root]@localhost
SELECT * FROM student WHERE stuno = N

Died at /usr/bin/mysqldumpslow line 162， <> chunk 2.
```

**工作常用参考：**

```
#得到返回记录集最多的10个SQL
mysqldumpslow -s r -t 10 /var/lib/mysql/atguigu-slow.log

#得到访问次数最多的10个SQL
mysqldumpslow -s c -t 10 /var/lib/mysql/atguigu-slow.log

#得到按照时间排序的前10条里面含有左连接的查询语句
mysqldumpslow -s t -t 10 -g "left join" /var/lib/mysql/atguigu-slow.log

#另外建议在使用这些命令时结合 | 和more 使用 ，否则有可能出现爆屏情况
mysqldumpslow -s r -t 10 /var/lib/mysql/atguigu-slow.log | more
```



#### 5.4.4 关闭慢查询日志

MySQL服务器停止慢查询日志功能有两种方法：

**方式1：永久性方式**

```
[mysqld]
slow_query_log=OFF
```

或者，把slow_query_log一项注释掉 或 删除

```
[mysqld]
#slow_query_log =OFF
```

重启MySQL服务，执行如下语句查询慢日志功能。

```
SHOW VARIABLES LIKE '%slow%'; #查询慢查询日志所在目录
SHOW VARIABLES LIKE '%long_query_time%'; #查询超时时长
```

**方式2：临时性方式**

使用SET语句来设置。

（1）停止MySQL慢查询日志功能，具体SQL语句如下。

```
SET GLOBAL slow_query_log=off;
```

（2）**重启MySQL服务**，使用SHOW语句查询慢查询日志功能信息，具体SQL语句如下。

```
SHOW VARIABLES LIKE '%slow%';
#以及
SHOW VARIABLES LIKE '%long_query_time%';
```



#### 5.4.5 删除慢查询日志

使用SHOW语句显示慢查询日志信息，具体SQL语句如下。

```
SHOW VARIABLES LIKE `slow_query_log%`;
```

![image-20220628203545536](https://img.shiguangdev.cn/i/2024/09/22/66f0154f3f0fe.png)

从执行结果可以看出，慢查询日志的目录默认为MySQL的数据目录，在该目录下 `手动删除慢查询日志文件` 即可。

使用命令 `mysqladmin flush-logs` 来重新生成查询日志文件，具体命令如下，执行完毕会在数据目录下重新生成慢查询日志文件。

```
mysqladmin -uroot -p flush-logs slow
```

> **提示:**
>
> 慢查询日志都是使用mysqladmin flush-logs命令来删除重建的。使用时一定要注意，一旦执行了这个命令，慢查询日志都只存在新的日志文件中，如果需要旧的查询日志，就必须事先备份。



### 5.5 分析查询语句：EXPLAIN

#### 5.5.1 概述

**定位了查询慢的SQL之后，我们就可以使用EXPLAIN或DESCRIBE工具做针对性的分析查询语句。**DESCRIBE 语句的使用方法与EXPLAIN语句是一样的，并且分析结果也是一样的。

MySQL中有专门负责优化SELECT语句的优化器模块，主要功能：通过计算分析系统中收集到的统计信息，为客户端请求的Query提供它认为**最优的执行计划**（他认为最优的数据检索方式，但不见得是DBA认为是最优的，这部分最耗费时间)。

这个执行计划展示了接下来具体执行查询的方式，比如多表连接的顺序是什么，对于每个表采用什么访问方法来具体执行查询等等。MySQL为我们提供了**`EXPLAIN`语句用来查看某个查询语句的具体执行计划**。

**作用**

- 表的读取顺序
- 数据读取操作的操作类型
- 哪些索引可以使用
- **哪些索引被实际使用**
- 表之间的引用
- **每张表有多少行被优化器查询**

**版本情况**

- `MySQL 5.6.3`以前只能 `EXPLAIN SELECT `，`MYSQL 5.6.3`以后就可以 `EXPLAIN SELECT，UPDATE， DELETE`
- 在5.7以前的版本中，想要显示 `partitions` 需要使用 `explain partitions` 命令；想要显示` filtered` 需要使用 `explain extended` 命令。在5.7版本后，默认explain直接显示`partitions`和 `filtered`中的信息。

![image-20220628211351678](https://img.shiguangdev.cn/i/2024/09/22/66f0199d6eb70.png)

#### 5.5.2 基本语法

如果我们想看看某个查询的执行计划的话，可以**在具体的查询语句前边加一个 EXPLAIN** ，就像这样：

EXPLAIN 或 DESCRIBE语句的语法形式如下：

```
mysql> EXPLAIN SELECT 1;
或者
mysql> DESCRIBE SELECT select_options
```

![image-20240922212505677](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f01ab25145f.png)

输出的上述信息就是所谓的`执行计划`。在这个执行计划的辅助下，我们需要知道应该怎样改进自己的查询语句以 使查询执行起来更高效。其实除了以`SELECT`开头的查询语句，其余的`DELETE`、`INSERT`、`REPLACE`以及 `UPDATE`语句等都可以加上`EXPLAIN`,用来查看这些语句的执行计划，只是平时我们对`SELECT`语句更感兴趣。

**注意：执行EXPLAIN时并没有真正的执行该后面的语句，因此可以安全的查看执行计划。**

EXPLAIN 语句输出的各个列的作用如下：

![image-20240922212649471](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f01b1a3d782.png)

在这里把它们都列出来知识为了描述一个轮廓，让大家有一个大致的印象。



#### 5.5.3 详细语法说明

**数据准备**

**1. 建表**

```
CREATE TABLE s1 (
    id INT AUTO_INCREMENT,
    key1 VARCHAR(100),
    key2 INT,
    key3 VARCHAR(100),
    key_part1 VARCHAR(100),
    key_part2 VARCHAR(100),
    key_part3 VARCHAR(100),
    common_field VARCHAR(100),
    PRIMARY KEY (id),
    INDEX idx_key1 (key1),
    UNIQUE INDEX idx_key2 (key2),
    INDEX idx_key3 (key3),
    INDEX idx_key_part(key_part1, key_part2, key_part3)
) ENGINE=INNODB CHARSET=utf8;
CREATE TABLE s2 (
    id INT AUTO_INCREMENT,
    key1 VARCHAR(100),
    key2 INT,
    key3 VARCHAR(100),
    key_part1 VARCHAR(100),
    key_part2 VARCHAR(100),
    key_part3 VARCHAR(100),
    common_field VARCHAR(100),
    PRIMARY KEY (id),
    INDEX idx_key1 (key1),
    UNIQUE INDEX idx_key2 (key2),
    INDEX idx_key3 (key3),
    INDEX idx_key_part(key_part1, key_part2, key_part3)
) ENGINE=INNODB CHARSET=utf8;
```

**2. 设置参数 log_bin_trust_function_creators**

创建函数，假如报错，需开启如下命令：允许创建函数设置：

```
set global log_bin_trust_function_creators=1; # 不加global只是当前窗口有效。
```

**3. 创建函数**

```
DELIMITER //
CREATE FUNCTION rand_string1(n INT)
	RETURNS VARCHAR(255) #该函数会返回一个字符串
BEGIN
	DECLARE chars_str VARCHAR(100) DEFAULT
'abcdefghijklmnopqrstuvwxyzABCDEFJHIJKLMNOPQRSTUVWXYZ';
    DECLARE return_str VARCHAR(255) DEFAULT '';
    DECLARE i INT DEFAULT 0;
    WHILE i < n DO
        SET return_str =CONCAT(return_str,SUBSTRING(chars_str,FLOOR(1+RAND()*52),1));
        SET i = i + 1;
    END WHILE;
    RETURN return_str;
END //
DELIMITER ;
```

**4. 创建存储过程**

创建往s1表中插入数据的存储过程：

```
DELIMITER //
CREATE PROCEDURE insert_s1 (IN min_num INT (10),IN max_num INT (10))
BEGIN
    DECLARE i INT DEFAULT 0;
    SET autocommit = 0;
    REPEAT
    SET i = i + 1;
    INSERT INTO s1 VALUES(
        (min_num + i),
        rand_string1(6),
        (min_num + 30 * i + 5),
        rand_string1(6),
        rand_string1(10),
        rand_string1(5),
        rand_string1(10),
        rand_string1(10));
    UNTIL i = max_num
    END REPEAT;
    COMMIT;
END //
DELIMITER ;
```

创建往s2表中插入数据的存储过程：

```
DELIMITER //
CREATE PROCEDURE insert_s2 (IN min_num INT (10),IN max_num INT (10))
BEGIN
    DECLARE i INT DEFAULT 0;
    SET autocommit = 0;
    REPEAT
    SET i = i + 1;
    INSERT INTO s2 VALUES(
        (min_num + i),
        rand_string1(6),
        (min_num + 30 * i + 5),
        rand_string1(6),
        rand_string1(10),
        rand_string1(5),
        rand_string1(10),
        rand_string1(10));
    UNTIL i = max_num
    END REPEAT;
    COMMIT;
END //
DELIMITER ;
```

**5. 调用存储过程**

s1表数据的添加：加入1万条记录：

```
CALL insert_s1(10001,10000);
```

s2表数据的添加：加入1万条记录：

```
CALL insert_s2(10001,10000);
```



##### **table**

不论我们的查询语句有多复杂，里边儿 包含了多少个表 ，到最后也是需要对每个表进行 单表访问 的，所 以MySQL规定EXPLAIN语句输出的每条记录都对应着某个单表的访问方法，该条记录的table列代表着该 表的表名（有时不是真实的表名字，可能是简称）。

```
mysql > EXPLAIN SELECT * FROM s1;
```

![image-20220628221143339](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f0c0f6adae5.png)

这个查询语句只涉及对s1表的单表查询，所以 `EXPLAIN` 输出中只有一条记录，其中的table列的值为s1，表明这条记录是用来说明对s1表的单表访问方法的。

下边我们看一个连接查询的执行计划

```
mysql > EXPLAIN SELECT * FROM s1 INNER JOIN s2;
```

![image-20220628221414097](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f0c0f6adaec.png)

可以看出这个连接查询的执行计划中有两条记录，这两条记录的table列分别是s1和s2，这两条记录用来分别说明对s1表和s2表的访问方法是什么。



##### **id**

我们写的查询语句一般都以 SELECT 关键字开头，比较简单的查询语句里只有一个 SELECT 关键字，比 如下边这个查询语句：

```
SELECT * FROM s1 WHERE key1 = 'a';
```

稍微复杂一点的连接查询中也只有一个 SELECT 关键字，比如：

```
SELECT * FROM s1 INNER JOIN s2
ON s1.key1 = s2.key1
WHERE s1.common_field = 'a';
```

但是下边两种情况下在一条查询语句中会出现多个SELECT关键字：

- **查询中包含子查询的情况** 比如下边这个查询语句中就包含2个SELECT关键字：

  ```
  SELECT * FROM s1
  WHERE key1 IN (SELECT key3 FROM s2);
  ```

- **查询中包含UNION语句的情况** 比如下边这个查询语句中也包含2个SELECT关键字：

  ```
  SELECT * FROM s1 UNION SELECT * FROM s2;
  ```

  **查询语句中每出现一个SELECT关键字，MySQL就会为它分配一个唯一的1d值。**这个`id`值就是`EXPLAIN`语句的第一个列，比如下边这个查询中只有一个`SELECT`关键字，所以`EXPLAIN`的结果中也就只有一条`id`列为1的记录：

  ```
  mysql > EXPLAIN SELECT * FROM s1 WHERE key1 = 'a';
  ```

  ![image-20240923092806748](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f0c426aff60.png)

对于连接查询来说，一个SELECT关键字后边的FROM字句中可以跟随多个表，所以在连接查询的执行计划中，每个表都会对应一条记录，但是这些记录的id值都是相同的，比如：

```
mysql> EXPLAIN SELECT * FROM s1 INNER JOIN s2;
```

![image-20220628222251309](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f0c437a4a61.png)

可以看到，上述连接查询中参与连接的s1和s2表分别对应一条记录，但是这两条记录对应的`id`都是1。这里需要大家记住的是，**在连接查询的执行计划中，每个表都会对应一条记录，这些记录的id列的值是相同的**，出现在前边的表表示`驱动表`，出现在后面的表表示`被驱动表`。所以从上边的EXPLAIN输出中我们可以看到，查询优化器准备让s1表作为驱动表，让s2表作为被驱动表来执行查询。

对于包含子查询的查询语句来说，就可能涉及多个`SELECT`关键字，所以在**包含子查询的查询语句的执行计划中，每个`SELECT`关键字都会对应一个唯一的id值**，比如这样：

```
mysql> EXPLAIN SELECT * FROM s1 WHERE key1 IN (SELECT key1 FROM s2) OR key3 = 'a';
```

![image-20220629165122837](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f0c437a7c94.png)

从输出结果中我们可以看到：s1表在外层查询中，外层查询有一个独立的`SELECT`关键字，所以第一条记录的1d 值就是1，s2表在子查询中，子查询有一个`独立的SELECT`关键字，所以第二条记录的`id`值就是`2`。

但是这里大家需要特别注意，**查询优化器可能对涉及子查询的查询语句进行重写，从而转换为连接查询。**所以如 果我们想知道查询优化器对某个包含子查询的语句是否进行了重写，直接查看执行计划就好了，比如说：

```
# 查询优化器可能对涉及子查询的查询语句进行重写，转变为多表查询的操作。  
mysql> EXPLAIN SELECT * FROM s1 WHERE key1 IN (SELECT key2 FROM s2 WHERE common_field = 'a');
```

![image-20240923093314852](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f0c55acaf83.png)

可以看到，虽然我们的查询语句是一个子查询，但是执行计划中s1和s2表对应的记录的`id`值全部是`1`，这就表明`查询优化器将子查询转换为了连接查询`。

对于包含`UNION`子句的查询语句来说，每个`SELECT`关键字对应一个`id`值也是没错的，不过还是有点儿特别的东西，比方说下边的查询：

```
# Union去重
mysql> EXPLAIN SELECT * FROM s1 UNION SELECT * FROM s2;
```

这个语句的执行计划的第三条记录是什么？为何`id`值是`NULL`,而且table列也很奇怪？`UNION`!它会把多个查询的结果集合并起来并对结果集中的记录进行去重，怎么去重呢？MySQL使用的是内部的临时表。正如上边的查询计划中所示，UNION子句是为了把id为1的查询和i为2的查询的结果集合并起来并去重，所以在内部创建了一个名为`<union1,2>`的临时表（就是执行计划第三条记录的table列的名称），id为`NULL`表明这个临时表是为了合并两个查询的结果集而创建的。

跟JNION对比起来，`UNION ALL`就不需要为最终的结果集进行去重，它只是单纯的把多个查询的结果集中的记录 合并成，一个并返回给用户，所以也就不需要使用临时表。所以在包含UNION ALL子句的查询的执行计划中，就没 有那个id为NULL的记录，如下所示

```
 mysql> EXPLAIN SELECT * FROM s1 UNION ALL SELECT * FROM s2;
```

![image-20240923093721716](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f0c651b14b2.png)**小结:**

- **id如果相同，可以认为是一组，从上往下顺序执行**
- **在所有组中，id值越大，优先级越高，越先执行**
- **关注点：id号每个号码，表示一趟独立的查询, 一个sql的查询趟数越少越好**



##### **select_type**

一条大的查询语句里边可以包含若干个SELECT关键字，`每个SELECT关键字代表着一个小的查询语句`，而每个SELECT关键字的FROM子句中都可以包含若干张表（这些表用来做连接查询），`每一张表都对应着执行计划输出中的一条记录`，对于在同一个SELECT关键字中的表来说，它们的id值是相同的。

MySQL为每一个SELECT关键字代表的小查询都定义了一个称之为`select_type`的属性，意思是我们只要知道了 某个小查询的`select_type`属性，就知道了这个`小查询在整个大查询中扮演了一个什么角色`，我们看一下 select_type都能取哪些值，请看官方文档：

![image-20240923094717708](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f0c8a5b1d86.png)

具体分析如下：

- SIMPLE

  查询语句中不包含`UNION`或者`子查询`的查询都算作是`SIMPLE`类型，比方说下边这个单表查询`select_type`的值就是`SIMPLE`:

  ```
  mysql> EXPLAIN SELECT * FROM s1;
  ```

![image-20220629171840300](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f0c8b879dbd.png)

 当然，连接查询也算是 SIMPLE 类型，比如：

```
mysql> EXPLAIN SELECT * FROM s1 INNER JOIN s2;
```

![image-20220629171904912](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f0c8b878a85.png)

- `PRIMARY`

  对于包含`UNION、UNION ALL`或者`子查询`的大查询来说，它是由几个小查询组成的，其中最左边的那个查询的`select_type`的值就是`PRIMARY`,比方说：

  ```
  mysql> EXPLAIN SELECT * FROM s1 UNION SELECT * FROM s2;
  ```

  ![image-20220629171929924](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f0c8b88f81e.png)

  从结果中可以看到，最左边的小查询`SELECT * FROM s1`对应的是执行计划中的第一条记录，它的`select_type`的值就是`PRIMARY`。

- `UNION`

  对于包含`UNION`或者`UNION ALL`的大查询来说，它是由几个小查询组成的，其中除了最左边的那个小查询意外，其余的小查询的`select_type`值就是UNION，可以对比上一个例子的效果。

- `UNION RESULT`

  MySQL 选择使用临时表来完成`UNION`查询的去重工作，针对该临时表的查询的`select_type`就是`UNION RESULT`, 例子上边有。

- `SUBQUERY`

  如果包含子查询的查询语句不能够转为对应的`semi-join`的形式，并且该子查询是`不相关子查询`，并且查询优化器决定采用将该子查询物化的方案来执行该子查询时，该子查询的第一个`SELECT`关键字代表的那个查询的`select_type`就是`SUBQUERY`，比如下边这个查询：

  ```
  mysql> EXPLAIN SELECT * FROM s1 WHERE key1 IN (SELECT key1 FROM s2) OR key3 = 'a';
  ```

  ![image-20220629172449267](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f0c8b886d63.png)

- `DEPENDENT SUBQUERY`

  ```
  mysql> EXPLAIN SELECT * FROM s1 WHERE key1 IN (SELECT key1 FROM s2 WHERE s1.key2 = s2.key2) OR key3 = 'a';
  ```

  ![image-20220629172525236](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f0c8b86cc1e.png)

- `DEPENDENT UNION`

  ```
  mysql> EXPLAIN SELECT * FROM s1 WHERE key1 IN (SELECT key1 FROM s2 WHERE key1 = 'a' UNION SELECT key1 FROM s1 WHERE key1 = 'b');
  ```

  ![image-20220629172555603](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f0c8b890179.png)

- `DERIVED`

  ```
  mysql> EXPLAIN SELECT * FROM (SELECT key1, count(*) as c FROM s1 GROUP BY key1) AS derived_s1 where c > 1;
  ```

  ![image-20220629172622893](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f0c8b8ec5ef.png)

  从执行计划中可以看出，id为2的记录就代表子查询的执行方式，它的`select_type`是`DERIVED`, 说明该子查询是以物化的方式执行的。`id`为`1`的记录代表外层查询，大家注意看它的table列显示的是`<derived2>`，表示该查询时针对将`派生表物化之后的表`(子查询的结果作为一张表)进行查询的。

- `MATERIALIZED`

  当查询优化器在执行包含子查询的语句时，选择将子查询物化之后的外层查询进行连接查询时，该子查询对应的`select_type`属性就是`DERIVED`，比如下边这个查询：

  ```
  mysql> EXPLAIN SELECT * FROM s1 WHERE key1 IN (SELECT key1 FROM s2);
  ```

  ![image-20220629172646367](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f0c8b969b8f.png)

- `UNCACHEABLE SUBQUERY`

  不常用，就不多说了。

- `UNCACHEABLE UNION`

  不常用，就不多说了。



##### **partitions (可略)**

- 代表分区表中的命中情况，非分区表，该项为`NULL`。一般情况下我们的额查询语句的执行计划的`partitions`列的值为`NULL`。
- [https://dev.mysql.com/doc/refman/5.7/en/alter-table-partition-operations.html](https://gitee.com/link?target=https%3A%2F%2Fdev.mysql.com%2Fdoc%2Frefman%2F5.7%2Fen%2Falter-table-partition-operations.html)
- 如果想详细了解，可以如下方式测试。创建分区表：

```
-- 创建分区表，
-- 按照id分区，id<100 p0分区，其他p1分区
CREATE TABLE user_partitions (id INT auto_increment,
NAME VARCHAR(12),PRIMARY KEY(id))
PARTITION BY RANGE(id)(
PARTITION p0 VALUES less than(100),
PARTITION p1 VALUES less than MAXVALUE
);
```

![image-20220629190304966](https://img.shiguangdev.cn/i/2024/09/23/66f0cb1bb4af6.png)

```
DESC SELECT * FROM user_partitions WHERE id>200;
```

查询id大于200（200>100，p1分区）的记录，查看执行计划，partitions是p1，符合我们的分区规则

![image-20220629190335371](https://img.shiguangdev.cn/i/2024/09/23/66f0cb1bbb0b1.png)



##### **type ☆**

执行计划的一条记录就代表着MySQL对某个表的 `执行查询时的访问方法` , 又称“访问类型”，其中的 `type` 列就表明了这个访问方法是啥，是较为重要的一个指标。比如，看到`type`列的值是`ref`，表明`MySQL`即将使用`ref`访问方法来执行对`s1`表的查询。

完整的访问方法如下： `system ， const ， eq_ref ， ref ， fulltext ， ref_or_null ， index_merge ， unique_subquery ， index_subquery ， range ， index ， ALL` 。

我们详细解释一下：

- `system`

  当表中`只有一条记录`并且该表使用的存储引擎的统计数据是精确的，比如`MyISAM`、`Memory`，那么对该表的访问方法就是`system`。比方说我们新建一个`MyISAM`表，并为其插入一条记录：

  ```
  mysql> CREATE TABLE t(i int) Engine=MyISAM;
  Query OK, 0 rows affected (0.05 sec)
  
  mysql> INSERT INTO t VALUES(1);
  Query OK, 1 row affected (0.01 sec)
  ```

  然后我们看一下查询这个表的执行计划：

  ```
  mysql> EXPLAIN SELECT * FROM t;
  ```

  

  可以看到`type`列的值就是`system`了，

  > 测试，可以把表改成使用InnoDB存储引擎，试试看执行计划的`type`列是什么。ALL

- `const`

  当我们根据主键或者唯一二级索引列与常数进行等值匹配时，对单表的访问方法就是`const`, 比如：

  ```
  mysql> EXPLAIN SELECT * FROM s1 WHERE id = 10005;
  ```

  

- `eq_ref`

  在连接查询时，如果被驱动表是通过主键或者唯一二级索引列等值匹配的方式进行访问的（如果该主键或者唯一二级索引是联合索引的话，所有的索引列都必须进行等值比较）。则对该被驱动表的访问方法就是`eq_ref`，比方说：

  ```
  mysql> EXPLAIN SELECT * FROM s1 INNER JOIN s2 ON s1.id = s2.id;
  ```

  

  从执行计划的结果中可以看出，MySQL打算将s2作为驱动表，s1作为被驱动表，重点关注s1的访问 方法是 `eq_ref` ，表明在访问s1表的时候可以 `通过主键的等值匹配` 来进行访问。

- `ref`

  当通过普通的二级索引列与常量进行等值匹配时来查询某个表，那么对该表的访问方法就可能是`ref`，比方说下边这个查询：

  ```
  mysql> EXPLAIN SELECT * FROM s1 WHERE key1 = 'a';
  ```

  

- `fulltext`

  全文索引

- `ref_or_null`

  当对普通二级索引进行等值匹配查询，该索引列的值也可以是`NULL`值时，那么对该表的访问方法就可能是`ref_or_null`，比如说：

  ```
  mysql> EXPLAIN SELECT * FROM s1 WHERE key1 = 'a' OR key1 IS NULL;
  ```

  

- `index_merge`

  一般情况下对于某个表的查询只能使用到一个索引，但单表访问方法时在某些场景下可以使用`Interseation、union、Sort-Union`这三种索引合并的方式来执行查询。我们看一下执行计划中是怎么体现MySQL使用索引合并的方式来对某个表执行查询的：

  ```
  mysql> EXPLAIN SELECT * FROM s1 WHERE key1 = 'a' OR key3 = 'a';
  ```

  

  从执行计划的 `type` 列的值是 `index_merge` 就可以看出，MySQL 打算使用索引合并的方式来执行 对 s1 表的查询。

- `unique_subquery`

  类似于两表连接中被驱动表的`eq_ref`访问方法，`unique_subquery`是针对在一些包含`IN`子查询的查询语句中，如果查询优化器决定将`IN`子查询转换为`EXISTS`子查询，而且子查询可以使用到主键进行等值匹配的话，那么该子查询执行计划的`type`列的值就是`unique_subquery`，比如下边的这个查询语句：

  ```
  mysql> EXPLAIN SELECT * FROM s1 WHERE key2 IN (SELECT id FROM s2 where s1.key1 = s2.key1) OR key3 = 'a';
  ```

  

- `index_subquery`

  `index_subquery` 与 `unique_subquery` 类似，只不过访问子查询中的表时使用的是普通的索引，比如这样：

  ```
  mysql> EXPLAIN SELECT * FROM s1 WHERE common_field IN (SELECT key3 FROM s2 where s1.key1 = s2.key1) OR key3 = 'a';
  ```

![image-20220703214407225](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f0cb1ca112c.png)

- `range`

  ```
  mysql> EXPLAIN SELECT * FROM s1 WHERE key1 IN ('a', 'b', 'c');
  ```

  ![image-20220703214633338](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f0cb1c94e2b.png)

  或者：

  ```
  mysql> EXPLAIN SELECT * FROM s1 WHERE key1 > 'a' AND key1 < 'b';
  ```

  ![image-20220703214657251](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f0cb1c98593.png)

- `index`

  当我们可以使用索引覆盖，但需要扫描全部的索引记录时，该表的访问方法就是`index`，比如这样：

  ```
  mysql> EXPLAIN SELECT key_part2 FROM s1 WHERE key_part3 = 'a';
  ```

  ![image-20220703214844885](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f0cb1cbea86.png)

  上述查询中的所有列表中只有key_part2 一个列，而且搜索条件中也只有 key_part3 一个列，这两个列又恰好包含在idx_key_part这个索引中，可是搜索条件key_part3不能直接使用该索引进行`ref`和`range`方式的访问，只能扫描整个`idx_key_part`索引的记录，所以查询计划的`type`列的值就是`index`。

  > 再一次强调，对于使用InnoDB存储引擎的表来说，二级索引的记录只包含索引列和主键列的值，而聚簇索引中包含用户定义的全部列以及一些隐藏列，所以扫描二级索引的代价比直接全表扫描，也就是扫描聚簇索引的代价更低一些。

- `ALL`

  最熟悉的全表扫描，就不多说了，直接看例子：

  ```
  mysql> EXPLAIN SELECT * FROM s1;
  ```

  ![image-20220703215958374](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f0cb1d134d6.png)

**小结: **

**system > const > eq_ref > ref** > fulltext > ref_or_null > index_merge > unique_subquery > index_subquery > range > index > ALL

**其中比较重要的几个提取出来（见上图中的蓝色部分）。SQL 性能优化的目标：至少要达到 range 级别，要求是 ref 级别，最好是 consts级别。（阿里巴巴 开发手册要求）**



##### **possible_keys和key**

在EXPLAIN语句输出的执行计划中，`possible_keys`列表示在某个查询语句中，对某个列执行`单表查询时可能用到的索引`有哪些。一般查询涉及到的字段上若存在索引，则该索引将被列出，但不一定被查询使用。`key`列表示`实际用到的索引`有哪些，如果为NULL，则没有使用索引。比方说下面这个查询：

```
mysql> EXPLAIN SELECT * FROM s1 WHERE key1 > 'z' AND key3 = 'a';
```

![image-20220703220724964](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f0d0938cb65.png)

上述执行计划的`possible_keys`列的值是`idx_key1, idx_key3`，表示该查询可能使用到`idx_key1, idx_key3`两个索引，然后`key`列的值是`idx_key3`，表示经过查询优化器计算使用不同索引的成本后，最后决定采用`idx_key3`。



##### **key_len ☆**

实际使用到的索引长度 (即：字节数)

帮你检查`是否充分的利用了索引`，`值越大越好`，主要针对于联合索引，有一定的参考意义。

```
mysql> EXPLAIN SELECT * FROM s1 WHERE id = 10005;
```

![image-20220704130030692](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f0d0938fb52.png)

> int 占用 4 个字节

```
mysql> EXPLAIN SELECT * FROM s1 WHERE key2 = 10126;
```

![image-20220704130138204](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f0d0938acb2.png)

> key2上有一个唯一性约束，是否为NULL占用一个字节，那么就是5个字节

```
mysql> EXPLAIN SELECT * FROM s1 WHERE key1 = 'a';
```

![image-20220704130214482](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f0d09388a55.png)

> key1 VARCHAR(100) 一个字符占3个字节，100*3，是否为NULL占用一个字节，varchar的长度信息占两个字节。

```
mysql> EXPLAIN SELECT * FROM s1 WHERE key_part1 = 'a';
```

![image-20220704130442095](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f0d09399315.png)

```
mysql> EXPLAIN SELECT * FROM s1 WHERE key_part1 = 'a' AND key_part2 = 'b';
```

![image-20220704130515031](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f0d09396fdf.png)

> 联合索引中可以比较，key_len=606的好于key_len=303



##### **ref**

显示索引的哪一列被使用了，如果可能的话，是一个常数。哪些列或常量被用于查找索引列上的值。

当使用索引列等值匹配的条件去执行查询时，也就是在访问方法是`const`、`eq_ref`、`ref`、`ref_or_nul1`、 `unique_subquery`、`1ndex_subquery`其中之一时，ref列展示的就是与索引列作等值匹配的结构是什么，比 如只是一个常数或者是某个列。大家看下边这个查询：

```
mysql> EXPLAIN SELECT * FROM s1 WHERE key1 = 'a';
```

![image-20240923103007640](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f0d2af93675.png)

可以看到`ref`列的值是`const`，表明在使用`idx_key1`索引执行查询时，与`key1`列作等值匹配的对象是一个常数，当然有时候更复杂一点:

```
mysql> EXPLAIN SELECT * FROM s1 INNER JOIN s2 ON s1.id = s2.id;
```

![image-20220704130925426](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f0d2ce51c73.png)

```
mysql> EXPLAIN SELECT * FROM s1 INNER JOIN s2 ON s2.key1 = UPPER(s1.key1);
```

![image-20220704130957359](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f0d2ce4deb8.png)



##### **rows ☆**

预估的需要读取的记录条数，`值越小越好`。

```
mysql> EXPLAIN SELECT * FROM s1 WHERE key1 > 'z';
```

![image-20220704131050496](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f0d2ce4b4f1.png)



##### **filtered**

某个表经过搜索条件过滤后剩余记录条数的百分比

如果使用的是索引执行的单表扫描，那么计算时需要估计出满足除使用到对应索引的搜索条件外的其他搜索条件的记录有多少条。

```
mysql> EXPLAIN SELECT * FROM s1 WHERE key1 > 'z' AND common_field = 'a';
```

![image-20220704131323242](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f0d2ce48aa3.png)

对于单表查询来说，这个filtered的值没有什么意义，我们`更关注在连接查询中驱动表对应的执行计划记录的filtered值`，它决定了被驱动表要执行的次数 (即: rows * filtered)

```
mysql> EXPLAIN SELECT * FROM s1 INNER JOIN s2 ON s1.key1 = s2.key1 WHERE s1.common_field = 'a';
```

![image-20220704131644615](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f0d2ce5bcc3.png)

从执行计划中可以看出来，查询优化器打算把`s1`作为驱动表，`s2`当做被驱动表。我们可以看到驱动表`s1`表的执行计划的`rows`列为`9688`，filtered列为`10.00`，这意味着驱动表`s1`的扇出值就是`9688 x 10.00% = 968.8`，这说明还要对被驱动表执行大约`968`次查询。



##### **Extra ☆**

顾名思义，`Extra`列是用来说明一些额外信息的，包含不适合在其他列中显示但十分重要的额外信息。我们可以通过这些额外信息来`更准确的理解MySQL到底将如何执行给定的查询语句`。MySQL提供的额外信息有好几十个，我们就不一个一个介绍了，所以我们只挑选比较重要的额外信息介绍给大家。

- `No tables used`

  当查询语句没有`FROM`子句时将会提示该额外信息，比如：

  ```
  mysql> EXPLAIN SELECT 1;
  ```

  ![image-20220704132345383](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f0d2ce4a694.png)

- `Impossible WHERE`

  当查询语句的`WHERE`子句永远为`FALSE`时将会提示该额外信息

  ```
  mysql> EXPLAIN SELECT * FROM s1 WHERE 1 != 1;
  ```

  ![image-20220704132458978](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f0d2cee740c.png)

- `Using where`

  不用读取表中所有信息，仅通过索引就可以获取所需数据，这发生在对表的全部的请求列都是同一个索引的 部分的时候，表示mysql服务器将在存储引擎检索行后再进行过滤。表明使用了where过滤。

  当我们使用全表扫描来执行对某个表的查询，并且该语句的`WHERE`子句中有针对该表的搜索条件时，在 `Extra`列中会提示上述额外信息。比如下边这个查询：

  ```
  mysql> EXPLAIN SELECT * FROM s1 WHERE common_field = 'a';
  ```

  ![image-20220704132655342](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f0d2cf508ce.png)

  当使用索引访问来执行对某个表的查询，并且该语句的`WHERE`子句中有除了该索引包含的列之外的其他搜索 条件时，在`Extra`列中也会提示上述额外信息。比如下边这个查询虽然使用`idx_key1`索引执行查询，但是搜索条件中除了包含`key1`的搜索条件`key1='a'`,还包含`common_field`的搜索条件，所以`Extra`列会显 示`Using where`的提示：

  ```
  mysql> EXPLAIN SELECT * FROM s1 WHERE key1 = 'a' AND common_field = 'a';
  ```

  ![image-20220704133130515](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f0d2cf79b05.png)

- `No matching min/max row`

  当查询列表处有`MIN`或者`MAX`聚合函数，但是并没有符合`WHERE`子句中的搜索条件的记录时。

  ```
  mysql> EXPLAIN SELECT MIN(key1) FROM s1 WHERE key1 = 'abcdefg';
  ```

  ![image-20220704134324354](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f0d2cf77355.png)

- `Using index`

  当我们的查询列表以及搜索条件中只包含属于某个索引的列，也就是在可以使用覆盖索引的情况下，在`Extra`列将会提示该额外信息。比方说下边这个查询中只需要用到`idx_key1`而不需要回表操作:

  ```
  mysql> EXPLAIN SELECT key1 FROM s1 WHERE key1 = 'a';
  ```

  ![image-20220704134931220](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f0d2cf85c0d.png)

- `Using index condition`

  有些搜索条件中虽然出现了索引列，但却不能使用到索引，比如下边这个查询：

  ```
  SELECT * FROM s1 WHERE key1 > 'z' AND key1 LIKE '%a';
  ```

  其中的`key1>‘z'`可以使用到索引，但是`key1 LIKE '%a'`却无法使用到索引，在以前版本的MySQL中， 是按照下边步骤来执行这个查询的：

  - 先根据`key1 >'z'`这个条件，从二级索引`idx_key1`中获取到对应的二级索引记录。
  - 根据上一步骤得到的二级索引记录中的主键值进行`回表`，找到完整的用户记录再检测该记录是否符合 `key1 LIKE '%a'`这个条件，将符合条件的记录加入到最后的结果集。
  - 但是虽然`key1 LIKE '%a'`不能组成范围区间参与`range`访问方法的执行，但这个条件毕竟只涉及到了 `key1`列，所以MySQL把上边的步骤改进一下：
  - 先根据`key1 >'z'`这个条件，定位到二级索引`idx_key1`中对应的二级索引记录。
  - 对于指定的二级索引记录，先不着急回表，而是先检测一下该记录是否满足`key1 LIKE '%a'`这个条件，如果这个条件不满足，则该二级索引记录压根儿就没必要回表。
  - 对于满足`key1 LIKE '%a'`这个条件的二级索引记录执行回表操作。

  我们说回表操作其实是一个`随机IO`，比较耗时，所以上述修改虽然只改进了一点点，但是可以省去好多回表操作的成本。MySQL把他们的这个改进称之为`索引条件下推`（英文名：`Index Condition Pushdown`)。

  如果在查询语句的执行过程中将要使用索引条件下推这个特性，在Extra列中将会显示`Using index condition`,比如这样：

  ```
  mysql> EXPLAIN SELECT * FROM s1 WHERE key1 > 'z' AND key1 LIKE '%b';
  ```

  ![image-20240923104201095](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f0d5790cf75.png)

- `Using join buffer (Block Nested Loop)`

  在连接查询执行过程中，当被驱动表不能有效的利用索引加快访问速度，MySQL一般会为其分配一块名叫`join buffer`的内存块来加快查询速度，也就是我们所讲的`基于块的嵌套循环算法`。

  ```
  mysql> EXPLAIN SELECT * FROM s1 INNER JOIN s2 ON s1.common_field = s2.common_field;
  ```

  ![image-20241016071244890](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/670ef6ebe8f7d.png)

- `Not exists`

  当我们使用左(外)连接时，如果`WHERE`子句中包含要求被驱动表的某个列等于`NULL`值的搜索条件，而且那个列是不允许存储`NULL`值的，那么在该表的执行计划的Extra列就会提示这个信息：

  ```
  mysql> EXPLAIN SELECT * FROM s1 LEFT JOIN s2 ON s1.key1 = s2.key1 WHERE s2.id IS NULL;
  ```

  ![image-20241016071416769](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/670ef747acaef.png)

- `Using intersect(...) 、 Using union(...) 和 Using sort_union(...)`

  如果执行计划的`Extra`列出现了`Using intersect(...)`提示，说明准备使用`Intersect`索引合并的方式执行查询，括号中的`...`表示需要进行索引合并的索引名称；

  如果出现`Using union(...)`提示，说明准备使用`Union`索引合并的方式执行查询;

  如果出现`Using sort_union(...)`提示，说明准备使用`Sort-Union`索引合并的方式执行查询。

  ```
  mysql> EXPLAIN SELECT * FROM s1 WHERE key1 = 'a' OR key3 = 'a';
  ```

  ![image-20241016071434599](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/670ef7597bad8.png)

- `Zero limit`

  当我们的`LIMIT`子句的参数为`0`时，表示压根儿不打算从表中读取任何记录，将会提示该额外信息

  ```
  mysql> EXPLAIN SELECT * FROM s1 LIMIT 0;
  ```

  ![image-20241016071449589](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/670ef7687d980.png)

- `Using filesort`

  有一些情况下对结果集中的记录进行排序是可以使用到索引的。

  ```
  mysql> EXPLAIN SELECT * FROM s1 ORDER BY key1 LIMIT 10;
  ```

  ![image-20240923104248280](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f0d5a835453.png)

  这个查询语句可以利用1 dx_key1索引直接取出key1列的10条记录，然后再进行回表操作就好了。但是很多 情况下排序操作无法使用到索引，只能在内存中（记录较少的时候）或者磁盘中（记录较多的时候）进行排 序，MySQL把这种在内存中或者磁盘上进行排序的方式统称为文件排序（英文名：f11 esort)。如果某个查 询需要使用文件排序的方式执行查询，就会在执行计划的Extra列中显示Using filesort提示，比如这 样：

  ```
  mysql> EXPLAIN SELECT * FROM s1 ORDER BY common_field LIMIT 10;
  ```

  ![image-20240923193127721](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f1518f724c0.png)

  需要注意的是，如果查询中需要使用`filesort`的方式进行排序的记录非常多，那么这个过程是很耗费性能的，我们最好想办法`将使用文件排序的执行方式改为索引进行排序`。

- `Using temporary`

  在许多查询的执行过程中，MySQL可能会借助临时表来完成一些功能，比如去重、排序之类的，比如我们在执 行许多包含`DISTINCT`、`GROUP BY`、`UNION`等子句的查询过程中，如果不能有效利用索引来完成查询，MySQL很有可能寻求通过建立内部的临时表来执行查询。如果查询中使用到了内部的临时表，在执行计划的 `Extra`列将会显示`Using temporary`提示，比方说这样：

  ```
  mysql> EXPLAIN SELECT DISTINCT common_field FROM s1;
  ```

  ![image-20240923193408862](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f152307704e.png)

  再比如：

  ```
  mysql> EXPLAIN SELECT common_field, COUNT(*) AS amount FROM s1 GROUP BY common_field;
  ```

  ![image-20240923193431866](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f152477620a.png)

  执行计划中出现`Using temporary`并不是一个好的征兆，因为建立与维护临时表要付出很大的成本的，所以我们`最好能使用索引来替代掉使用临时表`，比方说下边这个包含`GROUP BY`子句的查询就不需要使用临时表：

  ```
  mysql> EXPLAIN SELECT key1, COUNT(*) AS amount FROM s1 GROUP BY key1;
  ```

  ![image-20240923193457164](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f15260c2987.png)

  从 `Extra` 的 `Using index` 的提示里我们可以看出，上述查询只需要扫描 `idx_key1` 索引就可以搞 定了，不再需要临时表了。

- 其他

  其它特殊情况这里省略。



#### 5.5.4 小结

- EXPLAIN不考虑各种Cache
- EXPLAIN不能显示MySQL在执行查询时所作的优化工作
- EXPLAIN不会告诉你关于触发器、存储过程的信息或用户自定义函数对查询的影响情况
- 部分统计信息是估算的，并非精确值



# 三、MySQL事务篇

## 1. 事务的基本概念

**事务：**一组逻辑操作单元，使数据从一种状态变换到另一种状态。(一组不可分隔的操作)

**事务处理的原则：**保证所有事务都作为 `一个工作单元` 来执行，即使出现了故障，都不能改变这种执行方 式。当在一个事务中执行多个操作时，要么所有的事务都被提交( `commit` )，那么这些修改就 `永久` 地保 `存下来`；要么数据库管理系统将 `放弃` 所作的所有 `修改` ，整个事务回滚( rollback )到最初状态。

## 2. 引擎支持情况

`SHOW ENGINES` 命令来查看当前 MySQL 支持的存储引擎都有哪些，以及这些存储引擎是否支持事务。

![image-20240924221853655](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f2ca4cb45c2.png)

## 3. 事务的ACID特性

### 3.1 原子性（atomicity）

​	原子性是指事务是一个不可分割的工作单位，**要么全部提交，要么全部失败回滚**。即要么转账成功，要么转账失败，是不存在中间的状态。如果无法保证原子性会怎么样？就会出现数据不一致的情形，A账户减去100元，而B账户增加100元操作失败，系统将无故丢失100元。

### 3.2 一致性（consistency）

​	根据定义，一致性是指事务执行前后，数据从一个 `合法性状态` 变换到另外一个 `合法性状态` 。这种状态是 `语义上` 的而不是语法上的，跟具体的业务有关。

​	其中，满足 `预定的约束` 的状态就叫做合法的状态。通俗一点，这状态是由你自己来定义的（比如满足现实世界中的约束）。满足这个状态，数据就是一致的，不满足这个状态，数据就 是不一致的！如果事务中的某个操作失败了，系统就会自动撤销当前正在执行的事务，返回到事务操作之前的状态。

> **举例1：**A账户有200元，转账300元出去，此时A账户余额为-100元。你自然就发现此时数据是不一致的，为什么呢？因为你定义了一个状态，余额这列必须>=0。
>
> **举例2：**A账户有200元，转账50元给B账户，A账户的钱扣了，但是B账户因为各种意外，余额并没有增加。你也知道此时的数据是不一致的，为什么呢？因为你定义了一个状态，要求A+B的总余额必须不变。
>
> **举例3：**在数据表中我们将`姓名`字段设置为`唯一性约束`，这时当事务进行提交或者事务发生回滚的时候，如果数据表的姓名不唯一，就破坏了事务的一致性要求。

### 3.3 隔离性（isolation）

​	事务的隔离性是指一个事务的执行`不能被其他事务干扰`，即一个事务内部的操作及使用的数据对`并发`的其他事务是隔离的，并发执行的各个事务之间不能相互干扰。

​	如果无法保证隔离性会怎么样？假设A账户有200元，B账户0元。A账户往B账户转账两次，每次金额为50 元，分别在两个事务中执行。如果无法保证隔离性，会出现下面的情形：

```
UPDATE accounts SET money = money - 50 WHERE NAME = 'AA';
UPDATE accounts SET money = money + 50 WHERE NAME = 'BB';
```

![image-20240924222855450](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f2cca67ddf9.png)

### 3.4 持久性（durability)

​	持久性是指一个事务一旦被提交，它对数据库中数据的改变就是 `永久性的` ，接下来的其他操作和数据库故障不应该对其有任何影响。

​	持久性是通过 `事务日志` 来保证的。日志包括了 `redo log` 和 `undo log` 。当我们通过事务对数据进行修改 的时候，首先会将数据库的变化信息记录到重做日志中，然后再对数据库中对应的行进行修改。这样做 的好处是，即使数据库系统崩溃，数据库重启后也能找到没有更新到数据库系统中的重做日志，重新执 行，从而使事务具有持久性。



## 4 事务的状态

一个基本的状态转换图如下所示：

![image-20220708171859055](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f2ce192699f.png)

​	图中可见，只有当事务处于`提交的`或者`中止的`状态时，一个事务的生命周期才算是结束了。对于**已经提交的事务**来说，该事务对数据库所做的修改将永久生效；对于**处于中止状态的事务**，该事务对数据库所做的所有修改都会被回滚到没执行该事务之前的状态。

### 4.1 活动的（active）

​	事务对应的数据库操作正在执行过程中时，我们就说该事务处在 `活动的` 状态。

### 4.2 部分提交的（partially committed）

​	当事务中的最后一个操作执行完成，但由于操作都在内存中执行，所造成的影响并 `没有刷新到磁盘` 时，我们就说该事务处在 `部分提交的` 状态。

### 4.3 失败的（failed）

​	当事务处在 `活动的` 或者 部分提交的 状态时，可能遇到了某些错误（数据库自身的错误、操作系统 错误或者直接断电等）而无法继续执行，或者人为的停止当前事务的执行，我们就说该事务处在失败的状态。

### 4.4 中止的（aborted）

​	如果事务执行了一部分而变为 `失败的` 状态，那么就需要把已经修改的事务中的操作还原到事务执 行前的状态。换句话说，就是要撤销失败事务对当前数据库造成的影响。我们把这个撤销的过程称之为 `回滚` 。当 `回滚` 操作执行完毕时，也就是数据库恢复到了执行事务之前的状态，我们就说该事 务处在了 `中止的` 状态。

### 4.5 提交的（committed）

​	当一个处在 `部分提交的` 状态的事务将修改过的数据都 `同步到磁盘` 上之后，我们就可以说该事务处在了 `提交的` 状态。



## 5 事务的使用

使用事务有两种方式，分别为 `显式事务` 和 `隐式事务` 。

### 5.1 显式事务

**步骤1：** **START TRANSACTION** 或者 **BEGIN** ，作用是显式开启一个事务。

```
mysql> BEGIN;
#或者
mysql> START TRANSACTION;
```

`START TRANSACTION` 语句相较于 `BEGIN` 特别之处在于，后边能跟随几个 `修饰符` ：

① **READ ONLY** ：标识当前事务是一个 `只读事务` ，也就是属于该事务的数据库操作只能读取数据，而不能修改数据。

> 补充：只读事务中只是不允许修改那些其他事务也能访问到的表中的数据，对于临时表来说（我们使用 CREATE TMEPORARY TABLE 创建的表），由于它们只能再当前会话中可见，所有只读事务其实也是可以对临时表进行增、删、改操作的。

② **READ WRITE(默认)** ：标识当前事务是一个 `读写事务` ，也就是属于该事务的数据库操作既可以读取数据， 也可以修改数据。

③ **WITH CONSISTENT SNAPSHOT** ：启动一致性读，在事务开始时，立即创建一个数据的一致性快照，使得在事务执行期间，无论其他并发事务如何修改数据，本事务所看到的数据都是一致的

注意：

- `READ ONLY`和`READ WRITE`是用来设置所谓的事务`访问模式`的，就是以只读还是读写的方式来访问数据库中的数据，一个事务的访问模式不能同时即设置为`只读`的也设置为`读写`的，所以不能同时把`READ ONLY`和`READ WRITE`放到`START TRANSACTION`语句后边。
- 如果我们不显式指定事务的访问模式，那么该事务的访问模式就是`读写`模式

**步骤2：**一系列事务中的操作（主要是DML，不含DDL）

**步骤3：**提交事务 或 中止事务（即回滚事务）

```
# 提交事务。当提交事务后，对数据库的修改是永久性的。
mysql> COMMIT;
# 回滚事务。即撤销正在进行的所有没有提交的修改
mysql> ROLLBACK;

# 将事务回滚到某个保存点。
mysql> ROLLBACK TO [SAVEPOINT]
```

其中关于SAVEPOINT相关操作有：

```
# 在事务中创建保存点，方便后续针对保存点进行回滚。一个事务中可以存在多个保存点。
SAVEPOINT 保存点名称;
# 删除某个保存点
RELEASE SAVEPOINT 保存点名称;
```



### 5.2 隐式事务

#### 5.2.1 autocommit

​	MySQL中有一个系统变量 `autocommit` ，默认开启，此时每一句SQL语句都是一个事务。

```
mysql> SHOW VARIABLES LIKE 'autocommit';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| autocommit    |  ON   |
+---------------+-------+
1 row in set (0.01 sec)
```

> 当然，如果我们想关闭这种 `自动提交` 的功能，可以使用下边两种方法之一：
>
> - 显式的的使用 `START TRANSACTION` 或者 `BEGIN` 语句开启一个事务。这样在本次事务提交或者回滚前会暂时关闭掉自动提交的功能。
> - 把系统变量 `autocommit` 的值设置为 `OFF` ，就像这样：
>
> ```
> 针对DML操作有效，针对DDLc
> 
> SET autocommit = OFF;
> #或
> SET autocommit = 0;
> ```

#### 5.2.2 数据定义语言

​	数据定义语言（Data definition language，缩写为：DDL）是用来操作数据库对象的SQL语言。

​	数据库对象，指的就是`数据库、表、视图、存储过程`等结构。当我们`CREATE、ALTER、DROP`等语句去修改数据库对象时，就会隐式的提交前边语句所属于的事务。即：

```
BEGIN;

SELECT ... # 事务中的一条语句
UPDATE ... # 事务中的一条语句
... # 事务中的其他语句

CREATE TABLE ... # 此语句会隐式的提交前边语句所属于的事务
```

#### 5.2.3 隐式使用或修改mysql数据库中的表

​	当我们使用`ALTER USER`、`CREATE USER`、`DROP USER`、`GRANT`、`RENAME USER`、`REVOKE`、`SET PASSWORD`等语句时也会隐式的提交前边语句所属于的事务。

#### 5.2.4 事务控制或关于锁定的语句

​	① 当我们在一个事务还没提交或者回滚时就又使用 START TRANSACTION 或者 BEGIN 语句开启了另一个事务时，会隐式的提交上一个事务。即：

```
BEGIN;

SELECT ... # 事务中的一条语句
UPDATE ... # 事务中的一条语句
... # 事务中的其他语句

BEGIN; # 此语句会隐式的提交前边语句所属于的事务
```

​	② 当前的 autocommit 系统变量的值为 OFF ，我们手动把它调为 ON 时，也会隐式的提交前边语句所属的事务。

​	③ 使用 LOCK TABLES 、 UNLOCK TABLES 等关于锁定的语句也会隐式的提交前边语句所属的事务。

#### 5.2.5 加载数据的语句

​	使用`LOAD DATA`语句来批量往数据库中导入数据时，也会`隐式的提交`前边语句所属的事务。

#### 5.2.6 关于MySQL复制的一些语句

​	使用`START SLAVE、STOP SLAVE、RESET SLAVE、CHANGE MASTER TO`等语句会隐式的提交前边语句所属的事务。

#### ~~5.2.7 其他的一些语句~~

​	~~使用`ANALYZE TABLE、CACHE INDEX、CAECK TABLE、FLUSH、LOAD INDEX INTO CACHE、OPTIMIZE TABLE、REPAIR TABLE、RESET`等语句也会隐式的提交前边语句所属的事务。~~



## 6 事务隔离级别

### 6.1 数据并发问题

​	并发事务执行过程中可能遇到的一些问题，这些问题有轻重缓急之分，我们给这些问题按照严重性来排一下序：

```
脏写 > 脏读 > 不可重复读 > 幻读
```

#### 6.1.1 脏写（ Dirty Write ）

​	对于两个事务 Session A、Session B，如果事务Session A `修改了` 另一个 `未提交` 事务Session B `修改过` 的数据，那就意味着发生了 `脏写`，示意图如下：

![image-20220708214453902](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f37549cf668.png)

​	Session A 和 Session B 各开启了一个事务，Sesssion B 中的事务先将studentno列为1的记录的name列更新为'李四'，然后Session A中的事务接着又把这条studentno列为1的记录的name列更新为'张三'。如果之后Session B中的事务进行了回滚，那么Session A中的更新也将不复存在，这种现象称之为脏写。这时Session A中的事务就没有效果了，明明把数据更新了，最后也提交事务了，最后看到的数据什么变化也没有。

#### 6.1.2 脏读（ Dirty Read ）

​	对于两个事务 Session A、Session B，Session A `读取` 了已经被 Session B `更新` 但还 `没有被提交` 的字段。 之后若 Session B `回滚` ，Session A `读取 `的内容就是 `临时且无效` 的。

![image-20220708215109480](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f37549d6796.png)

​	Session A和Session B各开启了一个事务，Session B中的事务先将studentno列为1的记录的name列更新 为'张三'，然后Session A中的事务再去查询这条studentno为1的记录，如果读到列name的值为'张三'，而 Session B中的事务稍后进行了回滚，那么Session A中的事务相当于读到了一个不存在的数据，这种现象就称之为 `脏读` 。

#### 6.1.3 不可重复读（ Non-Repeatable Read ）

​	对于两个事务Session A、Session B，Session A `读取`了一个字段，然后 Session B `更新`了该字段。 之后 Session A `再次读取` 同一个字段， `值就不同` 了。那就意味着发生了不可重复读。

![image-20220708215626435](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f3754a07cc8.png)

​	我们在Session B中提交了几个 `隐式事务` （注意是隐式事务，意味着语句结束事务就提交了），这些事务 都修改了studentno列为1的记录的列name的值，每次事务提交之后，如果Session A中的事务都可以查看到最新的值，这种现象也被称之为 `不可重复读 `。

#### 6.1.4 幻读（ Phantom ）

​	对于两个事务Session A、Session B， Session A 从一个表中 `读取` 了一个字段， 然后 Session B 在该表中 插 入 了一些新的行。 之后， 如果 Session A `再次读取` 同一个表， 就会多出几行。那就意味着发生了`幻读`。

![image-20220708220102342](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f3754a0c239.png)

​	Session A中的事务先根据条件 studentno > 0这个条件查询表student，得到了name列值为'张三'的记录； 之后Session B中提交了一个 `隐式事务` ，该事务向表student中插入了一条新记录；之后Session A中的事务 再根据相同的条件 studentno > 0查询表student，得到的结果集中包含Session B中的事务新插入的那条记 录，这种现象也被称之为 幻读 。我们把新插入的那些记录称之为 `幻影记录` 。



### 6.2 SQL中的四种隔离级别

​	不同的隔离级别有不同的现象，并有不同的锁和并发机制，**隔离级别越高**，数据库的**并发性能就越差**，4 种事务隔离级别与并发性能的关系如下：

![image-20220708220957108](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f375edc39e5.png)

#### 6.2.1 `READ UNCOMMITTED`

​	读未提交，在该隔离级别，所有事务都可以看到其他未提交事务的执行结 果。**只能避免脏写**，不能避免脏读、不可重复读、幻读。

#### 6.2.2 `READ COMMITTED`

​	读已提交，它满足了隔离的简单定义：一个事务只能看见已经提交事务所做的改变。这是**大多数数据库系统的默认隔离级别**（但不是MySQL默认的）。可以**避免脏读**，但不可重复读、幻读问题仍然存在。

#### 6.2.3 `REPEATABLE READ`

​	可重复读，事务A在读到一条数据之后，此时事务B对该数据进行了修改并提交，那么事务A再读该数据，读到的还是原来的内容。可以**避免脏读、不可重复读**，但幻读问题仍 然存在。这是**MySQL的默认隔离级别**。

#### 6.2.4 `SERIALIZABLE`

​	可串行化，确保事务可以从一个表中读取相同的行。在这个事务持续期间，禁止其他事务对该表执行插入、更新和删除操作。所有的并发问题都可以避免，但性能十分低下。能**避免脏读、不可重复读和幻读**。



### 6.3 如何设置事务的隔离级别

通过下面的语句修改事务的隔离级别：

```
SET [GLOBAL|SESSION] TRANSACTION ISOLATION LEVEL 隔离级别;
#其中，隔离级别格式：
> READ UNCOMMITTED
> READ COMMITTED
> REPEATABLE READ
> SERIALIZABLE
```

或者：

```
SET [GLOBAL|SESSION] TRANSACTION_ISOLATION = '隔离级别'
#其中，隔离级别格式：
> READ-UNCOMMITTED
> READ-COMMITTED
> REPEATABLE-READ
> SERIALIZABLE
```

**关于设置时使用GLOBAL或SESSION的影响：**

- 使用 GLOBAL 关键字会对执行完该语句之后产生的会话起作用，但对当前会话无效。

```
SET GLOBAL TRANSACTION ISOLATION LEVEL SERIALIZABLE;
#或
SET GLOBAL TRANSACTION_ISOLATION = 'SERIALIZABLE';
```

- 使用 `SESSION` 关键字会对当前会话在执行完该语句之后创建的事务起作用，但是对其他会话以及当前会话执行该语句之前创建的事务无效。

```
SET SESSION TRANSACTION ISOLATION LEVEL SERIALIZABLE;
#或
SET SESSION TRANSACTION_ISOLATION = 'SERIALIZABLE';
```

​	如果在服务器启动时想改变事务的默认隔离级别，可以修改启动参数`transaction_isolation`的值。比如，在启动服务器时指定了`transaction_isolation=SERIALIZABLE`，那么事务的默认隔离界别就从原来的`REPEATABLE-READ`变成了`SERIALIZABLE`。


## 7. 三大日志

### 7.1 总述

#### 7.1.1 概念总述	

​	**Buffer Pool**：`MySQL`中数据是**以页为单数存储**，当你查询一条记录时，硬盘会把一整页的数据加载出来，加载出来的数据叫做数据页，会放到`Buffer Pool`中。后续的查询都是先从`Buffer Pool`中找，没有找到再去硬盘加载其他的数据页直到命中，这样子可以减少磁盘`IO`的次数，提高性能。

​	**redo log**：是**事务持久性和一致性**的保障，是一个**物理日志**，是  **InnoDB存储引擎** 独有的，它让`MySQL`有了**崩溃恢复**的能力。当`MySQL`实例挂了或者宕机了，**重启的时候**`InnoDB`存储引擎会使用`rede log`日志**恢复数据**，**保证事务的持久性和完整性**。

#### 7.1.2 执行流程总述

（1）从`Buffer Pool`中找被相关的数据，没有找到再去硬盘加载其他的数据页直到命中。

（2）将对数据的更新操作相反的操作记录到`Undo Log Buffer`当中。

（3）将对数据的操作更新到`buffer pool`里面的数据页，使其变成脏页。

（4）将对数据页的操作记录到`Redo Log Buffer`当中，此时Redo Log中的当前事务。

（5）将事务内的操作写入到`Binlog Cache`中。

（6）若事务未提交，继续执行其他操作，则重复步骤（1）~（5）。

（7）将`Undo Log`和`Redo Log`进行刷盘，并且`Redo Log`记为`Prepare`状态一同刷入磁盘中。

（8）将`Binlog Cache`刷入`Binlog`文件中，并将`Redo Log`状态更新为`Commit`状态。

![MySQL高频八股——事务过程中undolog、redolog、binlog的写入顺序（涉及两阶段提交）](https://picx.zhimg.com/70/v2-fe1e5401b53ebde013e929491aeda1d6_1440w.image?source=172ae18b&biz_tag=Post)

（注意上图更新为Prepare和Commit状态的是redo log，不是undo log）

### 7.2 Redo Log

#### 7.2.1 概念

​	**redo log**是事务**持久性和原子性**的保障；是一个**物理日志**（每条`redo log`记录由“`表空间号+数据页号+偏移量+修改数据长度+具体修改的数据`”组成）；是  **InnoDB存储引擎** 独有的；它让`MySQL`有了**崩溃恢复**的能力，当`MySQL`实例挂了或者宕机了，**重启的时候**`InnoDB`存储引擎会使用`rede log`日志**恢复数据**，**保证事务的持久性和完整性**。如下图：

![img](https://segmentfault.com/img/remote/1460000041758787)

#### 7.2.2 写入机制

​	更新数据的时候会把 “在某个数据页做了什么修改” 记录到`redo log buffer`里，在合适的时机写入`page cache`，并从`page cache`刷盘到磁盘的`redo log`日志文件里，如下图：

![img](https://segmentfault.com/img/remote/1460000041758788)

#### 7.2.3 刷盘时机

​	首先，`Innodb`存储引擎的`master thread`，每隔`1`秒，就会把会`redo log buffer`中的内容写入到文件系统缓存`page cache`，然后调用`fsync`刷盘，如下图。因此，即使是没有进行事务提交的`redo log`记录，也会被刷盘。

![img](https://segmentfault.com/img/remote/1460000041758789)



​	此外，`InnoDB`存储引擎为`redo log`的刷盘策略提供了`innodb_flush_log_at_trx_commit`参数，它支持三种策略：

- innodb_flush_log_at_trx_commit = 0：设置为0的时候，每次提交事务时不进行任何操作，刷盘全靠后台线程每秒一次进行。

  如下图，如果**服务器宕机**了或者**MySQL挂了**可能造成**1秒内的数据丢失**。

  ![img](https://segmentfault.com/img/remote/1460000041758790)

- innodb_flush_log_at_trx_commit = 1：**（默认）**设置为1的时候，每次提交事务时都将`redo log buffer`写入`page cache`，并让`page cache`刷盘。

  如下图，只要事务提交成功，`redo log`记录就一定在磁盘里，不会有任务数据丢失。如果执行事务的时候**MySQL挂了**或者**服务器宕机**了，这部分日志丢失了，但是因为事务没有提交，所以日志丢了也**不会有损失**。

  ![img](https://segmentfault.com/img/remote/1460000041758791)

- innodb_flush_log_at_trx_commit = 2：设置为2的时候，每次提交事务时都只把`redo log buffer`写入`page cache`。

  如下图，当事务提交成功时，`redo log buffer`日志会被写入`page cache`，然后由后台线程刷盘写入`redo log`，由于后台线程是`1`秒执行一次所以**服务器宕机**可能造成**1秒内的数据丢失**，但**MySQL挂了**并**不会有损失**。

  ![img](https://segmentfault.com/img/remote/1460000041758792)

#### 7.2.4 存储结构

写入redo log buffer 过程

> **补充概念：Mini-Transaction**
>
> ​	MySQL把对底层页面中的一次原子访问过程称之为一个`Mini-Transaction`，简称`mtr`，比如，向某个索引对应的B+树中插入一条记录的过程就是一个`Mini-Transaction`。一个所谓的`mtr`可以包含一组redo日志，在进行崩溃恢复时这一组`redo`日志可以作为一个不可分割的整体。
>
> ​	一个事务可以包含若干条语句，每一条语句其实是由若干个 `mtr` 组成，每一个 `mtr` 又可以包含若干条 redo日志，画个图表示它们的关系就是这样：
>
> ![image-20240925124441207](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f3953977ee4.png)

**redo 日志写入log buffer**

向`log buffer`中写入redo日志的过程是顺序的，也就是先往前边的block中写，当该block的空闲空间用完之后再往下一个block中写。当我们想往`log buffer`中写入redo日志时，第一个遇到的问题就是应该写在哪个`block`的哪个偏移量处，所以InnoDB的设计者特意提供了一个称之为`buf_free`的全局变量，该变量指明后续写入的 redo日志应该写入到`log buffer`中的哪个位置，如图所示：

![image-20240925124739050](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f395eb560f5.png)

一个mtr执行过程中可能产生若干条redo日志，`这些redo日志是一个不可分割的组`，所以其实并不是每生成一条 redo日志，就将其插入到log buffer中，而是每个mtr运行过程中产生的日志先暂时存到一个地方，当该mtr结束的时候，将过程中产生的一组redo日志再全部复制到log buffert中。我们现在假设有两个名为`T1`、`T2`的事务，每个事务都包含2个mtr，我们给这几个mtr命名一下：

- 事务`T1`的两个`mtr`分别称为`mtr_T1_1`和`mtr_T1_2`。
- 事务`T2`的两个`mtr`分别称为`mtr_T2_1`和`mtr_T2_2`

每个mtr都会产生一组redo日志，用示意图来描述一下这些mtr产生的日志情况：

![image-20240925125053236](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f396ad8862c.png)

不同的事务可能是 `并发` 执行的，所以 T1 、 T2 之间的 mtr 可能是 `交替执行` 的。每当一个mtr执行完成时，伴随该mtr生成的一组redo日志就需要被复制到log buffer中，也就是说不同事务的mtr可能是交替写入log buffer的，我们画个示意图（为了美观，我们把一个mtr中产生的所有redo日志当做一个整体来画）：

![image-20240925125245082](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f3971d5b243.png)

有的mtr产生的redo日志量非常大，比如`mtr_t1_2`产生的redo日志占用空间比较大，占用了3个block来存储。

**redo log block的结构图**

一个redo log block是由`日志头、日志体、日志尾`组成。日志头占用12字节，日志尾占用8字节，所以一个block真正能存储的数据是512-12-8=492字节。

> **为什么一个block设计成512字节？**
>
> 这个和磁盘的扇区有关，机械磁盘默认的扇区就是512字节，如果你要写入的数据大于512字节，那么要写入 的扇区肯定不止一个，这时就要涉及到盘片的转动，找到下一个扇区，假设现在需要写入两个扇区A和B，如果扇区A写入成功，而扇区B写入失败，那么就会出现`非原子性`的写入，而如果每次只写入和扇区的大小一样 的512字节，那么每次的写入都是原子性的。

![image-20240925125439200](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f3978f76c13.png)

真正的redo日志都是存储到占用`492`字节大小的`log block body`中，图中的`log block header`和`log block trailer`存储的是一些管理信息。我们来看看这些所谓`管理信息`都有什么。

![image-20240925125506782](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f397ab122a6.png)

- log block header
  
  属性介绍如下：

  - `LOG_BL0CK_HDR_NO`: log buffer是由log block组成，在内部log buffer就好似一个数组，因此 LOG_BLOCK_HDR_NO 用来标记这个数组中的位置。其是递增并且循环使用的，占用4个字节，但是由于第一位用来判断是否是flush bit，所以最大的值为2G。
  - `L0G_BL0CK_HDR_DATA_LEN`: 表示block中已经使用了多少字节，初始值为`12`（因为`log block body` 从第12个字节处开始)。随着往block中写入的redo日志越来也多，本属性值也跟着增长。如果`log block body `已经被全部写满，那么本属性的值被设置为`512`。
  - `LOG_BLOCK_FIRST_REC_GROUP`: 一条redo日志也可以称之为一条redo日志记录(redo log record)，一个mtr会生产多条redo日志记录，这些redo日志记录被称之为一个redo日志记录组(redo log record group)。LOG_BLOCK_FIRST_REC_GROUP 就代表该block中第一个mtr生成的redo日志记录组的偏移量（其实也就是这个block里第一个mtr生成的第一条redo日志的偏移量)。如果该值的大小和LOG_BLOCK_HDR_DATA_LEN相同，则表示当前log block不包含新的日志。
  - `L0G_BL0CK_CHECKPOINT_No`：占用4字节，表示该log block最后被写时的`checkpoint`
  
- log block trailer
  
  属性介绍如下：

  - `LOG_BLOCK_CHECKSUM`: 表示block的校验值，用于正确性校验（其值和LOG_BLOCK_HDR_NO相同），我们暂时不关心它。

**redo log file**

**相关参数设置**

- `innodb_log_group_home_dir` ：指定 redo log 文件组所在的路径，默认值为 `./` ，表示在数据库 的数据目录下。MySQL的默认数据目录（ `var/lib/mysql`）下默认有两个名为 `ib_logfile0` 和 `ib_logfile1` 的文件，log buffer中的日志默认情况下就是刷新到这两个磁盘文件中。此redo日志 文件位置还可以修改。

- `innodb_log_files_in_group`：指明redo log file的个数，命名方式如：ib_logfile0，iblogfile1... iblogfilen。默认2个，最大100个。

  ```
  mysql> show variables like 'innodb_log_files_in_group';
  +---------------------------+-------+
  | Variable_name             | Value |
  +---------------------------+-------+
  | innodb_log_files_in_group | 2     |
  +---------------------------+-------+
  #ib_logfile0
  #ib_logfile1
  ```

- `innodb_flush_log_at_trx_commit`：控制 redo log 刷新到磁盘的策略，默认为1。

- `innodb_log_file_size`：单个 redo log 文件设置大小，默认值为 `48M` 。最大值为512G，注意最大值指的是整个 redo log 系列文件之和，即（innodb_log_files_in_group * innodb_log_file_size ）不能大于最大值512G。

  ```
  mysql> show variables like 'innodb_log_file_size';
  +----------------------+----------+
  | Variable_name        | Value    |
  +----------------------+----------+
  | innodb_log_file_size | 50331648 |
  +----------------------+----------+
  ```

根据业务修改其大小，以便容纳较大的事务。编辑my.cnf文件并重启数据库生效，如下所示

```
[root@localhost ~]# vim /etc/my.cnf
innodb_log_file_size=200M
```

> 在数据库实例更新比较频繁的情况下，可以适当加大 redo log 数组和大小。但也不推荐 redo log 设置过大，在MySQL崩溃时会重新执行REDO日志中的记录。

**日志文件组**

从上边的描述中可以看到，磁盘上的`redo`日志文件不只一个，而是以一个`日志文件组`的形式出现的。这些文件以`ib_logfile[数字]`（数字可以是0、1、2)的形式进行命名，每个的redo日志文件大小都是一样的。

在将redo日志写入日志文件组时，是从`ib_logfile0`开始写，如果`ib_1ogfile0`写满了，就接着`ib_1ogfile1` 写。同理，`ib_logfi1e1`写满了就去写`ib_1ogfile2`，依此类推。如果写到最后一个文件该昨办？那就重新转 到`ib_1ogfile0`继续写，所以整个过程如下图所示：

![image-20240925130832626](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f39ad0defa0.png)

总共的redo日志文件大小其实就是： `innodb_log_file_size × innodb_log_files_in_group` 。

采用循环使用的方式向redo日志文件组里写数据的话，会导致后写入的redo日志覆盖掉前边写的redo日志？当然！所以InnoDB的设计者提出了`checkpoint`的概念。

**checkpoint**

在整个日志文件组中还有两个重要的属性，分别是 `write pos`、`checkpoint`

- `write pos`是当前记录的位置，一边写一边后移
- `checkpoint`是当前要擦除的位置，也是往后推移

每次刷盘 redo log 记录到日志文件组中，write pos 位置就会后移更新。每次MySQL加载日志文件组恢复数据时，会清空加载过的 redo log 记录，并把check point后移更新。write pos 和 checkpoint 之间的还空着的部分可以用来写入新的 redo log 记录。

![image-20240925131007125](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f39b2f5f1c6.png)

如果 write pos 追上 checkpoint ，表示`日志文件组`满了，这时候不能再写入新的 redo log记录，MySQL得停下来，清空一些记录，把 checkpoint 推进一下。



### 7.3 Bin Log

#### 7.3.1 概念

​	`bin log`又叫**二进制日志**，是**逻辑日志**，记录内容是语句的原始逻辑，**属于`MySQL Server`层**。**所有的存储引擎**只要发生了数据更新，都会产生`bin log`日志，`bin log`会记录所有涉及更新数据的逻辑规则，并且**按顺序写**。可以说`MySQL`数据库的数据备份、主备、主主、主从都离不开`bin log`，需要依赖`bin log`来同步数据，保证数据一致性，如下图所示。

![img](https://segmentfault.com/img/remote/1460000041758796)

#### 7.3.2 写入机制

​	`binlog`的写入时机为事务执行过程中，先把日志写到`binlog cache`，事务提交的时候再把`binlog cache`写到`binlog`文件中（实际先会写入`page cache`，然后再由`fsync`写入`binlog`文件）。

​	因为一个事务的`binlog`不能被拆开，无论这个事务多大，也要确保一次性写入，所以系统会给每个线程分配一块内存作为`binlog cache`。可以通过`binlog_cache_size`参数控制单线程`binlog_cache`大小，如果存储内容超过了这个参数，就要暂存到磁盘。

#### 7.3.3 刷盘时机

​	`binlog`日志刷盘流程如下：

![img](https://segmentfault.com/img/remote/1460000041758799)

- 上图的`write`，是指把日志写入到文件系统的`page cache`，并没有把数据持久化硬盘，所以速度比较快。
- 上图的` fsync`才是将数据库持久化到硬盘的操作。

​	`write`和`fsync`的时机可以由参数`sync_binlog`控制，可以配置成`0、1、N(N>1)`。

- **sync_binlog = 0**：MySQL 不强制将`binlog`数据从缓存同步到磁盘，而是依赖操作系统的机制来决定何时将缓存中的数据写入磁盘，只把日志写入`page cache`，虽然性能得到了提高，但是事务提交了`fsync`的时候宕机了，可能造成`binlog`日志的丢失。

  ![img](https://segmentfault.com/img/remote/1460000041758800)

- **sync_binlog = 1**：每次事务提交时，MySQL 都会调用 `fsync` 操作，将 binlog 缓存中的数据同步到磁盘。这是最安全的设置，但性能损耗最大。

- **sync_binlog = N (N > 1)**：每提交 N 个事务后，MySQL 调用一次 `fsync` 操作，将 binlog 缓存中的数据同步到磁盘，将`sync_binlog`设置成一个比较大的值，可以提升性能，但是如果机器宕机，会丢失最近`N`个事务的`binlog`日志。这种方式在性能和安全性之间做了平衡，如下图。

![img](https://segmentfault.com/img/remote/1460000041758801)

#### 7.3.4 存储结构

`binlog`日志有三种格式，可以通过`binlog_format`参数设置，有以下三种：

- statement
- row
- mixed

##### 7.3.3.1 statement

​	`statement`记录的内容是`SQL`语句原文，比如执行一条`update T set update_time = now() where id = 1`，记录内容如下：
![img](https://segmentfault.com/img/remote/1460000041758797)

​	同步数据时，会执行记录的`SQL`语句，但是有个问题`update_time = now()`这里会获取到当前系统问题，直接执行会导致与原库数据不一致。

##### 7.3.3.2 row

​	`row`不再是简单的`SQL`语句了，还包含了操作的具体数据，但是记录的内容看不到详细信息，需要通过`mysqlbinlog`工具解析出来，这样子就解决了statement格式的问题，我们需要将`binlog_format`设置成`row`，记录的，记录内容如下：

![img](https://segmentfault.com/img/remote/1460000041758798)

​	`update_time = now()`变成了具体的时间，条件后面的`@1、@2`都是该行数据第1个~2个字段的原始值（假设这张表只有2个字段）。

​	设置成`row`带来的好处就是**同步数据的一致性**，通常情况都设置成`row`，这样可以为数据库的恢复与同步带来更好的可靠性。但是这种格式需要大量的容量来记录，比较**占用空间**，恢复与同步时会**更消耗`IO`资源**，影响执行速度。

##### 7.3.3.3 mixed

​	`mixed`记录的内容是前两者的混合。`MySQL`会判断这条`SQL`语句是否会引起数据不一致，如果是就用`row`格式，否则就用`statement`格式。



#### 7.3.5 两阶段提交

在执行更新语句过程，会记录redo log与binlog两块日志，以基本的事务为单位，redo log在事务执行过程中可以不断写入，而binlog只有在提交事务时才写入，所以redo log与binlog的 `写入时机` 不一样。

![image-20240926154149931](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f5103e6eaa1.png)

**redo log与binlog两份日志之间的逻辑不一致，会出现什么问题？**

以update语句为例，假设`id=2`的记录，字段`c`值是`0`，把字段c值更新为`1`，SQL语句为update T set c = 1 where id = 2。

假设执行过程中写完redo log日志后，binlog日志写期间发生了异常，会出现什么情况呢？

![image-20220715195016492](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f50f1896320.png)

由于binlog没写完就异常，这时候binlog里面没有对应的修改记录。因此，之后用binlog日志恢复数据时，就会少这一次更新，恢复出来的这一行c值为0，而原库因为redo log日志恢复，这一行c的值是1，最终数据不一致。

![image-20220715195521986](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f50f18d3090.png)

为了解决两份日志之间的逻辑一致问题，InnoDB存储引擎使用**两阶段提交**方案。原理很简单，将redo log的写入拆成了两个步骤prepare和commit，这就是**两阶段提交**。

![image-20220715195635196](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f50f1943df9.png)

使用两阶段提交后，写入binlog时发生异常也不会有影响，因为MySQL根据redo log日志恢复数据时，发现redo log还处于prepare阶段，并且没有对应binlog日志，就会回滚该事务。

![image-20220715200248193](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f50f19a4fe9.png)

另一个场景，redo log设置commit阶段发生异常，那会不会回滚事务呢？

![image-20220715200321717](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f50f1985230.png)

并不会回滚事务，它会执行上图框住的逻辑，虽然redo log是处于prepare阶段，但是能通过事务id找到对应的binlog日志，所以MySQL认为是完整的，就会提交事务恢复数据。



### 7.4 Undo Log

#### 7.4.1 概念

​	`undo log`是事务**原子性**的保证，是一个**逻辑日志**。在事务中`更新数据`的`前置操作`其实是要先写入一个`undo log`。此外，**`undo log`会产生`redo log`**，也就是`undo log`的产生会伴随着`redo log`的产生，这是因为`undo log`也需要`持久性`的保护。

#### 7.4.2 作用

##### 7.4.2.1 回滚数据

​	`undo log`是**逻辑日志**，因此只是将数据库逻辑地恢复到原来的样子。所有修改都被逻辑地取消了，但是**数据结构和页本身在回滚之后可能大不相同**。

> 这是因为在多用户并发系统中，可能会有数十、数百甚至数千个并发事务。数据库的主要任务就是协调对数据记录的并发访问。比如，一个事务在修改当前一个页中某几条记录，同时还有别的事务在对同一个页中另几条记录进行修改。因此，不能将一个页回滚到事务开始的样子，因为这样会影响其他事务正在进行的工作。

##### 7.4.2.2 MVCC

​	undo log的另一个作用是MVCC，即在InnoDB存储引擎中MVCC的实现是通过undo来完成。当用户读取一行记录时，若该记录以及被其他事务占用，当前事务可以通过undo读取之前的行版本信息，以此实现非锁定读取。

#### 7.4.3 生命周期

##### 7.4.3.1 生成过程

>**隐藏字段**
>
>对于InnoDB引擎来说，每个行记录除了记录本身的数据之外，还有几个隐藏的列：
>
>- `DB_ROW_ID`: 如果没有为表显式的定义主键，并且表中也没有定义唯一索引，那么InnoDB会自动为表添加一 个row_id的隐藏列作为主键。
>- `DB_TRX_ID`: 每个事务都会分配一个事务ID,当对某条记录发生变更时，就会将这个事务的事务ID写入trx_id中。
>- `DB_ROLL_PTR`:回滚指针，本质上就是指向undo log的指针。
>
>![image-20240925140424684](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f3a7e8dfb55.png)

**当我们执行INSERT时：**

```
begin;
INSERT INTO user (name) VALUES ("tom");
```

插入的数据都会生成一条insert undo log，并且数据的回滚指针会指向它。undo log会记录undo log的序号、插入主键的列和值...，那么在进行rollback的时候，通过主键直接把对应的数据删除即可。

![image-20240925140530862](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f3a82b16488.png)

**当我们执行UPDATE时：**

对应更新的操作会产生update undo log，并且会分更新主键和不更新主键的，假设现在执行：

```
UPDATE user SET name="Sun" WHERE id=1;
```

![image-20240925140607905](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f3a8501faf7.png)

这时会把老的记录写入新的undo log，让回滚指针指向新的undo log，它的undo no是1，并且新的undo log会指向老的undo log（undo no=0）。

假设现在执行：

```
UPDATE user SET id=2 WHERE id=1;
```

![image-20240925140825594](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f3a8d9cd721.png)

对于更新主键的操作，会先把原来的数据`deletemark`标识打开，这时并没有真正的删除数据，真正的删除会交给清理线程去判断，然后在后面插入一条新的数据，新的数据也会产生undo log，并且undo log的序号会递增。

可以发现每次对数据的变更都会产生一个undo log，当一条记录被变更多次时，那么就会产生多条undo log，undo log记录的是变更前的日志，并且每个undo log的序号是递增的，那么当要回滚的时候，按照序号`依次向前推`，就可以找到我们的原始数据了。

##### 7.4.3.2 回滚过程

以上面的例子来说，假设执行rollback，那么对应的流程应该是这样：

1. 通过undo no=3的日志把id=2的数据删除
2. 通过undo no=2的日志把id=1的数据的deletemark还原成0
3. 通过undo no=1的日志把id=1的数据的name还原成Tom
4. 通过undo no=0的日志把id=1的数据删除

##### 7.4.3.3 删除过程

- 针对于insert undo log

  因为insert操作的记录，只对事务本身可见，对其他事务不可见。故该undo log可以在事务提交后直接删除，不需要进行purge操作。

- 针对于update undo log

  该undo log可能需要提供MVCC机制，因此不能在事务提交时就进行删除。提交时放入undo log链表，等待purge线程进行最后的删除。

#### 7.4.3 存储结构

**回滚段与undo页**

InnoDB对undo log的管理采用段的方式，也就是 `回滚段（rollback segment）` 。每个回滚段记录了 `1024` 个 `undo log segment` ，而在每个undo log segment段中进行 `undo页` 的申请。

- 在` InnoDB1.1版本之前` （不包括1.1版本），只有一个rollback segment，因此支持同时在线的事务限制为 `1024` 。虽然对绝大多数的应用来说都已经够用。
- 从1.1版本开始InnoDB支持最大 `128个rollback segment` ，故其支持同时在线的事务限制提高到 了 `128*1024` 。

```
mysql> show variables like 'innodb_undo_logs';
+------------------+-------+
| Variable_name    | Value |
+------------------+-------+
| innodb_undo_logs | 128   |
+------------------+-------+
```

虽然InnoDB.1.1版本支持了128个rollback segment,但是这些rollback segment都存储于共享表空间ibdata中。

从 InnoDB1.2版本开始，可通过参数对rollback segment做进一步的设置。这些参数包括：

- `innodb_undo_directory`: 设置rollback segment文件所在的路径。这意味着rollback segment可以存放在共享表空间以外的位置，即可以设置为独立表空间。该参数的默认值为`./`，表示当前InnoDB存储引擎的目
- `innodb_undo_logs`: 设置rollback segment的个数，默认值为128。在InnoDB1.2版本中，该参数用来替换之前版本的参数innodb_rollback_segments。
- `innodb_undo_tablespaces`：设置构成rollback segment文件的数量，这样rollback segment可以较为平均地 分布在多个文件中。设置该参数后，会在路径innodb_undo_directory看到undo为前缀的文件，该文件就代表 rollback segment文件。

undo log相关参数一般很少改动。

**undo页的重用**

当我们开启一个事务需要写undo log的时候，就得先去undo log segment中去找到一个空闲的位置，当有空位的时候，就去申请undo页，在这个申请到的undo页中进行undo log的写入。我们知道mysql默认一页的大小是16k。

> 为每一个事务分配一个页，是非常浪费的（除非你的事务非常长），假设你的应用的TPS(每秒处理的事务数目)为1000，那么1s就需要1000个页，大概需要16M的存储，1分钟大概需要1G的存储。如果照这样下去除非MySQL清理的非常勤快，否则随着时间的推移，磁盘空间会增长的非常快，而且很多空间都是浪费的。

于是undo页就被设计的`可以重用`了，当事务提交时，并不会立刻删除undo页。因为重用，所以这个undo页可能混杂着其他事务的undo log。undo log在commit后，会被放到一个`链表`中，然后判断undo页的使用空间是否小于`3/4`,如果小于3/4的话，则表示当前的undo页可以被重用，那么它就不会被回收，其他事务的undo log可以记录 在当前undo页的后面。由于undo log是`离散的`，所以清理对应的磁盘空间时，效率不高。

**回滚段与事务**

1. 每个事务只会使用一个回滚段，一个回滚段在同一时刻可能会服务于多个事务。

2. 当一个事务开始的时候，会制定一个回滚段，在事务进行的过程中，当数据被修改时，原始的数据会被复制到回滚段。

3. 在回滚段中，事务会不断填充盘区，直到事务结束或所有的空间被用完。如果当前的盘区不够用，事务会在段中请求扩展下一个盘区，如果所有已分配的盘区都被用完，事务会覆盖最初的盘区或者在回滚段允许的情况下扩展新的盘区来使用。

4. 回滚段存在于undo表空间中，在数据库中可以存在多个undo表空间，但同一时刻只能使用一个 undo表空间。

   ```
   mysql> show variables like 'innodb_undo_tablespaces';
   +-------------------------+-------+
   | Variable_name           | Value |
   +-------------------------+-------+
   | innodb_undo_tablespaces | 2     |
   +-------------------------+-------+
   # undo log的数量，最少为2. undo log的truncate操作有purge协调线程发起。在truncate某个undo log表空间的过程中，保证有一个可用的undo log可用。
   ```

5. 当事务提交时，InnoDB存储引擎会做以下两件事情：

   - 将undo log放入列表中，以供之后的purge操作
   - 判断undo log所在的页是否可以重用，若可以则分配给下个事务使用

**回滚段中的数据分类**

1. `未提交的回滚数据(uncommitted undo information)`：该数据所关联的事务并未提交，用于实现读一致性，所以该数据不能被其他事务的数据覆盖。
2. `已经提交但未过期的回滚数据(committed undo information)`：该数据关联的事务已经提交，但是仍受到undo retention参数的保持时间的影响。
3. `事务已经提交并过期的数据(expired undo information)`：事务已经提交，而且数据保存时间已经超过 undo retention参数指定的时间，属于已经过期的数据。当回滚段满了之后，就优先覆盖“事务已经提交并过期的数据"。

事务提交后不能马上删除undo log及undo log所在的页。这是因为可能还有其他事务需要通过undo log来得到行记录之前的版本。故事务提交时将undo log放入一个链表中，是否可以最终删除undo log以undo log所在页由purge线程来判断。



## 8. 多版本并发控制（MVCC）

### 8.1 概述

MVCC （Multiversion Concurrency Control），多版本并发控制。MVCC 是通过数据行的多个版本管理来实现数据库的 `并发控制 `。这项技术使得在InnoDB的事务隔离级别下执行 `一致性读` 操作有了保证。换言之，就是为了查询一些正在被另一个事务更新的行，并且可以看到它们被更新之前的值，这样在做查询的时候就不用等待另一个事务释放锁。

MVCC 的实现依赖于：`隐藏字段`、`Undo Log`、`Read View`。

隐藏字段帮助存储记录相关的事务信息，包括最近更新数据的事务id和指向Undo Log版本链的指针，Undo Log保存了历史快照，通过产生该日志的事务的ID进行区别，Read View规则帮我们判断当前版本的数据是否可见。

### 8.2 相关概念

#### 8.2.1 隐藏字段

[[隐藏字段介绍](#####7.4.3.1 生成过程)]

#### 8.2.2 Undo Log

[[Undo Log生成结构介绍](#####7.4.3.1 生成过程)]

#### 8.2.3 ReadView

##### 8.2.3.1 概述

在MVCC机制中，多个事务对同一个行记录进行更新会产生多个历史快照，这些历史快照保存在Undo Log里。如果一个事务想要查询这个行记录，需要读取哪个版本的行记录呢？这时就需要用到ReadView了，**它帮我们解决了行的可见性问题。**

**ReadView就是某一具体事务在使用MVCC机制进行快照读操作时产生的读视图**。当事务启动时，会生成数据库系统当前的一个快照，InnoDB为每个事务构造了一个数组，用来记录并维护系统当前`活跃事务`的ID(“活跃”指的就是，启动了但还没提交)。

##### 8.2.3.2 结构

> **活跃的事务**
>
> “活跃”指的就是，启动了但还没提交

这个ReadView中主要包含4个比较重要的内容，分别如下：

1. `creator_trx_id` ，创建这个 Read View 的事务 ID。

   > 说明：只有在对表中的记录做改动时（执行INSERT、DELETE、UPDATE这些语句时）才会为 事务分配事务id，否则在一个只读事务中的事务id值都默认为0。

2. `trx_ids` ，表示在生成ReadView时当前系统中活跃的读写事务的 `事务id列表`。

3. `up_limit_id` ，活跃的事务中最小的事务 ID。

4. `low_limit_id` ，表示生成ReadView时系统中应该分配给下一个事务的 id 值。low_limit_id 是**系统最大的事务id值**，这里要注意是系统中的事务id，需要区别于正在活跃的事务ID。

##### 8.2.3.3 生成时机

-  每当一个`REPEATABLE READ`事务启动时，只在第一次 SELECT 的时候会获取一次 Read View，而后面所有的 SELECT 都会复用这个 Read View。

  ![image-20240926122418461](https://gitee.com/an_shiguang/learn-mysql/raw/master/4_notes/images/66f4e1f2d50ce.png)

- 在`READ COMMITED`事务中，每一次 SELECT 查询都会生成一个专属于该查询操作的ReadView。 

  如表所示：

  | 事务                                | 说明             |
  | ----------------------------------- | ---------------- |
  | begin;                              |                  |
  | select * from student where id>2;   | 获取一次ReadView |
  | ...                                 |                  |
  | select * from student where id > 2; | 获取一次ReadView |
  | commit;                             |                  |

  > 注意，此时同样的查询语句都会重新获取一次 Read View，这时如果 Read View 不同，就可能产生不可重复读或者幻读的情况。



### 8.3 设计思路

使用 `READ UNCOMMITTED` 隔离级别的事务，由于可以读到未提交事务修改过的记录，所以直接读取记录的最新版本就好了。

使用 `SERIALIZABLE` 隔离级别的事务，InnoDB规定使用加锁的方式来访问记录。

使用 `READ COMMITTED` 和 `REPEATABLE READ` 隔离级别的事务，都必须保证读到 `已经提交了的` 事务修改过的记录。假如另一个事务已经修改了记录但是尚未提交，是不能直接读取最新版本的记录的，核心问题就是需要判断一下版本链中的哪个版本是当前事务可见的，这是ReadView要解决的主要问题，具体判断规则如下：

- 如果被访问版本的trx_id属性值与ReadView中的 creator_trx_id 值相同，意味着当前事务在访问它自己修改过的记录，所以该版本可以被当前事务访问。
- 如果被访问版本的trx_id属性值小于ReadView中的 up_limit_id 值，表明生成该版本的事务在当前事务生成ReadView前已经提交，所以该版本可以被当前事务访问。
- 如果被访问版本的trx_id属性值大于或等于ReadView中的 low_limit_id 值，表明生成该版本的事务在当前事务生成ReadView后才开启，所以该版本不可以被当前事务访问。
- 如果被访问版本的trx_id属性值在ReadView的 up_limit_id 和 low_limit_id 之间，那就需要判断一下trx_id属性值是不是在 trx_ids 列表中。
  - 如果在，说明创建ReadView时生成该版本的事务还是活跃的，该版本不可以被访问。
  - 如果不在，说明创建ReadView时生成该版本的事务已经被提交，该版本可以被访问。



### 8.4 整体流程

（1）首先获取事务自己的版本号，也就是事务 ID；

（2）获取 ReadView；

（3）查询得到的数据，然后与 ReadView 中的事务版本号进行比较；

（4）如果不符合 ReadView 规则，就需要从 Undo Log 中获取历史快照；

（5）最后返回符合规则的数据。

如果某个版本的数据对当前事务不可见的话，那就顺着版本链找到下一个版本的数据，继续按照上边的步骤判断 可见性，依此类推，直到版本链中的最后一个版本。如果最后一个版本也不可见的话，那么就意味着该条记录对 该事务完全不可见，查询结果就不包含该记录。























