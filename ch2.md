## 第二章 进程和内存结构

在本章中，总结了PostgreSQL中的进程体系结构和内存体系结构，以帮助阅读后续章节。 如果你已经熟悉它们，你可以跳过本章。

## 2.1. 进程体系结构 

PostgreSQL是一个具有多进程体系结构的C/S类型的关系数据库管理系统，并可在单个主机上运行。

协同管理一个数据库集群的多个进程的集合通常被称为*'PostgreSQL server'*，它包含以下类型的进程：

- **postgres服务进程(server process)**是与数据库集群管理相关的所有进程的父进程。
- 每个**后端进程(backend process)**处理由连接的客户端发出的所有查询和语句。
- 各种**后台进程(background processes)**执行用于数据库管理的每个特征的进程(例如，VACUUM和CHECKPOINT进程)。
- 在**复制相关进程(replication associated processes)**中，它们执行流复制。详情见[第11章](ch11.md)。 
- 在9.3版本支持的**后台工作进程(background worker process)**中，它可以执行用户执行的任何处理。 这里不详细介绍，请参考[官方文档](http://www.postgresql.org/docs/current/static/bgworker.html)。

在下面的小节中，将详细介绍前三种类型的进程。

**图. 2.1. PostgreSQL进程体系结构**

![Fig. 2.1. An example of the process architecture in PostgreSQL.](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch2/fig-2-01.png?raw=true)

------

### 2.1.1. 服务进程(server process) 

如上所述，postgres服务器进程是PostgreSQL服务中所有进程的父进程。 在早期版本中，它被称为‘postmaster’。

通过执行[pg_ctl](http://www.postgresql.org/docs/current/static/app-pg-ctl.html) start，启动postgres服务器进程。 然后，它在内存中分配一个共享内存区域，启动各种后台进程，根据需要启动复制关联进程和后台工作进程，并等待来自客户端的连接请求。 每当接收到来自客户端的连接请求时，都会启动后端进程。(然后，启动的后端进程处理客户端连接发出的所有查询)

一个postgres服务器进程监听一个网络端口，默认端口是5432.虽然可以在同一台主机上运行多个PostgreSQL服务，但应该设置每台服务监听彼此不同的端口号，例如5432,5433 等等

### 2.1.2. 后端进程(backend process)

后端进程(也称为*postgres*)由postgres服务进程启动，并处理由一个连接的客户端发出的所有查询。 它通过单个TCP连接与客户端进行通信，并在客户端断开连接时终止。

由于它只允许操作一个数据库，因此在连接到PostgreSQL服务器时必须明确指定使用的数据库。

PostgreSQL允许多个客户端同时连接; 配置参数max_connections控制客户端连接的最大数量(默认值为100)。

如果许多客户端(如WEB应用程序)频繁地重复与PostgreSQL服务器的连接和断开连接，则会增加建立连接和创建后端进程的成本，因为PostgreSQL尚未实现本地连接池功能。 这种情况对数据库服务器的性能有负面影响。 为了处理这种情况，通常使用连接池中间件([pgbouncer](https://pgbouncer.github.io/)或[pgpool-II](http://www.pgpool.net/mediawiki/index.php/Main_Page))。

### 2.1.3. 后台进程(background process)

表2.1描述了后台进程列表。 与postgres服务进程和后端进程相比，不可能简单地描述每个进程，因为这些进程取决于各自特定功能和PostgreSQL内部。 因此，在本章中，只做介绍。 细节将在下面的章节中描述。

| 进程                       | 描述                                                         | 参考                                                         |
| -------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| background writer          | 此进程用于将共享缓冲池上的脏页面逐渐被定期写入永久存储器(例如，HDD，SSD)。 (在9.1或更早版本中，它也负责检查点进程) | [8.6 节](http://www.interdb.jp/pg/pgsql08.html#_8.6.)        |
| checkpointer               | 在9.2或更高版本中，此进程用于执行检查点处理                  | [8.6 节](http://www.interdb.jp/pg/pgsql08.html#_8.6.), [9.7 节](http://www.interdb.jp/pg/pgsql09.html#_9.7.) |
| autovacuum launcher        | autovacuum-launcher进程定期调用autovacuum-worker进程进程清理工作 | [6.5 节](http://www.interdb.jp/pg/pgsql06.html#_6.5.)        |
| WAL writer                 | 此进程定期将Wal缓冲区上的Wal数据写入并刷新到持久存储中       | [9.9 节](http://www.interdb.jp/pg/pgsql09.html#_9.9.)        |
| statistics collector       | 此进程用于收集统计信息，比如pg_stat_activeforpg_stat_database等。 |                                                              |
| logging collector (logger) | 此进程将错误消息写入日志文件。                               |                                                              |
| archiver                   | 此进程将执行归档日志记录。                                   | [9.10 节](http://www.interdb.jp/pg/pgsql09.html#_9.10.)      |

这里显示了PostgreSQL服务的实际进程。 在以下示例中，一个postgres服务器进程(pid为9687)，两个后端进程(pid为9697和9717)，并且表2.1中列出的几个后台进程正在运行。 也参见图2.1。

```shell
postgres> pstree -p 9687
-+= 00001 root /sbin/launchd
 \-+- 09687 postgres /usr/local/pgsql/bin/postgres -D /usr/local/pgsql/data
   |--= 09688 postgres postgres: logger process     
   |--= 09690 postgres postgres: checkpointer process     
   |--= 09691 postgres postgres: writer process     
   |--= 09692 postgres postgres: wal writer process     
   |--= 09693 postgres postgres: autovacuum launcher process     
   |--= 09694 postgres postgres: archiver process     
   |--= 09695 postgres postgres: stats collector process     
   |--= 09697 postgres postgres: postgres sampledb 192.168.1.100(54924) idle  
   \--= 09717 postgres postgres: postgres sampledb 192.168.1.100(54964) idle in transaction  
```

## 2.2. 内存体系结构 

PostgreSQL中的内存架构可以分为两大类：

- 本地缓存(Local memory area) – 由每个后端进程分配以供其自己使用
- 共享缓存(Shared memory area) – PostgreSQL服务的所有进程使用

在下面的小节中，将简要描述这些小节。

**图. 2.2. PostgreSQL内存架构**

![Fig. 2.2. Memory architecture in PostgreSQL.](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch2/fig-2-02.png?raw=true)

### 2.2.1. 本地缓存(local memory)

每个后端进程为查询处理分配一个本地缓存区域; 每个区域被分成几个子区域 - 其大小可以是固定的或可变的。 表2.2列出了主要子区域。 细节将在下面的章节中描述。

| 子区域               | 描述                                                         | 参考                                                  |
| -------------------- | ------------------------------------------------------------ | ----------------------------------------------------- |
| work_mem             | 执行器(Executor)使用此区域通过ORDER BY和DISTINCT操作对元组进行排序，并通过merge-join和hash-join操作来连接表。 | [第三章](http://www.interdb.jp/pg/pgsql03.html)       |
| maintenance_work_mem | 某些类型的维护操作(例如，VACUUM，REINDEX)使用此区域。        | [6.1 节](http://www.interdb.jp/pg/pgsql06.html#_6.1.) |
| temp_buffers         | 执行器(Executor)使用这个区域来存储临时表。                   |                                                       |

### 2.2.2. 共享缓存(shared memory)

PostgreSQL服务器启动时共享内存区域。 这个区域也被分成几个固定大小的子区域。 表2.3列出了主要的子区域。 细节将在下面的章节中描述。

| 子区域             | 描述                                                         | 参考                                                       |
| ------------------ | ------------------------------------------------------------ | ---------------------------------------------------------- |
| shared buffer pool | PostgreSQL将表和索引中的页面从持久存储装载到这里，并直接操作它们 | [Chapter 8](http://www.interdb.jp/pg/pgsql08.html)         |
| WAL buffer         | 为了确保服务器故障没有数据丢失，PostgreSQL支持WAL机制。 WAL数据（也称为XLOG记录）是PostgreSQL中的事务日志; 而WAL缓冲区是在写入永久性存储之前WAL数据的缓冲区 | [Chapter 9](http://www.interdb.jp/pg/pgsql09.html)         |
| commit log         | 提交日志（CLOG）为并发控制（CC）机制保留所有事务的状态（例如in_progress，committed，aborted） | [Section 5.4](http://www.interdb.jp/pg/pgsql05.html#_5.4.) |

除此之外，PostgreSQL还分配了几个区域，如下所示：

- 各种访问控制机制的子区域。 (例如信号量(semaphores)，轻量级锁(lightweight locks)，共享和排他锁(shared and exclusive locks)等)
- 各种后台进程的子区域，如checkpointer和autovacuum。
- 用于事务处理的子区域，例如保存点(save-point)和两阶段提交(two-phase-commit)。

其他等等。
