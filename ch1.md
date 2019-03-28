## 第一章  数据库集群、数据库和表

本章和下一章总结了PostgreSQL的基本知识，以帮助阅读后续章节。 在本章中，将描述以下主题：

- 数据库集群的逻辑结构
- 数据库集群的物理结构
- 堆表文件的内部结构
- 向表写入和读取数据的方法

如果你已经熟悉它们，你可以跳过本章。

## 1.1. 数据库集群的逻辑结构

**数据库集群**是由单个PostgreSQL服务器实例管理的数据库的集合。 如果您现在第一次听到这个定义，您可能会对此感到疑惑，但PostgreSQL中的术语“数据库集群”并**不**意味着“一组数据库服务器”。 PostgreSQL服务器在单个主机上运行并管理单个数据库群集。

图1.1显示了数据库集群的逻辑结构。 数据库是数据库对象的集合。 在关系数据库理论中，数据库对象是用于存储或引用数据的数据结构。它的一个典型示例就是表(table)，并且还有很多像索引，序列，视图，函数等等。 在PostgreSQL中，数据库本身也是数据库对象，并且在逻辑上彼此分离。 所有其他数据库对象（例如表，索引等）都属于它们各自的数据库。

**图. 1.1. 数据库集群的逻辑结构**

![Fig. 1.1. Logical structure of a database cluster.](imgs/ch1/fig-1-01.png)

PostgreSQL中的所有数据库对象都由各自的**对象标识符(OID)**进行内部管理，它们是无符号的4字节整数。 数据库对象和各个OID之间的关系存储在适当的[系统目录](http://www.postgresql.org/docs/current/static/catalogs.html)中，具体取决于对象的类型。 例如，数据库和堆表的OID分别存储在*pg_database*和*pg_class*中，因此您可以通过发出如下查询来找出想要知道的OID：

```sql
sampledb=# SELECT datname, oid FROM pg_database WHERE datname = 'sampledb';
 datname  |  oid  
----------+-------
 sampledb | 16384
(1 row)

sampledb=# SELECT relname, oid FROM pg_class WHERE relname = 'sampletbl';
  relname  |  oid  
-----------+-------
 sampletbl | 18740 
(1 row)
```

## 1.2. 数据库集群的物理结构

*数据库集群*基本上是一个称为**数据目录**的目录，它包含一些子目录和大量文件。 如果您执行[initdb](http://www.postgresql.org/docs/current/static/app-initdb.html)来初始化新的数据库群集，则将在指定的目录下创建一个数据目录。 虽然它不是强制性的，但是数据目录的路径通常在环境变量*PGDATA*设置。

图1.2展示了PostgreSQL数据库集群的一个例子。一个数据库实例是base子目录下的子目录，每个表和索引(至少)是存储在其所属数据库子目录下的一个文件。数据目录下还有几个包含特定数据和配置文件的子目录。 虽然PostgreSQL支持表空间，但该术语的含义与其他RDBMS不同。PostgreSQL中的表空间是一个目录，其中包含数据目录之外的一些数据。 

**图 1.2. 数据库集群示例**

![Fig. 1.2. An example of database cluster.](imgs/ch1/fig-1-02.png)

在以下小节中，将描述数据库集群、数据库、与表和索引相关文件、表空间的结构。

### 1.2.1. 数据库集群的目录结构

数据库集群的目录结构已经在[官方文档](http://www.postgresql.org/docs/current/static/storage-file-layout.html)中进行了描述。表1.1中列出了文档中一部分的主要的文件和子目录： 

**表 1.1: 数据目录下的文件和子目录（来自官方文档）**

| **文件**                          | **描述**                                                     |
| --------------------------------- | ------------------------------------------------------------ |
| PG_VERSION                        | 包含PostgreSQL的主要版本号的文件                             |
| pg_hba.conf                       | 控制PosgreSQL的客户端身份验证的文件                          |
| pg_ident.conf                     | 控制PostgreSQL的用户名映射的文件                             |
| postgresql.conf                   | 用于设置配置参数的文件                                       |
| postgresql.auto.conf              | 用于存储在ALTER SYSTEM(9.4版本或更高版本)中设置的配置参数的文件 |
| postmaster.opts                   | 记录服务器上次启动的命令行选项的文件                         |
| **子目录**                        | **描述**                                                     |
| base/                             | 包含每个数据库实例子目录的子目录                             |
| global/                           | 包含一些共享系统表的子目录，例如pg_database和pg_control      |
| pg_commit_ts/                     | 包含事务提交时间戳数据的子目录。 9.5版本或更高版本           |
| pg_clog/ (Version 9.6 or earlier) | 包含事务提交状态数据的子目录。 它在10版本中重命名为pg_xact。CLOG将在5.4节中描述 |
| pg_dynshmem/                      | 包含动态共享内存子系统使用的文件的子目录。 9.4版本或更高版本 |
| pg_logical/                       | 包含逻辑解码状态数据的子目录。 9.4版本或更高版本             |
| pg_multixact/                     | 包含多事务状态数据的子目录（用于共享行锁）                   |
| pg_notify/                        | 包含LISTEN/NOTIFY状态数据的子目录                            |
| pg_repslot/                       | 包含[复制槽](http://www.postgresql.org/docs/current/static/warm-standby.html#STREAMING-REPLICATION-SLOTS)数据的子目录。 9.4版本或更高版本。 |
| pg_serial/                        | 包含有关已提交可序列化事务（9.1或更高版本）信息的子目录      |
| pg_snapshots/                     | 包含导出快照的子目录（9.2版本或更高版本）。 PostgreSQL的函数pg_export_snapshot在这个子目录中创建一个快照信息文件 |
| pg_stat/                          | 包含统计子系统永久文件的子目录                               |
| pg_stat_tmp/                      | 包含统计子系统临时文件的子目录                               |
| pg_subtrans/                      | 包含子事务状态数据的子目录                                   |
| pg_tblspc/                        | 包含表空间符号链接的子目录                                   |
| pg_twophase/                      | 包含准备事务的状态文件的子目录                               |
| pg_wal/ (Version 10 or later)     | 包含WAL（预写日志记录）段文件的子目录。 它在版本10中从pg_xlog重命名 |
| pg_xact/ (Version 10 or later)    | 包含事务提交状态数据的子目录。 它在版本10中从pg_clog更名。CLOG将在5.4节中描述 |
| pg_xlog/ (Version 9.6 or earlier) | 包含WAL（预写日志记录）段文件的子目录。 它在版本10中重命名为pg_wal |

### 1.2.2. 数据库实例的目录结构

一个数据库实例是*base*子目录下的子目录; 并且数据库实例的目录名称与相应的OID相同。 例如，当数据库sampledb的OID为16384时，其子目录名称为16384。

```shell
$ cd $PGDATA
$ ls -ld base/16384
drwx------  213 postgres postgres  7242  8 26 16:33 16384
```

### 1.2.3.与表和索引相关文件的目录结构

每个大小小于1GB的表或索引都是存储在其所属数据库目录下的单个文件。 作为数据库对象的表和索引由单个OID内部管理，而这些数据文件由变量relfilenode管理。 表和索引的relfilenode值基本上并**不**总是与各自的OID匹配，详细信息如下所述。

我们来看一下表sampletbl的OID和relfilenode：

```sql
sampledb=# SELECT relname, oid, relfilenode FROM pg_class WHERE relname = 'sampletbl';
  relname  |  oid  | relfilenode
-----------+-------+-------------
 sampletbl | 18740 |       18740 
(1 row)
```

从上面的结果中，可以看到oid和relfilenode值是相等的。 您还可以看到表sampletbl的数据文件路径是*'base/16384/18740'*。

```shell
$ cd $PGDATA
$ ls -la base/16384/18740
-rw------- 1 postgres postgres 8192 Apr 21 10:21 base/16384/18740
```

通过发出一些命令（例如，TRUNCATE，REINDEX，CLUSTER）来更改表和索引的relfilenode值。 例如，如果我们truncate表sampletbl，PostgreSQL会为表分配一个新的relfilenode（18812），删除旧的数据文件（18740）并创建一个新的（18812）。

```sql
sampledb=# TRUNCATE sampletbl;
TRUNCATE TABLE

sampledb=# SELECT relname, oid, relfilenode FROM pg_class WHERE relname = 'sampletbl';
  relname  |  oid  | relfilenode
-----------+-------+-------------
 sampletbl | 18740 |       18812 
(1 row)
```

在9.0或更高版本中，内置函数*pg_relation_filepath*非常有用，因为此函数会返回指定OID或名称的*relation*的文件路径名。

```sql
sampledb=# SELECT pg_relation_filepath('sampletbl');
 pg_relation_filepath 
----------------------
 base/16384/18812
(1 row)
```

当表和索引的文件大小超过1GB时，PostgreSQL会创建一个名为relfilenode.1的新文件并使用它。 如果新文件已经填满，则会创建名为relfilenode.2的下一个新文件，依此类推。

```shell
$ cd $PGDATA
$ ls -la -h base/16384/19427*
-rw------- 1 postgres postgres 1.0G  Apr  21 11:16 data/base/16384/19427
-rw------- 1 postgres postgres  45M  Apr  21 11:20 data/base/16384/19427.1
...
```

在构建PostgreSQL时可以使用--with-segsize配置选项，更改表和索引的最大文件大小。

仔细查看数据库子目录，您会发现每个表都有两个相关的文件，分别以'fsm'和'vm'作为后缀。 这些被称为**空闲空间映射表(free space map)**和**可见性映射表(visibility map)**，分别在表文件中存储每个页面的空闲空间容量和可见性的信息(更详细的信息见第5.3.4节和第6.2节)。索引只有空闲空间映射，没有可见性映射。

具体示例如下所示：

```shell
$ cd $PGDATA
$ ls -la base/16384/18751*
-rw------- 1 postgres postgres  8192 Apr 21 10:21 base/16384/18751
-rw------- 1 postgres postgres 24576 Apr 21 10:18 base/16384/18751_fsm
-rw------- 1 postgres postgres  8192 Apr 21 10:18 base/16384/18751_vm
```

They may also be internally referred to as the **forks** of each relation; the free space map is the first fork of the table/index data file (the fork number is 1), the visibility map the second fork of the table's data file (the fork number is 2). The fork number of the data file is 0.

### 1.2.4. 表空间

PostgreSQL中的表空间是数据目录之外的附加数据区域。 该功能已在8.0版本中实现。

图1.3显示了表空间的内部结构以及与数据目录的关系。

**Fig. 1.3. 数据库集群中的表空间**

![Fig. 1.3. A Tablespace in the Database Cluster.](imgs/ch1/fig-1-03.png)

使用[CREATE TABLESPACE](http://www.postgresql.org/docs/current/static/sql-createtablespace.html)语句时指定的目录下会创建一个表空间，在该目录下将创建特定于版本的子目录（例如，PG_9.4_201409291）。 下面显示了特定于版本的命名方法。

```shell
PG _ 'Major version' _ 'Catalogue version number'
```

例如，如果您在 *'/home/postgres/tblspc'*处创建表空间*'new_tblspc'*（其oid为16386），则将在该表空间下创建一个子目录（如“PG_9.4_201409291”）。

```shell
$ ls -l /home/postgres/tblspc/
total 4
drwx------ 2 postgres postgres 4096 Apr 21 10:08 PG_9.4_201409291
```

表空间目录由来自pg_tblspc子目录的符号链接寻址，并且链接名称与表空间的OID值相同。

```shell
$ ls -l $PGDATA/pg_tblspc/
total 0
lrwxrwxrwx 1 postgres postgres 21 Apr 21 10:08 16386 -> /home/postgres/tblspc
```

如果您在表空间下创建新数据库（OID为16387），则其目录将在版本特定的子目录下创建。

```shell
$ ls -l /home/postgres/tblspc/PG_9.4_201409291/
total 4
drwx------ 2 postgres postgres 4096 Apr 21 10:10 16387
```

如果您创建了新表，该表属于在数据目录下创建的数据库，则首先在*PG_SERVERVISION*子目录下创建名称与现有数据库OID相同的新目录，然后放置新的表文件在创建的目录下。

```sql
sampledb=# CREATE TABLE newtbl (.....) TABLESPACE new_tblspc;

sampledb=# SELECT pg_relation_filepath('newtbl');
             pg_relation_filepath             
----------------------------------------------
 pg_tblspc/16386/PG_9.4_201409291/16384/18894
```

## 1.3. 堆表文件的内部结构

在数据文件（堆表、索引以及空闲空间映射表和可见性映射表）内部，它被分成固定长度的**page页**（或**block块**），默认为8192字节（8 KB）。 每个文件中的这些page页从0开始按顺序编号，这些编号称为**块编号**。 如果文件已满，PostgreSQL会在文件末尾添加一个新的空白page页以增加文件大小。

page页的内部结构取决于数据文件类型。 在本节中，将描述表table的结构，因为在下面的章节中将需要这些信息。 

**图 1.4. 堆表文件的page页内部结构**

![Fig. 1.4. Page layout of a heap table file.](imgs/ch1/fig-1-04.png)

表中的page页包含三种数据描述如下：

1. **堆元组(heap tuples)(s)** – 一个tuple就是一条数据记录。 它们从page页底部开始依次堆叠。 第5.2节和第9章描述了tuple的内部结构，因为需要同时了解PostgreSQL中的并发控制（CC）和WAL。

2. **行指针(line pointer)(s)** – 一个行指针长度为4个字节，并且包含指向每个堆元组的指针。 它也被称为**条目指针(item pointer)**。

   行指针形成一个简单的数组，它扮演着元组索引的角色。 每个索引都从1开始顺序编号，称为**偏移号(offset number)**。 当一个新的元组被添加到页面时，一个新的行指针被推到数组上以指向新的行。

3. **头数据(header data)** – 由 [PageHeaderData](javascript:void(0)) 结构定义的数据分配在page页的开头。 它是24个字节长，并包含有关页面的一般信息。 下面描述该结构的主要变量。

   - *pd_lsn* – 该变量存储由该页面的最后一次更改写入的XLOG记录的LSN。 它是一个8字节的无符号整数，与WAL机制有关。 细节在第9章中描述。
   - *pd_checksum* – 该变量存储此页面的校验和值。 （请注意，此版本在9.3版本或更高版本中受支持;在较早版本中，此部分存储了页面的时间轴ID。）
   - *pd_lower, pd_upper* – pd_lower指向行指针的末尾，pd_upper指向最新的堆元组的开头。
   - *pd_special* – 这个变量是用于索引的。 在表table的page页中，它指向页面的结尾。 （在索引内的page页中，它指向特殊空间的开始，它是仅由索引保存的数据区域，并且根据诸如B-tree，GiST，GiN等索引类型的种类包含特定数据）


行指针末尾和最新元组开头之间的空白空间称为**空闲空间(free space)**或**空洞(hole)**。

为了标识表中的元组，在内部使用**元组标识符(tuple identifier)（TID）**。 一个TID包含一对值：包含该元组的页面的块号，以及指向该元组的行指针的偏移号。 其典型的应用实例是索引。 更多细节见第1.4.2节。

PageHeaderData结构在 [src/include/storage/bufpage.h](https://github.com/postgres/postgres/blob/master/src/include/storage/bufpage.h).

另外，使用称为**TOAST**（The Oversized-Attribute Storage Technique）的方法存储和管理大小大于约2KB（约为8KB的1/4）的堆元组。 有关详细信息，请参阅 [PostgreSQL 文档](http://www.postgresql.org/docs/current/static/storage-toast.html)。

## 1.4. 写表和读表的方法

在本章的最后，描述了写入和读取堆元组tuple的方法。

### 1.4.1. 写元组(Heap Tuples)

假设一个表由一个只包含一个堆元组的page页组成。 该页面的pd_lower指向第一行指针，并且行指针和pd_upper都指向第一个堆元组。 参见图1.5(a)。

当第二个元组被插入时，它被放置在第一个元组之后。 第二行指针被推到第一行指针后，并指向第二个元组。 pd_lower更改为指向第二行指针，pd_upper更改为第二个堆元组。 参见图1.5(b)。 此页面中的其他头数据（例如，pd_lsn，pg_checksum，pg_flag）也被重写为适当的值; 更多细节在第5.3节和第9章中描述。

图 1.5. 写tuple

![Fig. 1.5. Writing of a heap tuple.](imgs/ch1/fig-1-05.png)

### 1.4.2. 读元组(Heap Tuples)

这里概述了两种典型的访问方法，顺序扫描和B树索引扫描：

- **顺序扫描(Sequential scan)** – 所有page页中的所有元组都通过扫描每个页中的所有行指针顺序读取。 参见图1.6(a)。
- **B树索引扫描(B-tree index scan)** – 索引文件包含索引元组，每个索引元组由索引键和指向目标堆元组的TID组成。 如果找到了正在查找的key值的索引元组，PostgreSQL使用获取的TID值读取所需的堆元组。 （在B树索引中查找索引元组的方法的描述在这里没有解释，因为它很常见，并且这里的空间是有限的，参见相关资料）。例如，在图1.6(b)中，TID 获得的索引元组的值是‘(block = 7, Offset = 2)’。 这意味着目标堆元组是表中第7页的第2个元组，因此PostgreSQL可以读取所需的堆元组，而不必在页面中进行不必要的扫描。

图. 1.6.  顺序扫描和索引扫描

![Fig. 1.6. Sequential scan and index scan.](imgs/ch1/fig-1-06.png)

 PostgreSQL也支持TID扫描(TID-Scan)，[位图扫描](https://wiki.postgresql.org/wiki/Bitmap_Indexes)(Bitmap-Scan))和仅索引扫描(Index-Only-Scan)。

TID扫描(TID-Scan)是一种通过使用所需元组的TID直接访问元组的方法。 例如，要查找表中第0页的第一个元组，请发出以下查询：

```sql
sampledb=# SELECT ctid, data FROM sampletbl WHERE ctid = '(0,1)';
 ctid  |   data    
-------+-----------
 (0,1) | AAAAAAAAA
(1 row)
```

Index-Only-Scan将在第7章中详细介绍。


