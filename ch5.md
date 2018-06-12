第五章 并发控制

当数据库中同时运行多个事务时，并发控制是维护一致性和隔离性的一种机制。

有三种主要的并发控制技术，即 *Multi-version Concurrency Control* (MVCC), *Strict Two-Phase Locking* (S2PL) 和 *Optimistic Concurrency Control* (OCC)，每种技术都有很多变化。 在MVCC中，每个写操作都会创建新版本的数据项，同时保留旧版本。 当事务读取数据项时，系统会选择其中一个版本来确保单个事务的隔离。 MVCC的主要优势在于”读不会阻塞写，而写也从不阻塞读“，相反，例如，基于S2PL的系统当有写操作时必须阻塞读，因为写操作获得了独占锁。 PostgreSQL和一些RDBMS使用称为 **快照隔离 Snapshot Isolation (SI)** 的MVCC变体。

为了实现SI，一些RDBMS(例如Oracle)使用回滚段(rollback segments)。 当写入新的数据项时，旧版本的项目被写入回滚段，随后新项被覆盖到数据区域。 PostgreSQL使用更简单的方法。 一个新的数据项被直接插入相关表页。 读取数据时，PostgreSQL通过**可见性检查规则(visibility check rules)**来选择项的适当版本以响应单个事务。

SI不支持ANSI SQL-92标准中定义的三种异常，即 *脏读 Dirty Reads*, *不可重复读 Non-Repeatable Reads* 和 *幻读 Phantom Reads*。 但是，SI不能实现真正的可串行化，因为它允许序列化异常，例如 *Write Skew* 和 *Read-only Transaction Skew*。 请注意，基于传统可串行化定义的ANSI SQL-92标准并不等同于现代理论中的定义。 为了解决这个问题，从9.1版本开始添加了**可串行化的快照隔离 Serializable Snapshot Isolation (SSI)**。 SSI可以检测序列化异常，并且可以解决由这种异常引起的冲突。 因此，PostgreSQL 9.1及更高版本提供了真正的SERIALIZABLE隔离级别。 (另外，SQL Server也使用SSI，Oracle仍然只使用SI)

本章包括以下四个部分：

- **第一部分:** 5.1. — 5.3节

  本部分提供理解后续部分所需的基本信息。

  5.1节和5.2节分别描述事务ID和元组结构。 5.3节描述如何插入，删除和更新元组。

- **第二部分:** 5.4. — 5.6节

  这部分描述实现并发控制机制所需的关键功能。

  5.4,5.5和5.6节描述了事务提交日志(clog)，它分别保存所有事务状态，事务快照和可见性检查规则。

- **第三部分:** 5.7. — 5.9节

  这部分使用具体示例来描述PostgreSQL中的并发控制。

  5.7节介绍了可见性检查。 本节还将介绍如何防止ANSI SQL标准中定义的三种异常。5.8节描述了防止 *丢失更新 Lost Updates*，5.9节简要描述了SSI。

- **第四部分:** 5.10节

  本部分介绍运行并发控制机制所需的几个维护过程。 维护过程由[第六章](ch6.md)介绍的vcacuum机制完成。

虽然有许多与并发控制相关的内容，本章重点讨论PostgreSQL独有的部分。另外，死锁和锁模式这里不进行描述(更多信息请参考[官方文档](https://www.postgresql.org/docs/current/static/explicit-locking.html))。

  

*PostgreSQL中的事务隔离界别  Transaction Isolation Level in PostgreSQL*

下表描述了PostgreSQL实现的事务隔离级别：

| 级别               | 脏读   | 不可重复读 | 幻读                                         | 序列化异常 |
| ------------------ | ------ | ---------- | -------------------------------------------- | ---------- |
| READ COMMITTED     | 不可能 | 可能       | 可能                                         | 可能       |
| REPEATABLE READ *1 | 不可能 | 不可能     | PG中不可能; 参见5.7.2节。 (在ANSI SQL中可能) | 可能       |
| SERIALIZABLE       | 不可能 | 不可能     | 不可能                                       | 不可能     |


*1：在9.0及更早版本中，此级别被用作'SERIALIZABLE'，因为它不允许ANSI SQL-92标准中定义的三种异常。 但是，在9.1版本实现SSI，此级别已更改为“REPEATABLE READ”，并引入了真正的SERIALIZABLE级别。

 

PostgreSQL对于DML(Data Manipulation Language，例如SELECT，UPDATE，INSERT，DELETE)使用SSI，对DDL(Data Definition Language，例如CREATE TABLE等)使用2PL。

## 5.1. 事务ID

每当事务开始时，由事务管理器分配一个唯一标识符 **事务id(txid)**。 PostgreSQL的txid是一个32位无符号整数，约为42亿(千万)。 如果在事务开始后执行内置函数 txid_current()，则函数按如下所示返回当前的txid。

```sql
testdb=# BEGIN;
BEGIN
testdb=# SELECT txid_current();
 txid_current 
--------------
          100
(1 row)
```

PostgreSQL保留以下三个特殊的 txid：

- **0** 表示 **Invalid** txid.
- **1** 表示 **Bootstrap** txid, 它仅用于数据库集群的初始化。
- **2** 表示 **Frozen** txid, 这在第5.10.1节中有描述。

txid可以相互比较。 例如，从txid 100的角度看，大于100的txid表示“将来的”，并且它们在txid 100中不可见; 小于100的txid表示"过去的"并且可见(图5.1a))。

**图 5.1. PostgreSQL中事务ID示例**

![Fig. 5.1. Transaction ids in PostgreSQL.](http://www.interdb.jp/pg/img/fig-5-01.png)![img]()

由于txid空间在实际系统中是不够的，PostgreSQL将txid空间视为一个圆。之前的21亿txid是“过去的”，之后的21亿txid是“将来的”(图5.1 b)。 

注意，在5.10.1节中描述所谓的 *txid wraparound 问题*。 

注意，没有为BEGIN命令分配一个txid。在PostgreSQL中，当执行BEGIN命令后执行第一个命令时，事务管理器将分配一个tixd，然后开始事务处理。

 

## 5.2. tuple结构

表页中的堆元组被分类为普通元组和TOAST元组。 本节仅介绍普通元组。

堆元组包括三部分，即HeapTupleHeaderData structure, NULL bitmap, and user data(图5.2)。

**图. 5.2. Tuple 结构**

![Fig. 5.2. Tuple structure.](http://www.interdb.jp/pg/img/fig-5-02.png)![img]()

HeapTupleHeaderData 结构在 [src/include/access/htup_details.h](https://github.com/postgres/postgres/blob/ee943004466418595363d567f18c053bae407792/src/include/access/htup_details.h) 中定义。

虽然 [HeapTupleHeaderData](javascript:void(0)) 结构包含七个元素，但后续章节中只涉及其中四个元素。

- **t_xmin** 记录插入此元组的事务的txid。
- **t_xmax** 记录删除或更新此元组的事务txid。 如果这个元组没有被删除或更新，t_xmax被设置为0，这意味着INVALID。
- **t_cid** 记录命令ID(command id，cid)，这意味着在从0开始的当前事务中执行此命令之前执行了多少个SQL命令。例如，假定我们在单个事务中执行三个INSERT命令：BEGIN; INSERT; INSERT; INSERT; COMMIT;'。 如果第一个命令插入这个元组，则t_cid被设置为0.如果第二个命令插入该元组，则t_cid被设置为1，依此类推。
- **t_ctid** 记录指向自身或新元组的元组标识符(tuple identifier，tid)。 在1.3节中描述的tid用于标识表中的元组。 当这个元组更新时，这个元组的t_ctid指向新的元组; 否则，t_ctid指向自己。

## 5.3. 插入、删除、更新tuple

本节介绍如何插入，删除和更新元组。 然后，简要描述用于插入和更新元组的 *空闲空间映射表 Free Space Map (FSM)*。

这里主要描述tuple，不再描述 page header、line pointer。图5.3给出tuple示例。

**图. 5.3. tuple示例**

![Fig. 5.3. Representation of tuples.](http://www.interdb.jp/pg/img/fig-5-03.png)![img]()

### 5.3.1. 插入

通过插入操作，可以将新元组直接插入到目标表的page页中(图5.4)。

**图 5.4. 插入tuple**

![Fig. 5.4. Tuple insertion.](http://www.interdb.jp/pg/img/fig-5-04.png)![img]()

假设一个元组通过txid为99的事务插入到一个页面中。在这种情况下，插入元组的header设置如下。

- Tuple_1:

  **t_xmin** 设置为99，因为此元组由txid 99插入。

  **t_xmax** 被设置为0，因为这个元组还没有被删除或更新。

  **t_cid** 被设置为0，因为这个元组是txid 99插入的第一个元组。

  **t_ctid** 被设置为(0,1)，它指向自身，因为这是最新的元组。

   

*pageinspect*

PostgreSQL提供了一个扩展pageinspect，用于显示数据库page页的内容。

```sql
testdb=# CREATE EXTENSION pageinspect;
CREATE EXTENSION
testdb=# CREATE TABLE tbl (data text);
CREATE TABLE
testdb=# INSERT INTO tbl VALUES('A');
INSERT 0 1
testdb=# SELECT lp as tuple, t_xmin, t_xmax, t_field3 as t_cid, t_ctid 
                FROM heap_page_items(get_raw_page('tbl', 0));
 tuple | t_xmin | t_xmax | t_cid | t_ctid 
-------+--------+--------+-------+--------
     1 |     99 |      0 |     0 | (0,1)
(1 row)
```

 

### 5.3.2. 删除

在删除操作中，从逻辑上删除目标元组。 执行DELETE命令的txid值被设置到元组的t_xmax(图5.5)。

**图. 5.5. 删除tuple**

![Fig. 5.5. Tuple deletion.](http://www.interdb.jp/pg/img/fig-5-05.png)![img]()

假设Tuple_1被txid 111删除。在这种情况下，Tuple_1的header被设置如下。

- Tuple_1:
  **t_xmax** 设置为111。

如果提交txid 111，则Tuple_1不再需要。 一般来说，不需要的元组在PostgreSQL中被称为 **dead tuples**。

dead tuple 最终应该从page页中删除。 清理 dead tuple 被称为**VACUUM**处理，这在[第六章](ch6.md)中介绍。

### 5.3.3. 更新

在更新操作中，PostgreSQL从逻辑上删除最新的元组并插入一个新元组(图5.6)。

**图. 5.6. 更新行两次**

![Fig. 5.6. Update the row twice.](http://www.interdb.jp/pg/img/fig-5-06.png)![img]()

假设由txid 99插入的行由txid 100更新两次。

执行第一个UPDATE命令时，通过将txid 100设置到t_xmax，Tuple_1被逻辑删除，然后插入Tuple_2。 然后，Tuple_1的t_ctid被重写为指向Tuple_2。 Tuple_1和Tuple_2的header如下。

- Tuple_1:

  t_xmax 设置为100

  t_ctid 从(0,1)改成(0,2)。

- Tuple_2:

  t_xmin 设置为100

  t_xmax 设置为0

  t_cid 设置为0

  t_ctid 设置为 (0,2).

当执行第二个UPDATE命令时，跟第一个UPDATE命令一样，Tuple_2在逻辑上被删除并且Tuple_3被插入。 Tuple_2和Tuple_3的header如下。

- Tuple_2:

  t_xmax 设置为100

  t_ctid 从(0, 2) 改成 (0, 3).

- Tuple_3:

  t_xmin 设置为100

  t_xmax 设置为0

  t_cid 设置为1

  t_ctid 设置为 (0,3).

与删除操作一样，如果提交txid 100，则Tuple_1和Tuple_2将为dead tuple，并且如果txid 100被中止，则Tuple_2和Tuple_3将为dead tuple。

### 5.3.4. 空闲空间映射表(Free Space Map)

当插入堆元组或索引元组时，PostgreSQL使用相应表或索引的**FSM**来选择可插入它的page页。

如第1.2.3节所述，所有表和索引都有各自的FSM。 每个FSM在相应的表或索引文件中存储有关每个页的空闲空间信息。

所有FSM都以后缀'fsm'存储，并在必要时加载到共享内存中。



pg_freespacemap

扩展[pg_freespacemap](https://www.postgresql.org/docs/current/static/pgfreespacemap.html)提供指定表/索引的空闲空间。 以下查询显示指定表中每个页的空闲空间比率。

```sql
testdb=# CREATE EXTENSION pg_freespacemap;
CREATE EXTENSION

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

 

------

## 5.4. 事务提交日志 (clog)

PostgreSQL在**事务提交日志(Commit Log)**中保存事务的状态。 提交日志通常称为**clog**，它被分配给共享内存，并在整个事务处理过程中使用。

本节介绍PostgreSQL中事务的状态、clog的操作方式以及clog的维护。 

### 5.4.1 事务状态

PostgreSQL定义了四种事务状态，即 IN_PROGRESS, COMMITTED, ABORTED, 和 SUB_COMMITTED。  

前三种状态是显而易见的。例如，当事务正在进行时，它的状态是IN_PROCESS，等等。  

SUB_COMMITTED用于子事务，其描述在本文档中被省略。 

### 5.4.2. Clog 如何执行

clog 在共享内存中是一个或多个8 KB page页。clog逻辑上是一个数组。 数组的索引对应于相应的事务ID，并且数组中的每个项都保存相应事务的状态。 图5.7显示了clog及其运作方式。

**图. 5.7. clog如何执行**

![Fig. 5.7. How the clog operates.](http://www.interdb.jp/pg/img/fig-5-07.png)![img]()

------

- **T1:** txid 200 commits; txid 200 状态从 IN_PROGRESS 改为 COMMITTED.
- **T2:** txid 201 aborts; txid 201 状态从 IN_PROGRESS 改为 ABORTED.

当前的txid前进并且clog不够存储它时，会追加一个新页。

当需要事务的状态时，调用内部函数。 这些函数读取clog并返回请求的事务的状态。 (另请参见第5.7.1节中的“Hint Bits”)。

### 5.4.3. 维护Clog

当PostgreSQL停库或检查点进程(checkpoint process)运行时，clog的数据将被写入存储在**pg_clog**子目录下的文件中。 (注意，pg_clog将在版本10中重命名为pg_xact)这些文件被命名为0000，0001等。最大文件大小为256 KB。 例如，当clog使用8个page页(第一页至第八页;总大小为64KB)时，它的数据被写入0000(64 KB)，而37页(296 KB)中的数据被写入0000和0001，它们的大小分别为256 KB和40 KB。 

当PostgreSQL启动时，存储在pg_clog文件(pg_xact的文件)中的数据将被加载以初始化clog。 

clog的大小不断增加，因为每当clog被填满时，就会追加一个新页。然而，并不是所有的数据都是必要的。VACUUM处理，在[第6章](ch6.md)中描述，定期删除这样的旧数据(clog的页面和文件)。有关清除clog数据的详细信息，请参阅第6.4节。 

## 5.5. 事务快照

**事务快照(transaction snapshot)**是一个数据集，用于存储有关每个事务在某个时间点是否所有事务都处于活动状态的信息。在这里，活动事务意味着它正在进行或尚未启动。 

PostgreSQL在内部将事务快照的文本表示格式定义为"100：100："。例如，"100：100："表示"小于99的txid不活动，等于或大于100的txid是活动的"。在下面的描述中，使用了这种方便的表示形式。如果您不熟悉它，请参阅下文。 

 

*内置函数txid_current_snapshot及其文本表示格式* 

函数 [txid_current_snapshot](http://www.postgresql.org/docs/current/static/functions-info.html#FUNCTIONS-TXID-SNAPSHOT) 查看当前事务的快照。

```sql
testdb=# SELECT txid_current_snapshot();
 txid_current_snapshot 
-----------------------
 100:104:100,102
(1 row)
```

txid_current_snapshot的文本表示形式为“xmin：xmax：xip_list“，这些内容描述如下。

- xmin

  最早的还在活动的txid。所有之前的事务要么提交且可见，要么回滚而无效。 

- xmax

  尚未分配的txid。所有大于或等于此值的txid在快照时间之前尚未启动，因此不可见。 

- xip_list

  在快照时间活动的txid。 该list只包含xmin和xmax之间的活动txid。 

例如，在快照100:104:100,102‘ 中，xmin是'100'，xmax '104'和xip_list '100,102'。

以下是两个具体示例：

**图. 5.8. 事务快照示例**

![Fig. 5.8. Examples of transaction snapshot representation.](http://www.interdb.jp/pg/img/fig-5-08.png)

第一个例子是'100'。 该快照意味着以下内容(图5.8(a))：

- 等于或小于99的txids不活动，因为xmin是100。
- 等于或大于100的txids是活动的，因为xmax是100。

第二个例子是'100:104:100,102'。 这个快照意味着以下内容(图5.8(b))：

- 等于或小于99的txids不活动。
- 等于或大于104的txids处于活动状态。
- 因为它们存在于xip list中，所以txids 100和102是活动的，而txids 101和103不活动。

事务快照由事务管理器提供。 在READ COMMITTED隔离级别中，只要执行SQL命令，事务就会获得快照; 除此之外(REPEATABLE READ或SERIALIZABLE)，事务只会在执行第一个SQL命令时获取快照。 获取的事务快照用于元组的可见性检查，这在5.7节中有描述。

将获取的快照用于可见性检查时，即使快照中的活动事务实际上committed或aborted，也必须将其视为*in progress*。 此规则很重要，因为它会导致READ COMMITTED和REPEATABLE READ(或SERIALIZABLE)行为之间的差异。 我们在下面的章节中多次提到这个规则。

在本节的其余部分中，在特定场景中描述事务管理器和事务(图5.9)。

**图. 5.9. 事务管理器和事务**

![Fig. 5.9. Transaction manager and transactions.](http://www.interdb.jp/pg/img/fig-5-09.png)![img]()

事务管理器始终保存有关当前正在运行的事务的信息。假设三个事务一个接一个地启动，并且Transaction_A和Transaction_B的隔离级别是READ COMMITTED，而Transaction_C的隔离级别是REPEATABLE READ。 

- **T1:**

   Transaction_A启动并执行第一个SELECT命令。 在执行第一个命令时，Transaction_A请求此时的txid和快照。 在这种情况下，事务管理器分配txid 200，并返回事务快照'200：200：'。

- **T2:**

   Transaction_B启动并执行第一个SELECT命令。 事务管理器分配txid 201，并返回事务快照'200：200：'，因为Transaction_A(txid 200)正在进行。 因此，从Transaction_B中不能看到Transaction_A。

- **T3:**

   Transaction_C启动并执行第一个SELECT命令。 事务管理器分配txid 202，并返回事务快照'200：200：'，因此从Transaction_C中不能看到Transaction_A和Transaction_B。

- **T4:**

   Transaction_A已被提交。 事务管理器删除有关此事务的信息。

- **T5:**

   Transaction_B和Transaction_C执行它们各自的SELECT命令。

   Transaction_B需要事务快照，因为它处于READ COMMITTED级别。 在这种情况下，Transaction_B获取新的快照'201：201：'，因为Transaction_A(txid 200)已提交。 因此，在Transaction_B中Transaction_A不再不可见。

   Transaction_C不需要事务快照，因为它处于REPEATABLE READ级别并使用获得的快照，即'200：200：'。 因此，在Transaction_C中Transaction_A仍然不可见。

   

## 5.6. 可见性检查规则

可见性检查规则是一组规则，通过使用元组的t_xmin和t_xmax、clog和事务快照来确定每个元组是可见的还是不可见 。这些规则太复杂，不好详细解释。 因此，本文档仅描述后续说明所需的规则。 在下文中，我们省略了与子事务相关的规则，并忽略关于t_ctid的讨论，即我们不考虑在事务内更新两次以上元组。

选定的规则数量为10个，可以分为三种情况。

### 5.6.1. t_xmin状态为ABORTED

t_xmin状态为ABORTED的元组始终是不可见的(Rule 1)，因为插入此元组的事务已被中止

```sql
/* t_xmin status = ABORTED */
Rule 1:	IF t_xmin status is 'ABORTED' THEN
			RETURN 'Invisible'
		END IF
```

该规则明确表示为以下表达式。

**Rule 1:** If Status(t_xmin) = ABORTED ⇒ Invisible

### 5.6.2. t_xmin状态为IN_PROGRESS

t_xmin状态为IN_PROGRESS的元组基本上是不可见的(Rule 3和4)，有一个情况下例外。

```sql
/* t_xmin status = IN_PROGRESS */
	IF t_xmin status is 'IN_PROGRESS' THEN
		IF t_xmin = current_txid THEN
Rule 2: 	IF t_xmax = INVALID THEN
				RETURN 'Visible'
Rule 3: 	ELSE /* this tuple has been deleted or updated by the current transaction itself.*/
				RETURN 'Invisible'
			END IF
Rule 4: ELSE   /* t_xmin ≠ current_txid */
			RETURN 'Invisible'
		END IF
	END IF
```

如果这个元组被另一个事务插入，并且t_xmin的状态是IN_PROGRESS，这个元组显然是不可见的(Rule 4)。

如果t_xmin等于当前txid(即，该元组被当前事务插入)并且t_xmax**不**是INVALID，则该元组不可见，因为它已被当前事务更新或删除(Rule 3)。

例外情况是由当前事务插入此元组并且t_xmax为INVALID的情况。 在这种情况下，这个元组在当前事务中是可见的(Rule 2)。

- **Rule 2:** If Status(t_xmin) = IN_PROGRESS ∧ t_xmin = current_txid ∧ t_xmax = INVAILD ⇒ Visible
- **Rule 3:** If Status(t_xmin) = IN_PROGRESS ∧ t_xmin = current_txid ∧ t_xmax ≠ INVAILD ⇒ Invisible
- **Rule 4:** If Status(t_xmin) = IN_PROGRESS ∧ t_xmin ≠ current_txid ⇒ Invisible

### 5.6.3. t_xmin状态为COMMITTED

t_xmin状态为COMMITTED的元组可见(Rules 6,8, 和 9)，但有三种情况例外。

```sql
/* t_xmin status = COMMITTED */
        IF t_xmin status is 'COMMITTED' THEN
Rule 5:     IF t_xmin is active in the obtained transaction snapshot THEN
                RETURN 'Invisible'
Rule 6:     ELSE IF t_xmax = INVALID OR status of t_xmax is 'ABORTED' THEN
                RETURN 'Visible'
            ELSE IF t_xmax status is 'IN_PROGRESS' THEN
Rule 7:         IF t_xmax =  current_txid THEN
                    RETURN 'Invisible'
Rule 8:         ELSE  /* t_xmax ≠ current_txid */
                    RETURN 'Visible'
                END IF
            ELSE IF t_xmax status is 'COMMITTED' THEN
Rule 9:         IF t_xmax is active in the obtained transaction snapshot THEN
                    RETURN 'Visible'
Rule 10:        ELSE
                    RETURN 'Invisible'
                END IF
            END IF
        END IF
```

Rule 6很明显，因为t_xmax是INVALID或ABORTED。下面描述了三种例外情况以及Rule 8和9。

第一个例外情况是t_xmin在获得的事务快照中处于活动状态(Rule 5)。在这种情况下，这个元组是不可见的，因为t_xmin应该被视为in progress。

第二个例外情况是t_xmax是当前的txid(Rule 7)。在这种情况下，和Rule 3一样，这个元组是不可见的，因为它已经被这个事务本身更新或删除。

相反，如果t_xmax的状态是IN_PROGRESS并且t_xmax不是当前的txid(Rule 8)，则该元组可见，因为它尚未被删除。

第三个例外情况是t_xmax的状态是COMMITTED，并且t_xmax在获得的事务快照中不活动(Rule 10)。在这种情况下，这个元组是不可见的，因为它已被另一个事务更新或删除。

相反，如果t_xmax的状态为COMMITTED，但t_xmax在获取的事务快照中处于活动状态(Rule 9)，则该元组可见，因为t_xmax应视为in progress。

- **Rule 5:** If Status(t_xmin) = COMMITTED ∧ Snapshot(t_xmin) = active ⇒ Invisible
- **Rule 6:** If Status(t_xmin) = COMMITTED ∧ (t_xmax = INVALID ∨ Status(t_xmax) = ABORTED) ⇒ Visible
- **Rule 7:** If Status(t_xmin) = COMMITTED ∧ Status(t_xmax) = IN_PROGRESS ∧ t_xmax = current_txid ⇒ Invisible
- **Rule 8:** If Status(t_xmin) = COMMITTED ∧ Status(t_xmax) = IN_PROGRESS ∧ t_xmax ≠ current_txid ⇒ Visible
- **Rule 9:** If Status(t_xmin) = COMMITTED ∧ Status(t_xmax) = COMMITTED ∧ Snapshot(t_xmax) = active ⇒ Visible
- **Rule 10:** If Status(t_xmin) = COMMITTED ∧ Status(t_xmax) = COMMITTED ∧ Snapshot(t_xmax) ≠ active ⇒ Invisible

------

## 5.7. 可见性检查

本节介绍PostgreSQL如何执行可见性检查，即如何选择给定事务中适当版本的堆元组。 本节还介绍了PostgreSQL如何防止ANSI SQL-92标准中定义的异常：脏读Dirty Reads, 不可重复读Repeatable Reads 和 幻读Phantom Reads.。

### 5.7.1. 可见性检查

图5.10显示了描述可见性检查的场景。

**图 5.10. 可见性检查情景示例**

![Fig. 5.10. Scenario to describe visibility check.](http://www.interdb.jp/pg/img/fig-5-10.png)![img]()

在图5.10所示的场景中，SQL命令按以下时间顺序执行。

**T1:** 开始事务 (txid 200)

**T2:** 开始事务 (txid 201)

**T3:** txid 200 和 201执行SELECT命令

**T4:** txid 200执行UPDATE命令

**T5:** txid 200 and 201执行SELECT命令

**T6:** 提交 txid 200

**T7:** txid 201执行SELECT命令

为了简化描述，假设只有两个事务，即txid 200和201. txid 200的隔离级别是READ COMMITTED，并且txid 201的隔离级别是READ COMMITTED或REPEATABLE READ。

我们将探索SELECT命令如何为每个元组执行可见性检查。

**SELECT commands of T3:**

在T3，表tbl中只有一个Tuple_1，并且Rule 6可见; 因此，两个事务中的SELECT命令都返回'Jekyll'。

- Rule6(Tuple_1) ⇒ Status(t_xmin:199) = COMMITTED ∧ t_xmax = INVALID ⇒ Visible

![code-5-01](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch5/code-5-01.png?raw=true)


**SELECT commands of T5:**

首先，我们研究由txid 200执行的SELECT命令.Tuple_1在Rule 7中是不可见的，而Tuple_2在Rule 2中是可见的; 因此，这个SELECT命令返回'Hyde'。

- Rule7(Tuple_1): Status(t_xmin:199) = COMMITTED ∧ Status(t_xmax:200) = IN_PROGRESS ∧ t_xmax:200 = current_txid:200 ⇒ Invisible
- Rule2(Tuple_2): Status(t_xmin:200) = IN_PROGRESS ∧ t_xmin:200 = current_txid:200 ∧ t_xmax = INVAILD ⇒ Visible

```sql
testdb=# -- txid 200
testdb=# SELECT * FROM tbl;
 name 
------
 Hyde
(1 row)
```

另一方面，在由txid 201执行的SELECT命令中，Tuple_1在Rule 8中可见，而Tuple_2在Rule 4中不可见; 因此，这个SELECT命令返回'Jekyll'。

- Rule8(Tuple_1): Status(t_xmin:199) = COMMITTED ∧ Status(t_xmax:200) = IN_PROGRESS ∧ t_xmax:200 ≠ current_txid:201 ⇒ Visible
- Rule4(Tuple_2): Status(t_xmin:200) = IN_PROGRESS ∧ t_xmin:200 ≠ current_txid:201 ⇒ Invisible

```sql
testdb=# -- txid 201
testdb=# SELECT * FROM tbl;
  name  
--------
 Jekyll
(1 row)
```

如果更新的元组在提交之前在其他事务中可见，则它们被称为**脏读 Dirty Reads**，也称为 **wr-conflicts**。 但是，如上所示，在PostgreSQL中，脏读不会出现在任何隔离级别中。

**SELECT command of T7:**

在下文中，将描述T7的SELECT命令在两个隔离级别中的行为。

首先，我们探讨何时txid 201处于READ COMMITTED级别。 在这种情况下，由于事务快照是“201：201：”，因此txid 200被视为COMMITTED。 因此，Tuple_1在Rule 10不可见，Tuple_2在Rule 6可见，并且SELECT命令返回'Hyde'。

- Rule10(Tuple_1): Status(t_xmin:199) = COMMITTED ∧ Status(t_xmax:200) = COMMITTED ∧ Snapshot(t_xmax:200) ≠ active ⇒ Invisible
- Rule6(Tuple_2): Status(t_xmin:200) = COMMITTED ∧ t_xmax = INVALID ⇒ Visible

```sql
testdb=# -- txid 201 (READ COMMITTED)
testdb=# SELECT * FROM tbl;
 name 
------
 Hyde
(1 row)
```

请注意，在txid 200提交之前和之后执行的SELECT命令的结果不同。 这通常称为**不重复读 Non-Repeatable Reads**。

相反，当txid 201处于REPEATABLE READ级别时，由于事务快照为'200：200：'，所以必须将txid 200视为IN_PROGRESS。 因此，Tuple_1在Rule 9中可见，Tuple_2在Rule 5中不可见，并且SELECT命令返回'Jekyll'。 请注意，在REPEATABLE READ(和SERIALIZABLE)级别中不会出现不可重复读取。

- Rule9(Tuple_1): Status(t_xmin:199) = COMMITTED ∧ Status(t_xmax:200) = COMMITTED ∧ Snapshot(t_xmax:200) = active ⇒ Visible
- Rule5(Tuple_2): Status(t_xmin:200) = COMMITTED ∧ Snapshot(t_xmin:200) = active ⇒ Invisible

```sql
testdb=# -- txid 201 (REPEATABLE READ)
testdb=# SELECT * FROM tbl;
  name  
--------
 Jekyll
(1 row)
```

 

*Hint Bits*

为了获得一个事务的状态，PostgreSQL在内部提供了三个函数，即TransactionIdIsInProgress，TransactionIdDidCommit和TransactionIdDidAbort。 实现这些功能是为了减少频繁访问clog，例如缓存。 但是，如果在检查每个元组时执行它们，瓶颈将会发生。

为了处理这个问题，PostgreSQL使用*hint bits*，如下所示。

```c
#define HEAP_XMIN_COMMITTED       0x0100   /* t_xmin committed */
#define HEAP_XMIN_INVALID         0x0200   /* t_xmin invalid/aborted */
#define HEAP_XMAX_COMMITTED       0x0400   /* t_xmax committed */
#define HEAP_XMAX_INVALID         0x0800   /* t_xmax invalid/aborted */
```

在读或写元组时，PostgreSQL尽可能将hint bits设置为元组的t_informask。 例如，假设PostgreSQL检查元组的t_xmin的状态并获得COMMITTED状态。 在这种情况下，PostgreSQL将hint bits HEAP_XMIN_COMMITTED设置为元组的t_infomask。 如果已经设置了hint bits，则不再需要TransactionIdDidCommit和TransactionIdDidAbort。 因此，PostgreSQL可以高效地检查每个元组的t_xmin和t_xmax的状态。

 

### 5.7.2. PG中 REPEATABLE READ 级别的幻读

按ANSI SQL-92标准定义的REPEATABLE READ允许**幻读 Phantom Reads**。 但是，PostgreSQL的实现不允许。 原则上，SI不允许幻读。

假设两个事务，即Tx_A和Tx_B同时运行。 它们的隔离级分别是READ COMMITTED和REPEATABLE READ，它们的txids分别是100和101。 首先，Tx_A插入一个元组。 然后，提交。 插入的元组的t_xmin为100.接下来，Tx_B执行SELECT命令; 然而，由Tx_A插入的元组在Rule 5中不可见。因此，幻读不会发生。

- Rule5(new tuple): Status(t_xmin:100) = COMMITTED ∧ Snapshot(t_xmin:100) = active ⇒ Invisible

![code-5-02](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch5/code-5-02.png?raw=true)



## 5.8. 防止 Lost Updates


**Lost Update**(也称为**ww-conflict**)是当并发事务更新相同行时发生的异常，并且在REPEATABLE READ和SERIALIZABLE级别都必须防止出现这种异常。 本节介绍PostgreSQL如何防止 Lost Updates 并给出示例。

### 5.8.1. 并发更新

当执行UPDATE命令时，内部调用ExecUpdate函数。 ExecUpdate的伪代码如下所示：

 

*伪代码: ExecUpdate*

```c
[1]	FOR each row that will be updated by this UPDATE command
[2]		WHILE true

			/* The First Block */
[3]			IF the target row is being updated THEN
[4]				WAIT for the termination of the transaction that updated the target row

[5]				IF (the status of the terminated transaction is COMMITTED)
					AND (the isolation level of this transaction is REPEATABLE READ or SERIALIZABLE) THEN
[6]					ABORT this transaction  /* First-Updater-Win */
				ELSE 
[7]					GOTO step (2)
				END IF

			/* The Second Block */
[8]			ELSE IF the target row has been updated by another concurrent transaction THEN
[9]				IF (the isolation level of this transaction is READ COMMITTED THEN
[10]				UPDATE the target row
				ELSE
[11]				ABORT this transaction  /* First-Updater-Win */
			END IF

			/* The Third Block */
			ELSE  /* The target row is not yet modified or has been updated by a terminated transaction. */
[12]			UPDATE the target row
			END IF
		END WHILE 
	END FOR 
```

(1)  获取将由此UPDATE命令更新的每一行。

(2) 重复以下过程，直到目标行被更新(或该事务被中止)。

(3) 如果目标行正在更新，请继续步骤(3); 否则，继续步骤(8)。

(4) 等待更新目标行的事务终止，因为PostgreSQL在SI中使用 *first-updater-win* 方案。

(5) 如果更新目标行的事务的状态为COMMITTED，并且此事务的隔离级别为REPEATABLE READ(或SERIALIZABLE)，则继续步骤(6); 否则，继续步骤(7)。

(6) 中止此事务以防止丢失更新。

(7) 继续步骤(2)并尝试更新下一轮中的目标行。

(8) 如果目标行已被另一个并发事务更新，则继续步骤(9); 否则，继续步骤(12)。

(9) 如果此事务的隔离级别为READ COMMITTED，则继续执行步骤(10); 否则，进入步骤(11)。

(10) 更新目标行，然后继续步骤(1)。

(11) 中止此事务以防止Lost Update。

(12) 更新目标行，并继续执行步骤(1)，因为目标行尚未修改或已被终止的事务更新，即存在 ww-confict。

 

该函数为每个目标行执行更新操作。 它有一个while循环来更新每一行，while循环如图5.11所示。

**图5.11 ExecUpdate中三个内部块**

![Fig. 5.11. Three internal blocks in ExecUpdate.](http://www.interdb.jp/pg/img/fig-5-11.png)![img]()

[1] 目标航正在被更新 (图. 5.11[1])

“Being updated “意味着该行被另一个并发事务更新，并且其事务没有终止。 在这种情况下，当前事务必须等待终止更新目标行的事务，因为PostgreSQL的SI使用 **first-updater-win** 方案。 例如，假设事务Tx_A和Tx_B同时运行，并且Tx_B尝试更新一行; 但是，Tx_A已更新它并仍在进行中。 在这种情况下，Tx_B等待Tx_A的终止。

在更新目标行的事务提交之后，当前事务的更新操作继续。 如果当前事务处于READ COMMITTED级别，则目标行将被更新; 否则(REPEATABLE READ或SERIALIZABLE)，当前事务立即中止以防止丢失更新。

[2] 目标行已由并发事务更新 (图. 5.11[2])

当前事务尝试更新目标元组; 但是，另一个并发事务已更新目标行并已提交。 在这种情况下，如果当前事务处于READ COMMITTED级别，则目标行将被更新; 否则，立即中止当前事务以防止丢失更新。

[3] 没有冲突 (图. 5.11[3])

当没有冲突时，当前事务可以更新目标行。

 

 *first-updater-win / first-commiter-win*

基于SI的PostgreSQL的并发控制使用*first-updater-win*机制。 相反，如下一节所述，PostgreSQL的SSI使用*first-committer-win*机制。

 

### 5.8.2. 示例

以下给出三个示例。 第一个和第二个示例描述目标行正在更新时的行为，第三个示例描述目标行更新完成时的行为。

**示例 1:**

事务Tx_A和Tx_B更新同一个表中的同一行，其隔离级别为READ COMMITTED。

![code-5-03](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch5/code-5-03.png?raw=true)

Tx_B执行如下。

1) 执行UPDATE命令后，由于目标元组正在被Tx_A更新(ExecUpdate中的Step(4))，Tx_B应等待Tx_A的终止。

2) Tx_A提交后，Tx_B尝试更新目标行(ExecUpdate中的Step(7))。

3) 在第二轮ExecUpdate中，通过Tx_B再次更新目标行(ExecUpdate中的Step(2)，(8)，(9)，(10))。

**示例 2:**

Tx_A和Tx_B更新同一个表中的同一行，其隔离级别分别为READ COMMITTED和REPEATABLE READ。

![code-5-04](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch5/code-5-04.png?raw=true)

Tx_B执行如下。

1) 执行UPDATE命令后，Tx_B应等待Tx_A的终止(ExecUpdate中的步骤(4))。

2) 在Tx_A提交之后，由于目标行已更新并且此事务的隔离级别为REPEATABLE READ(ExecUpdate中的步骤(5)和(6))，所以Tx_B中止以防止冲突。

**示例 3:**

Tx_B(REPEATABLE READ)尝试更新已由Tx_A更新提交的目标行。 在这种情况下，Tx_B被中止(ExecUpdate中的Step(2)，(8)，(9)和(11))。

![code-5-05](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch5/code-5-05.png?raw=true)



## 5.9. 可串行化的快照隔离

自9.1版以来，可串行化的快照隔离(Serializable Snapshot Isolation(SSI))已嵌入到SI中，以实现真正的 SERIALIZABLE 隔离级别。 由于SSI的解释并不简单，只介绍大致轮廓。 有关详情，请参见[[2\]](http://www.interdb.jp/pg/pgsql05.html#_5.ref.2)。

在下文中，在没有定义情况下使用下面所示的技术术语。 如果您对这些术语不熟悉，请参阅 [[1](http://www.interdb.jp/pg/pgsql05.html#_5.ref.1), [3](http://www.interdb.jp/pg/pgsql05.html#_5.ref.3)]。

- *优先图 precedence graph*(也称为 *依赖图 dependency graph* 和 *串行图 serialization graph*)
- *序列化异常 serialization anomalies*(例如 *Write-Skew*)

### 5.9.1. SSI 基本实现策略

如果在优先图中存在一个由某些冲突产生的循环，则会出现序列化异常。 这是用最简单的异常来解释，即 Write-Skew。

图5.12(1)显示了一个时间表。 在这里，Transaction_A读Tuple_B，而Transaction_B读Tuple_A。 然后，Transaction_A写Tuple_A，Transaction_B写Tuple_B。 在这种情况下，存在两个rw-conflicts，并且它们在此时间表的优先图中形成一个循环，如图5.12(2)所示。 因此，该时间表具有序列化异常，即Write-Skew。

**图. 5.12. Write-Skew 时间表和他的优先图**

![Fig. 5.12. Write-Skew schedule and its precedence graph.](http://www.interdb.jp/pg/img/fig-5-12.png)![img]()

从概念上讲，有三种类型的冲突：wr-conflicts (脏读Dirty Reads), ww-conflicts (丢失更新Lost Updates), 和 rw-conflicts。 但是，wr- conficts和ww-conflicts并不需要考虑，因为如前面的章节所示，PostgreSQL可以防止这种冲突。 因此，PostgreSQL中的SSI实现只需要考虑rw-conflicts。

PostgreSQL采用以下策略来实现SSI：

1. 将事务访问的所有对象(tuples, pages, relations)记录为SIREAD锁。
2. 每当写入任何堆元组或索引元组时，都使用SIREAD锁来检测rw-conflicts。
3. 如果通过检查检测到的 re-conflict 检测到序列化异常 ，则中止事务。

### 5.9.2. PostgreSQL中 SSI 的实现

为了实现上述策略，PostgreSQL实现了许多功能和数据结构。 但是，这里我们只使用两种数据结构：**SIREAD locks** 和 **rw-conflicts** 来描述SSI机制。 它们存储在共享内存中。



为简单起见，本文档中省略了一些重要的数据结构，例如SERIALIZABLEXACT。 因此，函数的解释，即CheckTargetForConflictOut，CheckTargetForConflictIn和PreCommit_CheckForSerializationFailure也是非常简单的描述。 例如，我们指出哪些功能检测到冲突; 但是，如何检测冲突并没有详细解释。 如果你想知道细节，请参考源代码：predicate.c。

 

**SIREAD locks:**

SIREAD锁在内部称为谓词锁，它是一对对象和(虚拟)txids，用于存储关于谁访问了哪个对象的信息。 请注意，虚拟txid的介绍被省略。 后面的内容使用txid代替虚拟txid来描述。

每当在SERIALIZABLE模式下执行一个DML命令时，由CheckTargetForConflictsOut函数创建SIREAD锁。 例如，如果txid 100读取给定表的Tuple_1，则创建SIREAD锁{Tuple_1，{100}}。 如果另一个事务，例如 txid 101，读取Tuple_1，SIREAD锁更新为{Tuple_1，{100,101}}。 请注意，当读取索引页时，还会创建SIREAD锁，因为索引页仅在使用 [Index-Only Scans](https://www.postgresql.org/docs/current/static/indexes-index-only-scans.html) 时才读取而不读取表页。

SIREAD锁具有三个级别：tuple, page 和 relation。 如果创建了单个page页内所有元组的SIREAD锁，则它们被聚合为该页的单个SIREAD锁，并且关联元组的所有SIREAD锁被释放(移除)，以减少存储空间。 所有读取的页都是如此。

为索引创建SIREAD锁时，page页级的SIREAD锁从一开始就创建。 使用顺序扫描时，不管是否存在索引和/或WHERE子句，都会从头开始创建 relation级别的SIREAD锁。 请注意，在某些情况下，此实现可能导致错误地检测到序列化异常。 细节在5.9.4节中描述。

**rw-conflicts:**

A rw-conflict is a triplet of an SIREAD lock and two txids that reads and writes the SIREAD lock.

rw-conflict 是SIREAD锁和两个读取和写入SIREAD锁的txids的三元组。

每当在SERIALIZABLE模式下执行INSERT，UPDATE或DELETE命令时，都会调用CheckTargetForConflictsIn函数，并且在通过检查SIREAD锁来检测冲突时会创建rw-conflict。

例如，假设txid 100读取Tuple_1，然后txid 101更新Tuple_1。 在这种情况下，由txid 101中的UPDATE命令调用的CheckTargetForConflictsIn函数在txid 100和101之间检测到与Tuple_1的rw-conflict，然后创建rw-conflict {r=100, w=101, {Tuple_1}}。

CheckTargetForConflictOut和CheckTargetForConflictIn函数以及在COMMIT命令在SERIALIZABLE模式下执行时调用的PreCommit_CheckForSerializationFailure函数都使用创建的 rw-conflict 检查序列化异常。 如果他们检测到异常情况，则只有第一次提交的事务被提交，其他事务将被中止(根据**first-committer-win**规则)。



### 5.9.3. SSI 如何执行 

在这里，我们描述SSI如何解决Write-Skew异常。 我们使用如下所示的表tbl：

```sql
testdb=# CREATE TABLE tbl (id INT primary key, flag bool DEFAULT false);
testdb=# INSERT INTO tbl (id) SELECT generate_series(1,2000);
testdb=# ANALYZE tbl;
```

事务Tx_A和Tx_B执行以下命令(图5.13)。

**图. 5.13. Write-Skew scenario.**

![Fig. 5.13. Write-Skew scenario.](http://www.interdb.jp/pg/img/fig-5-13.png)![img]()

假设所有的命令都使用索引扫描。 因此，执行这些命令时，它们将读取堆元组和索引页，其中每个元组都包含指向相应堆元组的索引元组。 见图5.14。

**图. 5.14. 图5.13所示场景中索引与表之间的关系**

![Fig. 5.14. Relationship between the index and table in the scenario shown in Fig. 5.13.](http://www.interdb.jp/pg/img/fig-5-14.png)![img]()

**T1**：Tx_A执行SELECT命令。 该命令读取一个堆元组(Tuple_2000)和一个主键(Pkey_2)。

**T2**：Tx_B执行一个SELECT命令。 该命令读取一个堆元组(Tuple_1)和一个主键(Pkey_1)。

**T3**：Tx_A执行UPDATE命令更新Tuple_1。

**T4**：Tx_B执行UPDATE命令更新Tuple_2000。

**T5**：Tx_A提交。

**T6**：Tx_B提交; 但是，由于写偏异常而中止。

图5.15显示了PostgreSQL如何检测和解决上述场景中描述的Write-Skew异常。

**图5.15. SIREAD锁和rw-conflicts，以及图5.13中所示场景的时间表**

![Fig. 5.15. SIREAD locks and rw-conflicts, and schedule of the scenario shown in Fig. 5.13.](http://www.interdb.jp/pg/img/fig-5-15.png)![img]()

**T1**：

当Tx_A执行SELECT命令时，CheckTargetForConflictsOut创建SIREAD锁。在这种情况下，函数创建两个SIREAD锁：L1和L2。

L1和L2分别与Pkey_2和Tuple_2000相关联。

**T2**：

当Tx_B执行SELECT命令时，CheckTargetForConflictsOut会创建两个SIREAD锁：L3和L4。

L3和L4分别与Pkey_1和Tuple_1相关联。

**T3**：

当Tx_A执行UPDATE命令时，在ExecUpdate前后调用CheckTargetForConflictsOut和CheckTargetForConflictsIN。

在这种情况下，CheckTargetForConflictsOut什么都不做。

CheckTargetForConflictsIn创建rw-confict C1，这是Tx_B和Tx_A之间Pkey_1和Tuple_1两者的冲突，因为Pkey_1和Tuple_1都被Tx_B读取并由Tx_A写入。

**T4**：

当Tx_B执行UPDATE命令时，CheckTargetForConflictsIn创建rw-conflict C2，这是Tx_A和Tx_B之间的Pkey_2和Tuple_2000的冲突。

在这种情况下，C1和C2在优先图中创建一个循环；因此，Tx_A和Tx_B处于 non-serializable 状态。但是，事务Tx_A和Tx_B都未提交，因此CheckTargetForConflictsIn不会中止Tx_B。请注意，这是因为PostgreSQL的SSI实现基于 *first-committer-win* 方案。

**T5**：

当Tx_A尝试提交时，调用 PreCommit_CheckForSerializationFailure。该函数可以检测序列化异常，并且可以执行提交操作。在这种情况下，Tx_A被提交，因为Tx_B仍在进行中。

**T6**：

当Tx_B尝试提交时，PreCommit_CheckForSerializationFailure 检测到序列化异常并且Tx_A已经提交；因此，Tx_B被中止。

另外，如果在Tx_A已被提交(在T5)之后由Tx_B执行UPDATE命令，则Tx_B立即中止，因为由Tx_B的UPDATE命令调用的CheckTargetForConflictsIn检测到串行异常(图5.16(1))。

如果在T6执行SELECT命令而不是COMMIT，则Tx_B立即中止，因为Tx_B的SELECT命令调用CheckTargetForConflictsOut检测到序列化异常(图5.16(2))。

**图. 5.16. 其他 Write-Skew 场景**

![Fig. 5.16. Other Write-Skew scenarios.](http://www.interdb.jp/pg/img/fig-5-16.png)![img]()

 

这个 [Wiki](https://wiki.postgresql.org/wiki/SSI) 解释了几个更复杂的异常。 

 

### 5.9.4. False-Positive 序列化异常

在SERIALIZABLE模式下，并发事务的可串行化始终得到充分保证，因为从未检测到 false-negative 序列化异常。 但是，在某些情况下，可以检测到 false-positive 异常。 因此，在使用SERIALIZABLE模式时，用户应该牢记这一点。 在下文中，描述了PostgreSQL检测到 false-positive 异常的情况。

图5.17展示了 false-positive 序列化异常的情况。

**图. 5.17. false-positive 序列化异常示例**

![Fig. 5.17. Scenario where false-positive serialization anomaly occurs.](http://www.interdb.jp/pg/img/fig-5-17.png)![img]()

当使用顺序扫描时，正如SIREAD锁的解释中所述，PostgreSQL创建一个relation级别的SIREAD锁。 图5.18(1)展示了PostgreSQL使用顺序扫描时的SIREAD锁和 rw-conflict。 在这种情况下，会创建与tbl的SIREAD锁相关的rw-conflict C1和C2，并在优先图中创建一个循环。 因此，检测到false-positive Write-Skew异常(即使没有冲突，Tx_A或Tx_B也会中止)。

**图. 5.18. False-positive 异常 (1) – 使用顺序扫描**

![Fig. 5.18. False-positive anomaly (1) – Using sequential scan.](http://www.interdb.jp/pg/img/fig-5-18.png)![img]()

即使使用索引扫描，如果事务Tx_A和Tx_B都获得相同的索引SIREAD锁，PostgreSQL将检测到 false-positive 异常。 图5.19给出了这种情况。 假设索引页Pkey_1包含两个索引项，其中一个指向Tuple_1，另一个指向Tuple_2。 当Tx_A和Tx_B执行相应的SELECT和UPDATE命令时，Pkey_1被Tx_A和Tx_B读写。 在这种情况下，两个与Pkey_1关联的 rw-conflict C1和C2在优先图中创建一个循环; 因此，检测到 false-positive Write-Skew 异常。 (如果Tx_A和Tx_B获取不同索引页的SIREAD锁，则不会检测到 false-positive，并且可以提交这两个事务。)

**图. 5.19. False-positive 异常 (2) – 索引扫描使用相同的索引页**

![Fig. 5.19. False-positive anomaly (2) – Index scan using the same index page.](http://www.interdb.jp/pg/img/fig-5-19.png)![img]()

------

## 5.10. 维护

PostgreSQL的并发控制机制需要以下过程维护。

1. 删除dead tuple和指向对应的dead tuple的索引元组。 
2. 删除clog不必要的部分。 
3. 冻结旧txid。 
4. 更新FSM、VM和统计信息。 

第5.3.2和5.4.3节分别介绍了第一和第二条。 第三条与事务ID txid wraparound 问题有关，在下面的小节中对此进行了简要介绍。

在PostgreSQL中，VACUUM处理负责这些过程。[第六章](ch6.md)介绍VACUUM处理。

### 5.10.1. FREEZE 处理

在这里，我们描述 txid wraparound 问题。

假定 txid 100 插入元组 Tuple_1，即 Tuple_1 的 t_xmin 为 100。服务器运行了很长时间，Tuple_1 没有被修改。 当前 txid 为21亿+100，并执行 SELECT 命令。 此时，Tuple_1 可见，因为 txid 100 是过去。 然后，执行相同的SELECT 命令; 此时，目前的 txid 为21亿+101。然而，Tuple_1 不再可见，因为 txid 100 在未来(图5.20)。 这是PostgreSQL中所谓的 *transaction wraparound problem*。

**图. 5.20. Wraparound 问题**

![Fig. 5.20. Wraparound problem.](http://www.interdb.jp/pg/img/fig-5-20.png)![img]()

为了解决这个问题，PostgreSQL引入了一个名为 *frozen txid* 的概念，并实现了  *FREEZE* 的过程。

在PostgreSQL中，定义了一个 frozen txid，它是一个特殊的保留 txid 2，它总是比所有其他 txid 更早。 换句话说，frozen txid 总是不活动和可见的。

freeze 处理由 vacuum 调用。 如果 t_xmin 值早于当前 txid 减去 [vacuum_freeze_min_age](https://www.postgresql.org/docs/current/static/runtime-config-client.html#GUC-VACUUM-FREEZE-MIN-AGE)(缺省值为5000万)，则 freeze 处理将扫描所有表文件并将元组的t_xmin重写为 frozen txid。 这在[第六章](http://www.interdb.jp/pg/pgsql06.html)中有更详细的解释。

例如，如图5.21 a)所示，当前txid为5000万，freeze 处理由 VACUUM 调用。 在这种情况下，Tuple_1和Tuple_2的t_xmin都被重写为2。

在9.4版本或更高版本中，将XMIN_FROZEN位设置到元组的t_infomask字段，而不是将元组的t_xmin重写为frozen txid(图5.21 b)。

**图. 5.21. Freeze 处理**

![Fig. 5.21. Freeze process.](http://www.interdb.jp/pg/img/fig-5-21.png)![img]()

## 参考

[1] Abraham Silberschatz, Henry F. Korth, and S. Sudarshan, "[Database System Concepts](https://www.amazon.com//dp/0073523321)", McGraw-Hill Education, ISBN-13: 978-0073523323

[2] Dan R. K. Ports, and Kevin Grittner, "[Serializable Snapshot Isolation in PostgreSQL](https://drkp.net/papers/ssi-vldb12.pdf)", VDBL 2012

[3] Thomas M. Connolly, and Carolyn E. Begg, "[Database Systems](https://www.amazon.com/dp/0321523067)", Pearson, ISBN-13: 978-0321523068
