## 第六章 VACUUM

vacuum处理是一个维护过程，有助于PostgreSQL的持续运行。 它的两个主要任务是 *清理 dead tuples* 和 *冻结(freezing)事务ID*，这两个都在[Section 5.10 节](http://www.interdb.jp/pg/pgsql05.html#_5.10.)中简要提及。

为了清理 dead tuple，vacuum提供了两种模式，即 **Concurrent VACUUM** 和 **Full VACUUM**。 Concurrent VACUUM(通常简称为VACUUM)为表文件的每个页清理 dead tuple，其他事务可以在此过程运行时读取表。 相比之下，Full VACUUM 清理 dead tuple 并且整理文件的 live tuple 碎片，而其他事务无法在 Full VACUUM 运行时访问表。

尽管vacuum处理对于PostgreSQL来说至关重要，但与其他功能相比，改进其功能的速度一直很慢。 例如，在版本8.0之前，这个过程必须手动执行(使用psql工具或使用cron进程)。在2005年实现 **autovacuum** 守护进程的自动执行。

由于vacuum处理涉及扫描整个表，所以这是一个昂贵的过程。 在8.4版本(2009)中，引入了**可见性映射 Visibility Map (VM)**以提高清理dead tuple的效率。 在9.6版本(2016)中，通过增强VM来改进冻结处理。

6.1节概述了concurrent VACUUM。 接下来的部分将介绍以下内容。

- Visibility Map
- Freeze processing
- Removing unnecessary clog files
- Autovacuum daemon
- Full VACUUM

## 6.1. Concurrent VACUUM 概述

vacuum处理对指定的表或数据库中的所有表执行以下操作

1. 清理 dead tuple
   - 为每个page页清理dead tuple和整理live tuple碎片。
   - 删除指向dead tuple的索引元组。

2. 冻结 txid
   - 如果需要，冻结元组的txid。
   - 更新冻结的txid相关系统目录(pg_database和pg_class)。
   - 如果可能的话去除clog不必要的部分。

3. 其他
   - 更新已处理表的FSM和VM。
   - 更新统计信息(pg_stat_all_tables等)。


假设读者熟悉以下术语：[dead tuples](http://www.interdb.jp/pg/pgsql05.html#_5.3.), [freezing txid](http://www.interdb.jp/pg/pgsql05.html#_5.10.1.), [FSM](http://www.interdb.jp/pg/pgsql05.html#_5.3.4.) 和 [clog](http://www.interdb.jp/pg/pgsql05.html#_5.4.); 如果不了解，请参阅第5章。VM在6.2节介绍。

以下伪代码描述 vacuum。

 

 *伪代码: Concurrent VACUUM*

```sql
(1)  FOR each table
(2)       Acquire ShareUpdateExclusiveLock lock for the target table

          /* The first block */
(3)       Scan all pages to get all dead tuples, and freeze old tuples if necessary 
(4)       Remove the index tuples that point to the respective dead tuples if exists

          /* The second block */
(5)       FOR each page of the table
(6)            Remove the dead tuples, and Reallocate the live tuples in the page
(7)            Update FSM and VM
           END FOR

          /* The third block */
(8)       Truncate the last page if possible
(9)       Update both the statistics and system catalogs of the target table
           Release ShareUpdateExclusiveLock lock
       END FOR

        /* Post-processing */
(10)  Update statistics and system catalogs
(11)  Remove both unnecessary files and pages of the clog if possible
```

(1)从指定的表中获取每个表。

(2)获取表的ShareUpdateExclusiveLock锁。 该锁允许从其他事务中读取。

(3)扫描所有页以获取所有dead tuple，并在必要时冻结dead tuple。

(4)删除指向相应dead tuple的索引元组(如果存在的话)。

(5)为表的每个页执行以下处理，步骤(6)和(7)。

(6)删除dead tuple并重新分配页中的live tuple。

(7)更新目标表的相应FSM和VM。

(8)如果最后一页没有任何元组，则截断最后一页。

(9)更新与vacuum处理的表相关的统计数据和系统目录。

(10)更新与vacuum处理相关的统计数据和系统目录。

(11)如果可能，删除不必要的文件和clog。



这个伪代码有两个部分：每个表的循环和后处理。 内循环可以分为三块。

下面概述这三块和后处理。

### 6.1.1. 第一块

该块执行冻结处理并删除指向dead tuple的索引元组。

首先，PostgreSQL扫描一个目标表来建立一个dead tuple列表，并尽可能冻结旧的元组。 该列表存储在本地缓存的 [maintenance_work_mem](https://www.postgresql.org/docs/current/static/runtime-config-resource.html#GUC-MAINTENANCE-WORK-MEM) 中。 6.3节描述了冻结处理。

扫描后，PostgreSQL通过引用dead tuple列表来删除索引元组。

当maintenance_work_mem满了并且扫描不完整时，PostgreSQL进行下一个任务，即步骤(4)到(7); 然后返回步骤(3)并继续进行剩余扫描。

### 6.1.2. 第二块

该块清理dead tuple，并逐页更新FSM和VM。 图6.1给出了一个例子：

**图. 6.1. 清理一个dead tuple**

![Fig. 6.1. Removing a dead tuple.](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch6/fig-6-01.png?raw=true)

假设该表包含三个page页。 我们专注于第0页(即第一页)。 这个页有三个元组。 Tuple_2是一个dead tuple(图6.1(1))。 在这种情况下，PostgreSQL清理Tuple_2并重新排序剩余的元组以整理碎片，然后更新此页面的FSM和VM(图6.1(2))。 PostgreSQL继续这个过程直到最后一页。

请注意，不必要的行指针不会被删除，它们将在未来重用。 因为如果删除行指针，则必须更新关联索引的所有索引元组。

### 6.1.3. 第三块

第三块更新与vacuum处理的表相关的统计数据和系统目录。

而且，如果最后一页没有元组，它将从表文件中截断。

### 6.1.4. 后处理

当vacuum处理完成后，PostgreSQL会更新与vacuum处理有关的多个统计数据和系统目录，并且如果可能的话，它会删除clog不必要的部分(见第6.4节)。 

vacuum处理使用8.5节描述的 *ring buffer*; 因此，处理的页面不会缓存在共享缓冲区中。

 

## 6.2. Visibility Map

vacuum处理成本高昂; 因此，在8.4版本中引入了VM以降低成本。

VM的基本概念很简单。 每个表都有一个单独的可见性映射表，用于保存表中每个页的可见性。 页的可见性决定了每个页是否有dead tuple。 vacuum处理可以跳过一个没有dead tuple的页。

图6.2显示了如何使用VM。 假设表由三页组成，第0页和第2页包含dead tuple，第1页不包含。 该表的VM拥有关于哪些页包含dead tuple的信息。 在这种情况下，vacuum处理通过参考VM的信息跳过第一页。

每个VM由一个或多个8 KB页组成，并且该文件使用“vm”后缀存储。 例如，一个表文件的relfilenode为18751，其FSM(18751_fsm)和VM(18751_vm)文件显示在它下面。

**图 6.2. VMS使用示例**

![Fig. 6.2. How the VM is used.](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch6/fig-6-02.png?raw=true)

```shell
$ cd $PGDATA
$ ls -la base/16384/18751*
-rw------- 1 postgres postgres  8192 Apr 21 10:21 base/16384/18751
-rw------- 1 postgres postgres 24576 Apr 21 10:18 base/16384/18751_fsm
-rw------- 1 postgres postgres  8192 Apr 21 10:18 base/16384/18751_vm
```

### 6.2.1. VM 的增强

VM在版本9.6中进行了增强，以提高冻结处理的效率。 新的VM显示page页可见性以及关于每个页中的元组是否冻结的信息(见第6.3.3节)。

## 6.3. 冻结(Freeze)处理

冻结处理有两种模式，根据特定条件在任一模式下执行。 为了方便，这些模式被称为 **lazy 模式** 和 **eager 模式**。

Concurrent VACUUM在内部通常被称为“lazy vacuum”。 但是，本文定义的 lazy 模式是冻结处理如何执行的模式。

冻结处理通常以 lazy 模式运行; 然而，当特定条件得到满足时，eager模式运行。

在 lazy 模式下，冻结处理仅使用目标表的相应VM扫描包含dead tuple的页面。

相反，无论page页是否包含dead tuple，eager 模式都会对所有页进行扫描，并且还会更新与冻结处理相关的系统目录，并在可能的情况下删除clog不必要的部分。

6.3.1节和6.3.2节分别描述了这些模式。 第6.3.3节描述了在 eager 模式下改进冻结处理。

### 6.3.1. Lazy 模式

启动冻结处理时，PostgreSQL计算FreezeLimittxid并冻结t_xmin小于FreezeLimittxid的元组。 

freezeLimit txid定义如下：

​	freezeLimit_txid =(OldestXmin-vacuum_freeze_min_age)

OldestXminOldestXmin是当前正在运行的事务中最老的txid。例如，如果执行VACUUM命令时有三个事务(txids 100,101和102)正在运行，则OldestXminOldestXmin为100.如果不存在其他事务，则OldestXminOldestXmin是执行此VACUUM命令的txid。这里，[vacuum_freeze_min_age](https://www.postgresql.org/docs/current/static/runtime-config-client.html#GUC-VACUUM-FREEZE-MIN-AGE) 是一个配置参数(默认50,000,000)。

图6.3给出了一个具体的例子。在这里，Table_1由三个页组成，每个页有三个元组。当执行VACUUM命令时，当前的txid是5,002,500，并且没有其他事务。在这种情况下，OldestXminOldestXmin是5,002,500;因此，freezeLimit txid是2500.冻结处理执行如下。

**图. 6.3. 以 lazy 模式冻结元组**

![Fig. 6.3. Freezing tuples in lazy mode.](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch6/fig-6-03.png?raw=true)

$0^{th}$ page:

由于所有t_xmin值都小于freezeLimit txid，因此三个元组被冻结。 另外，在这个vacuum过程中，由于Tuple_1是dead tuple，所以被清理。

$1^{st}$ page:

通过引用VM来跳过此页。

$2^{nd}$ page:

Tuple_7和Tuple_8被冻结; Tuple_7被清理。

在完成vacuum处理之前，与vacuum有关的统计数据被更新，例如，[pg_stat_all_tables](https://www.postgresql.org/docs/current/static/monitoring-stats.html#PG-STAT-ALL-TABLES-VIEW)'n_live_tup，n_dead_tup，last_vacuum，vacuum_count等

如上例所示，lazy 模式可能无法完全冻结元组，因为它可以跳过页。

### 6.3.2. Eager 模式

eager模式弥补了lazy模式的缺陷。它扫描所有页以检查表中的所有元组，更新相关的系统目录，并在可能的情况下删除不必要的文件和clog页。

当满足以下条件时执行eager模式。

​	pg_database.datfrozenxid <(OldestXmin-vacuum_freeze_table_age)

在上面的条件中，pg_database.datfrozenxid表示[pg_database](https://www.postgresql.org/docs/current/static/catalog-pg-database.html)系统目录的列，并保存每个数据库的最早的冻结txid。细节在后面描述;因此，我们假设所有pg_database.datfrozenxid的值都是1821(这是刚刚在版本9.5中安装新数据库集群后的初始值).[Vacuum_freeze_table_age](https://www.postgresql.org/docs/current/static/runtime-config-client.html#GUC-VACUUM-FREEZE-TABLE-AGE)是一个配置参数(默认值为150,000,000)。

图6.4给出了一个具体的例子。在Table_1中，Tuple_1和Tuple_7都已被清理。 Tuple_10和Tuple_11已经被插入第二页。当执行VACUUM命令时，当前txid是150,002,000，并且没有其他事务。因此，OldestXmin是150,002,000，freezeLimit txid是100,002,000。在这种情况下，由于'1821 <(150002000-150000000)1821 <(150002000-150000000)'，因此满足上述条件。因此，在eager模式中冻结处理如下执行。

(请注意，这是9.5或更早版本的行为;最新行为在6.3.3节中描述)。

**图. 6.4. 以eager模式(版本9.5或更低版本)冻结旧元组**

![Fig. 6.4. Freezing old tuples in eager mode (version 9.5 or earlier).](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch6/fig-6-04.png?raw=true)

$0^{th}$ page:

即使所有元组都已被冻结，Tuple_2和Tuple_3也被检查。

$1^{st}$ page:

此页中的三个元组已冻结，因为所有t_xmin值都小于freezeLimit txid。 请注意，在lazy模式此页会跳过。

$2^{nd}$ page:

Tuple_10已被冻结。 Tuple_11没有。

冻结每个表后，目标表的pg_class.relfrozenxid被更新。 [pg_class](https://www.postgresql.org/docs/current/static/catalog-pg-class.html) 是一个系统目录，每个pg_class.relfrozenxid列保存相应表的最新冻结xid。在这个例子中，Table_1的pg_class.relfrozenxid更新为当前的freezeLimit txid(即100,002,000)，这意味着Table_1中t_xmin小于100,002,000的所有元组都被冻结。

在完成vacuum处理之前，必要时更新pg_database.datfrozenxid。每个pg_database.datfrozenxid列在相应的数据库中保存最小pg_class.relfrozenxid。例如，如果只有Table_1在eager模式下被冻结，则此数据库的pg_database.datfrozenxid不会更新，因为其他关系(可从当前数据库中看到的其他表和系统目录)的pg_class.relfrozenxid尚未更改(图6.5(1))。如果当前数据库中的所有关系都以预先模式冻结，则会更新数据库的pg_database.datfrozenxid，因为此数据库的所有关系的pg_class.relfrozenxid已更新为当前的freezeLimit txid(图6.5(2))。

**图. 6.5. pg_database.datfrozenxid和pg_class.relfrozenxid(s)之间的关系**

![Fig. 6.5. Relationship between pg_database.datfrozenxid and pg_class.relfrozenxid(s).](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch6/fig-6-05.png?raw=true)

 

*查看pg_class.relfrozenxid 和 pg_database.datfrozenxid*

在下文中，第一个查询显示'testdb'数据库中所有可见关系的relfrozenxids，第二个查询显示'testdb'数据库的pg_database.datfrozenxld。

```sql
testdb=# VACUUM table_1;
VACUUM

testdb=# SELECT n.nspname as "Schema", c.relname as "Name", c.relfrozenxid
             FROM pg_catalog.pg_class c
             LEFT JOIN pg_catalog.pg_namespace n ON n.oid = c.relnamespace
             WHERE c.relkind IN ('r','')
                   AND n.nspname <> 'information_schema' AND n.nspname !~ '^pg_toast'
                   AND pg_catalog.pg_table_is_visible(c.oid)
                   ORDER BY c.relfrozenxid::text::bigint DESC;
   Schema   |            Name         | relfrozenxid 
------------+-------------------------+--------------
 public     | table_1                 |    100002000
 public     | table_2                 |         1846
 pg_catalog | pg_database             |         1827
 pg_catalog | pg_user_mapping         |         1821
 pg_catalog | pg_largeobject          |         1821

...

 pg_catalog | pg_transform            |         1821
(57 rows)

testdb=# SELECT datname, datfrozenxid FROM pg_database WHERE datname = 'testdb';
 datname | datfrozenxid 
---------+--------------
 testdb  |         1821
(1 row)
```

 

 *FREEZE option*

具有FREEZE选项的VACUUM命令强制冻结指定表中的所有txid。 这在eager模式下执行; 但是，freezeLimit设置为OldestXmin(不是'OldestXmin - vacuum_freeze_min_age')。 例如，当VACUUM FULL命令由txid 5000执行且没有其他正在运行的事务时，OldesXmin被设置为5000，而小于5000的txids被冻结。

 

### 6.3.3. 在Eager模式下改进冻结处理

9.5版本或更早版本中的eager模式效率不高，因为它总是扫描所有页。 例如，在6.3.2节的例子中，即使页中的所有元组都被冻结，也会扫描第0页。

为了解决这个问题，VM和冻结处理在9.6版本中得到了改进。 如第6.2.1节所述，新VM具有关于每个页中是否冻结所有元组的信息。 当在eager模式下执行冻结处理时，可以跳过仅包含冻结元组的页面。

图6.6给出了一个例子。 当冻结该表时，通过参考VM的信息跳过第0页。 冻结第1页后，由于此页面的所有元组都已冻结，所以更新了相关的VM信息。

**图. 6.6. 以eager模式冻结旧元组(版本9.6或更高版本)**

![Fig. 6.6. Freezing old tuples in eager mode (version 9.6 or later).](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch6/fig-6-06.png?raw=true)

## 6.4. 清理不需要的Clog文件

5.4节描述的clog存储事务状态。 当pg_database.datfrozenxid更新时，PostgreSQL将尝试清理不必要的clog文件。 请注意，相应的clog页也会被删除。

图6.7给出了一个例子。 如果最小pg_database.datfrozenxid包含在clog文件'0002'中，则可以清理较旧的文件('0000'和'0001')，因为存储在这些文件中的所有事务可以被视为整个数据库集群中的冻结txid。

**图. 6.7. 清理不必要的clog文件和页**

![Fig. 6.7. Removing unnecessary clog files and pages.](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch6/fig-6-07.png?raw=true)

 

 *pg_database.datfrozenxid and the clog file*

下面显示了pg_database.datfrozenxid和clog文件的实际输出：

```sql
$ psql testdb -c "SELECT datname, datfrozenxid FROM pg_database"
  datname  | datfrozenxid 
-----------+--------------
 template1 |      7308883
 template0 |      7556347
 postgres  |      7339732
 testdb    |      7506298
(4 rows)

$ ls -la -h data/pg_clog/	# In version 10 or later, "ls -la -h data/pg_xact/"
total 316K
drwx------  2 postgres postgres   28 Dec 29 17:15 .
drwx------ 20 postgres postgres 4.0K Dec 29 17:13 ..
-rw-------  1 postgres postgres 256K Dec 29 17:15 0006
-rw-------  1 postgres postgres  56K Dec 29 17:15 0007
```

 

## 6.5. Autovacuum 守护进程

autovacuum守护进程已经使vacuum处理自动化; 因此，PostgreSQL的操作变得非常简单。

autovacuum守护进程定期调用几个autovacuum_worker进程。 默认情况下，它每1分钟唤醒一次(由[autovacuum_naptime](https://www.postgresql.org/docs/current/static/runtime-config-autovacuum.html#GUC-AUTOVACUUM-NAPTIME)定义)，并调用三个worker(由[autovacuum_max_works](https://www.postgresql.org/docs/current/static/runtime-config-autovacuum.html#GUC-AUTOVACUUM-MAX-WORKERS)定义)。

由autovacuum调用的autovacuum worker对各个表同时执行vacuum处理，而对数据库活动影响最小。

## 6.6. Full VACUUM

虽然Concurrent VACUUM至关重要，但这还不够。 例如，即使删除了许多dead tuple，它也不能减小表的大小。

图6.8给出了一个极端的例子。 假设一个表由三个页组成，每个页包含六个元组。 执行以下DELETE命令以删除元组，并执行VACUUM命令以清理dead tuple：

**图. 6.8. VACUUM(concurrent)缺点示例**

![Fig. 6.8. An example showing the disadvantages of (concurrent) VACUUM.](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch6/fig-6-08.png?raw=true)

```sql
testdb=# DELETE FROM tbl WHERE id % 6 != 0;
testdb=# VACUUM tbl;
```

dead tuple被清理; 但是，表大小并未减少。 这既浪费磁盘空间，也会对数据库性能产生负面影响。 例如，在上面的示例中，当读取表中的三个元组时，必须从磁盘加载三个页。

为了处理这种情况，PostgreSQL提供了Full VACUUM模式。 图6.9显示了这种模式的概要。

**图. 6.9. Full VACUUM 模式概述**

![Fig. 6.9. Outline of Full VACUUM mode.](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch6/fig-6-09.png?raw=true)

[1]创建新表：图6.9(1)

当对表执行VACUUM FULL命令时，PostgreSQL首先获取表的AccessExclusiveLock锁并创建一个大小为8 KB的新表文件。 AccessExclusiveLock锁不允许访问。

[2]将live tuple复制到新表中：图6.9(2)

PostgreSQL只将旧表文件中的live tuple复制到新表中。

[3]删除旧文件，重建索引，并更新统计信息，FSM和VM：图6.9(3)

复制所有live tupel后，PostgreSQL删除旧文件，重建所有关联表索引，更新此表的FSM和VM，并更新关联的统计信息和系统目录。

Full VACUUM的伪代码如下所示：

 

*伪代码：Full VACUUM*

```sql
(1)  FOR each table
(2)       Acquire AccessExclusiveLock lock for the table
(3)       Create a new table file
(4)       FOR each live tuple in the old table
(5)            Copy the live tuple to the new table file
(6)            Freeze the tuple IF necessary
            END FOR
(7)        Remove the old table file
(8)        Rebuild all indexes
(9)        Update FSM and VM
(10)      Update statistics
            Release AccessExclusiveLock lock
       END FOR
(11)  Remove unnecessary clog files and pages if possible
```

使用VACUUM FULL命令时应考虑两点。

1. 当Full VACUUM正在处理时，不能访问(读取/写入)表。

2. 临时使用表的磁盘空间至多为磁盘空间的两倍; 因此，在处理巨大的表时，需要检查剩余的磁盘容量。

 

*什么时候做VACUUM FULL？*

不幸的是，当执行'VACUUM FULL'时没有最佳时机。 但是，扩展[pg_freespacemap](https://www.postgresql.org/docs/current/static/pgfreespacemap.html)可能会给你很好的建议。

以下查询显示了表的平均空闲空间比例。

```sql
testdb=# CREATE EXTENSION pg_freespacemap;
CREATE EXTENSION

testdb=# SELECT count(*) as "number of pages",
       pg_size_pretty(cast(avg(avail) as bigint)) as "Av. freespace size",
       round(100 * avg(avail)/8192 ,2) as "Av. freespace ratio"
       FROM pg_freespace('accounts');
 number of pages | Av. freespace size | Av. freespace ratio 
-----------------+--------------------+---------------------
            1640 | 99 bytes           |                1.21
(1 row)
```

如上所述，您可以发现空闲空间很少。

如果删除大部分元组并执行VACUUM命令，则可以发现页几乎是空的。

```sql
testdb=# DELETE FROM accounts WHERE aid %10 != 0 OR aid < 100;
DELETE 90009

testdb=# VACUUM accounts;
VACUUM

testdb=# SELECT count(*) as "number of pages",
       pg_size_pretty(cast(avg(avail) as bigint)) as "Av. freespace size",
       round(100 * avg(avail)/8192 ,2) as "Av. freespace ratio"
       FROM pg_freespace('accounts');
 number of pages | Av. freespace size | Av. freespace ratio 
-----------------+--------------------+---------------------
            1640 | 7124 bytes         |               86.97
(1 row)
```

以下查询将检查指定表的每个页的空闲空间比例。

```sql
testdb=# SELECT *, round(100 * avail/8192 ,2) as "freespace ratio"
                FROM pg_freespace('accounts');
 blkno | avail | freespace ratio 
-------+-------+-----------------
     0 |  7904 |           96.00
     1 |  7520 |           91.00
     2 |  7136 |           87.00
     3 |  7136 |           87.00
     4 |  7136 |           87.00
     5 |  7136 |           87.00
....
```

在执行VACUUM FULL之后，您会发现表文件已被压缩。

```sql
testdb=# VACUUM FULL accounts;
VACUUM
testdb=# SELECT count(*) as "number of blocks",
       pg_size_pretty(cast(avg(avail) as bigint)) as "Av. freespace size",
       round(100 * avg(avail)/8192 ,2) as "Av. freespace ratio"
       FROM pg_freespace('accounts');
 number of pages | Av. freespace size | Av. freespace ratio 
-----------------+--------------------+---------------------
             164 | 0 bytes            |                0.00
(1 row)
```