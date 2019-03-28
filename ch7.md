## 第七章 HOT(Heap-Only-Tuple)技术和仅索引扫描(Index-Only Scans)

本章介绍与索引扫描相关的两个功能，即 heap only tuple 和 index-only scans.。

## 7.1. Heap Only Tuple (HOT)

HOT在8.3版本中实现，当更新的行与旧行将存储在同一个表页中时，HOT能有效使用索引页和表页。 HOT还降低了VACUUM处理的必要性。

由于HOT的详细信息在源代码目录的[README.HOT](https://github.com/postgres/postgres/blob/master/src/backend/access/heap/README.HOT)中描述，本章简要介绍了HOT。 首先，第7.1.1节描述了如何在不使用HOT的情况下更新一行，以了解它解决的问题。 接下来，第7.1.2节描述HOT如何执行。

### 7.1.1. 不使用HOT时更新一行

假设表'tbl'有两列：'id'和'data'; 'id'是'tbl'的主键。

```sql
testdb=# \d tbl
                Table "public.tbl"
 Column |  Type   | Collation | Nullable | Default 
--------+---------+-----------+----------+---------
 id     | integer |           | not null | 
 data   | text    |           |          | 
Indexes:
    "tbl_pkey" PRIMARY KEY, btree (id)
```

表'tbl'有1000个元组; 其ID为'1000'的最后一个元组存储在表的第5页中。 最后一个元组指向相应的索引元组，其中键为'1000'，其tid为'(5,1)'。 参考图7.1(a)。

**图. 7.1. Update a row without HOT**

![Fig. 7.1. Update a row without HOT](imgs/ch7/fig-7-01.png)

我们考虑如何在不使用HOT的情况下更新最后一个元组。

```sql
testdb=# UPDATE tbl SET data = 'B' WHERE id = 1000;
```

在这种情况下，PostgreSQL不仅插入新的表元组，而且还插入索引页中的新索引元组。 参照图7.1(b)。

索引元组的插入消耗索引页空间，并且索引元组的插入和vacuum成本都很高。 HOT减少了这些问题的影响。

### 7.1.2. HOT 如何执行

当用HOT更新行时，如果更新的行与旧行将存储在同一个表页中，则PostgreSQL不会插入相应的索引元组，并将HEAP_HOT_UPDATED位和HEAP_ONLY_TUPLE位分别设置为旧元组和新元组的t_informask2字段。 参考图7.2和7.3。

**图. 7.2. Update a row with HOT**

![Fig. 7.2. Update a row with HOT](imgs/ch7/fig-7-02.png)

例如，在这种情况下，分别将“Tuple_1”和“Tuple_2”设置为HEAP_HOT_UPDATED位和HEAP_ONLY_TUPLE位。

此外，无论是否执行 *修剪 pruning* 和 *整理碎片 defragmentation* 过程(将在下面介绍)，都会使用HEAP_HOT_UPDATED和HEAP_ONLY_TUPLE位。

**图. 7.3. HEAP_HOT_UPDATED and HEAP_ONLY_TUPLE bits**

![Fig. 7.3. HEAP_HOT_UPDATED and HEAP_ONLY_TUPLE bits](imgs/ch7/fig-7-03.png)

在下面，描述了PostgreSQL在使用HOT更新元组之后使用索引扫描访问更新元组的方式。 参考图7.4(a)。

**图. 7.4. 修剪行指针**

![Fig. 7.4. Pruning of the line pointers](imgs/ch7/fig-7-04.png)

(1) 找到指向目标元组的索引元组。

(2) 访问从获取索引元组中指向的行指针'[1]'。

(3) 读“Tuple_1”。

(4) 通过'Tuple_1'的t_ctid读'Tuple_2'。

在这种情况下，PostgreSQL读取两个元组“Tuple_1”和“Tuple_2”，并使用第5章中描述的并发控制机制来决定哪些元组可见。

但是，如果表页中的dead tuple被清理，则会出现问题。 例如，在图7.4(a)中，如果'Tuple_1'因为它是一个dead tuple而被清理，'Tuple_2'不能从索引访问。

为了解决这个问题，在适当的时候，PostgreSQL将指向旧元组的行指针重定向到指向新元组的行指针。 在PostgreSQL中，这个处理被称为**修剪 pruning**。 图7.4(b)描述了PostgreSQL在修剪后如何访问更新的元组。

(1) 找到索引元组。

(2) 访问从获取索引元组中指向的行指针'[1]'。

(3) 通过重定向的行指针访问指向“Tuple_2”的行指针'[2]'。

(4) 读取从行指针'[2]'指向的'Tuple_2'。

如果可能，执行SQL命令(如SELECT，UPDATE，INSERT和DELETE)时将执行修剪处理。 本章没有描述确切的执行时间，因为它非常复杂。 详细信息在[README.HOT](https://github.com/postgres/postgres/blob/master/src/backend/access/heap/README.HOT)文件中描述。

如果可能的话，PostgreSQL会在修剪过程中在适当的时候清理dead tuple。 在PostgreSQL的文档中，这种处理称为碎片整理。 图7.5描述了HOT的**碎片整理 defragmentation**.。

**图. 7.5. Defragmentation of the dead tuples**

![Fig. 7.5. Defragmentation of the dead tuples](imgs/ch7/fig-7-05.png)

请注意，碎片整理的成本低于正常VACUUM处理的成本，因为碎片整理不涉及删除索引元组。

因此，使用HOT可以减少页索引和表的消耗; 这也减少了VACUUM处理必须处理的元组数量。 因此，HOT对性能有很好的影响，因为它通过更新和VACUUM处理的必要性最终减少了索引元组的插入次数。

 

 *HOT 不可用情况*

为了清楚地理解HOT如何执行，描述一下HOT不可用的情况。

当更新的元组存储在另一个不存储旧元组的页中时，指向元组的索引元组也被插入到索引页中。 参考图7.6(a)。

当索引元组的key值被更新时，新的索引元组被插入到索引页中。 参照图7.6(b)。

**图. 7.6. HOT不可用的情况**

![Fig. 7.6. The Cases in which HOT is not available](imgs/ch7/fig-7-06.png)

 

*与HOT相关的统计信息*

 [pg_stat_all_tables](https://www.postgresql.org/docs/current/static/monitoring-stats.html#PG-STAT-ALL-TABLES-VIEW) 视图为每个表提供统计值。 另请参阅此[扩展](https://github.com/s-hironobu/pg_stats)。

 

## 7.2. Index-Only Scans

为了减少I/O(输入/输出)成本，index-only scans(通常称为 index-only access)直接使用索引键而不访问相应的表页，当SELECT语句的所有目标条目都包含在 索引键。几乎所有的商业RDBMS都提供这种技术，如DB2和Oracle。 PostgreSQL从9.2版本开始引入这个选项。

下面用一个具体的例子，描述PostgreSQL中 index-only scans。

假设这个例子如下：

- Table definition

  我们有一个表'tbl'，其定义如下所示：

  ```sql
  testdb=# \d tbl
        Table "public.tbl"
   Column |  Type   | Modifiers 
  --------+---------+-----------
   id     | integer | 
   name   | text    | 
   data   | text    | 
  Indexes:
      "tbl_idx" btree (id, name)
  ```

- Index

  表'tbl'有一个索引'tbl_idx'，它由两列组成：'id'和'name'。

- Tuples

  'tbl'已经插入了元组。

  'Tuple_18'，其id为'18'，name为'Queen'，存储在第0页。

  'Tuple_19'，其id为'19'，name为'BOSTON'，存储在第一页。

- Visibility

  第0页中的所有元组始终可见; 第一页中的元组并不总是可见的。 请注意，每个页的可见性存储在相应的可见性映射表中，可见性映射表在6.2节中描述。

让我们来看看PostgreSQL在执行下面的SELECT命令时如何读取元组。

```sql
testdb=# SELECT id, name FROM tbl WHERE id BETWEEN 18 and 19;
 id |  name   
----+--------
 18 | Queen
 19 | Boston
(2 rows)
```

该查询从表的两列中获取数据：'id'和'name'，索引'tbl_idx'由这些列组成。 因此，当使用索引扫描时，乍一看似乎不需要访问表页面，因为索引元组包含必要的数据。 但事实上，PostgreSQL原则上必须检查元组的可见性，并且索引元组没有关于事务的任何信息，例如5.2节中描述的堆元组的t_xmin和t_xmax。 因此，PostgreSQL必须访问表数据来检查索引元组中数据的可见性。 这有点本末倒置。

为了避免这种困境，PostgreSQL使用目标表的可见性映射。 如果存储在页中的所有元组都可见，则PostgreSQL使用索引元组的键，并且不访问索引元组指向的表页以检查其可见性; 否则，PostgreSQL读取从索引元组指向的表元组并检查元组的可见性，这是普通的过程。

在这个例子中，不需要访问'Tuple_18'，因为存储'Tuple_18'的第0页是可见的，也就是说，包括第0页中的Tuple_18的所有元组都可见。 相反，需要访问'Tuple_19'来处理并发控制，因为第一页的可见性不可见。 参考图7.7。

**图. 7.7. Index-Only Scans 如何执行**

![Fig. 7.7. How Index-Only Scans performs](imgs/ch7/fig-7-07.png)
