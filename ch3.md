## 第三章  查询处理

根据PostgreSQL在[官方文档](https://www.postgresql.org/docs/current/static/features.html)中的描述，它支持2011年SQL标准所需的大量特性。查询处理是PostgreSQL中最复杂的子系统，它可以高效地处理受支持的SQL。 本章概述了查询处理;，重点描述查询优化。

本章包括以下三部分内容：

- **第一部分:** 3.1节.

  本节概述了PostgreSQL中的查询处理。

- **第二部分:** 3.2. — 3.4节.

  本部分介绍获取单表查询最优计划的过程。 在第3.2和3.3节中，分别介绍了估算成本(explain)和创建计划树(plan tree)的过程。 在3.4节中，简要描述了执行器(executor)的操作。

- **第三部分:** 3.5. — 3.6节.

  本部分介绍获取多表查询最优计划的过程。 在3.5节中，描述了三种连接方法：nested loop join、mege join和hash join。 在3.6节中，描述创建多表查询计划树的过程。

PostgreSQL支持两个有趣的技术和实用的特性，即[外部数据包(Foreign Data Wrappers) (FDW)](http://www.postgresql.org/docs/current/static/fdwhandler.html)和[并行查询Parallel Query](https://www.postgresql.org/docs/current/static/parallel-query.html); 他们将在第4章中介绍。

## 3.1. 概述

在PostgreSQL中，虽然9.6版本中实现的并行查询使用多个后台工作进程，但后端进程基本上处理连接客户端发出的所有查询。 该后端由五个子系统组成，如下所示：

1. 编译器(Parser)

   解析器解析纯文本形式的SQL语句生成解析树(parse tree)。

2. 分析器(Analyzer/Analyser)

   分析器对解析树进行语义分析并生成查询树(query tree)。

3. 重写器(Rewriter)

   重写器使用存储在[规则系统](http://www.postgresql.org/docs/current/static/rules.html)中的规则转换查询树。

4. 优化器(Planner)

   优化器从查询树中生成执行最优计划树(plan tree)。

5. 执行器(Executor)

   执行器通过按照计划树创建的顺序访问表和索引来执行查询。

**图. 3.1. 查询处理**

![Fig. 3.1. Query Processing.](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch3/fig-3-01.png?raw=true)

在本节中，提供了这些子系统的概述。 由于优化器和执行器非常复杂，下面将对这些功能进行详细说明。

PostgreSQL的查询处理在[官方文档](http://www.postgresql.org/docs/current/static/overview.html)中有详细描述。

### 3.1.1. 解析器(Parser)

解析器解析纯文本形式的SQL语句生成解析树。 这里有一个具体的例子，后面不做详细的描述。

让我们思考如下所示的查询。 

```sql
testdb=# SELECT id, data FROM tbl_a WHERE id < 300 ORDER BY data;
```

解析树是在 [parsenodes.h](https://github.com/postgres/postgres/blob/master/src/include/nodes/parsenodes.h) 中定义根节点为 [SelectStmt](javascript:void(0)) 结构的树。

图3.2(B)演示了图3.2(A)中所示查询的解析树。 

**图. 3.2. 解析树示例**

![Fig. 3.2. An example of a parse tree.](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch3/fig-3-02.png?raw=true)

SELECT查询的元素和解析树的相应元素的编号相同。 例如，(1)是第一个目标列表中的一项，它是表的'id'列，(4)是WHERE子句，等等。

由于解析器仅在生成解析树时检查输入的语法，因此只有在查询中存在语法错误时才会返回错误。

解析器不检查输入查询的语义。 例如，即使查询包含不存在的表名，解析器也不会返回错误。 语义检查由分析器完成。

### 3.1.2. 分析器(Analyzer)

分析器对解析树进行语义分析并生成查询树。

查询树的根是 [Query](javascript:void(0)) 结构体([parsenodes.h](https://github.com/postgres/postgres/blob/master/src/include/nodes/parsenodes.h)中定义); 此结构包含其相应查询的元数据，例如此命令的类型(SELECT，INSERT或其他)以及多个叶子; 每个叶子形成一个列表或一棵树并保存个别特定子句的数据。

图3.3说明了前一小节中图3.2(a)所示查询的查询树。

**图. 3.3. 查询树示例**

![Fig. 3.3. An example of a query tree.](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch3/fig-3-03.png?raw=true)

上面的查询树简要描述如下。

- targetList是该查询结果列的列表。 在这个例子中，这个列表由两列组成：'id'和'data'。 如果输入查询树使用'*'，则分析器将明确地将其替换为所有列。
- rtable是此查询中使用的关系列表。 在这个例子中，这个表保存了表'tbl_a'的信息，比如这个表的oid和这个表的名字。
- jointree存储了FROM子句和WHERE子句。
- sortClause是SortGroupClause的列表。

[官方文档](http://www.postgresql.org/docs/current/static/querytree.html)中描述了查询树的详细信息。

### 3.1.3. 重写器(Rewriter)

重写是实现规则系统的系统，根据存储在pg_rules系统目录中的规则转换查询树。 规则系统本身就是一个有趣的体系，但是为了防止本章变得太长，规则系统和重写器的描述被忽略了。

PostgreSQL中的[视图](https://www.postgresql.org/docs/current/static/rules-views.html)使用规则系统实现。 当视图由[CREATE VIEW](http://www.postgresql.org/docs/current/static/sql-createview.html) 命令定义时，自动生成相应的规则并将其存储在系统目录中。 

假定已经定义了以下视图，并且相应的规则存储在pg_rules系统目录中。

```sql
sampledb=# CREATE VIEW employees_list 
sampledb-#      AS SELECT e.id, e.name, d.name AS department 
sampledb-#            FROM employees AS e, departments AS d WHERE e.department_id = d.id;
```

当发出包含如下所示的视图查询时，解析器将创建解析树，如图3.4(a)所示。

```sql
sampledb=# SELECT * FROM employees_list;
```

在此阶段，重写器将范围表节点处理为子查询的解析树，子查询是存储在*pg_rules*中的对应视图。 

**图. 3.4. 重写示例**

![Fig. 3.4. An example of the rewriter stage.](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch3/fig-3-04.png?raw=true)

由于PostgreSQL使用这种机制来实现视图，在9.2版本之前视图无法更新。 但是，从9.3版本开始视图可以更新; 尽管如此，更新视图有很多限制。这些细节在[官方文档](https://www.postgresql.org/docs/current/static/sql-createview.html#SQL-CREATEVIEW-UPDATABLE-VIEWS)中作了说明。 

### 3.1.4. 优化器(planner)和执行器(exector)

优化器从重写器接收查询树并生成可由执行器最优处理的(查询)计划树。 

PostgreSQL中的优化器是基于成本的优化；它不支持基于规则的优化和提示。此计划器是RDBMS中最复杂的子系统；因此，本章后面的部分将详细描述优化器。 

 *pg_hint_plan*

PostgreSQL不支持在SQL中规划提示，它将永远不被支持。如果您想在查询中使用提示，那么值得考虑引用*pg_hint_plan*扩展。请详细参考[官方网站](http://pghintplan.osdn.jp/pg_hint_plan.html)。 

与其他RDBMS一样，PostgreSQL中的 [EXPLAIN](http://www.postgresql.org/docs/current/static/sql-explain.html) 命令显示计划树。 示例如下所示。

```sql
testdb=# EXPLAIN SELECT * FROM tbl_a WHERE id < 300 ORDER BY data;                          					QUERY PLAN                           
--------------------------------------------------------------- 
Sort  (cost=182.34..183.09 rows=300 width=8)   
	Sort Key: data   
		->  Seq Scan on tbl_a  (cost=0.00..170.00 rows=300 width=8)         
			Filter: (id < 300)
(4 rows)
```

这个结果显示了图3.5所示的计划树。

**图. 3.5. 计划树示例，计划树与EXPLAIN命令的结果之间的关系**![Fig. 3.5. A simple plan tree and the relationship between the plan tree and the result of the EXPLAIN command.](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch3/fig-3-05.png?raw=true)

计划树由称为 *计划节点(plan nodes)* 的元素组成，并连接到 *PlannedStmt* 结构的 Plantree 列表。 这些元素在 [plannodes.h](https://github.com/postgres/postgres/blob/master/src/include/nodes/plannodes.h) 中定义。 细节将在第3.3.3节(和第3.5.4.2节)中描述。 

每个计划节点都具有执行器处理所需的信息，并且在单表查询的情况下，执行器将从计划树的末尾处理到根节点。 

例如，图3.5所示的计划树是一个排序节点和一个顺序扫描节点的列表; 因此，执行器通过顺序扫描表：tbl_a，然后对获得的结果进行排序。 

执行器通过缓冲区管理器([第八章](ch8.md)中描述)读写数据库集群中的表和索引。处理查询时，执行器使用一些预先分配的内存区(如temp_buffers和work_mem)，并在必要时创建临时文件。

另外，在访问元组时，PostgreSQL使用并发控制机制来维护正在运行的事务的一致性和隔离性。 [第5章](ch5.md)介绍了并发控制机制。

**图. 3.6. 执行器，缓冲区管理者和临时文件之间的关系**

![Fig. 3.6. The relationship among the executor, buffer manager and temporary files.](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch3/fig-3-06.png?raw=true)

 

------

## 3.2. 单表查询中的成本估算

PostgreSQL的查询优化基于成本。 成本是无量纲的价值，而这些并不是绝对的绩效指标，而是用来比较经营相对绩效的指标。

成本通过[costsize.c](https://github.com/postgres/postgres/blob/master/src/backend/optimizer/path/costsize.c) 中定义的函数估算。 执行器执行的所有操作都具有相应的成本估算函数。 例如，顺序扫描和索引扫描的成本分别由cost_seqscan()和cost_index()进行估算。

在PostgreSQL中，有三种成本：**启动(start-up)**，**运行(run)**和**总计(total)**。 总成本是启动和运行成本的总和; 因此，独立估计只有启动和运行成本。

- **启动成本**是在获取第一个元组之前花费的成本。 例如，索引扫描节点的启动成本是读取索引页以访问目标表中的第一个元组的开销。
- **运行成本**是获取所有元组的成本。
- **总成本**是启动成本和运行成本的总和。

[EXPLAIN](https://www.postgresql.org/docs/current/static/sql-explain.html)命令显示每个操作的启动和总成本。 最简单的例子如下所示：

```sql
testdb=# EXPLAIN SELECT * FROM tbl;
					QUERY PLAN                        
--------------------------------------------------------- 
Seq Scan on tbl  (cost=0.00..145.00 rows=10000 width=8)
(1 row)
```

在第4行中，该命令显示有关顺序扫描的信息。 在成本部分，有两个值; 0.00和145.00。 在这种情况下，启动成本和总成本分别为0.00和145.00。

在本节中，我们将详细探讨如何估计顺序扫描(sequential scan)，索引扫描(index scan)和排序操作(sort operation)。

在下面的描述中，我们使用如下所示的表和索引：

```sql
testdb=# CREATE TABLE tbl (id int PRIMARY KEY, data int);
testdb=# CREATE INDEX tbl_data_idx ON tbl (data);
testdb=# INSERT INTO tbl SELECT generate_series(1,10000),generate_series(1,10000);
testdb=# ANALYZE;
testdb=# \d tbl
      Table "public.tbl"
 Column |  Type   | Modifiers 
--------+---------+-----------
 id     | integer | not null
 data   | integer | 
Indexes:
    "tbl_pkey" PRIMARY KEY, btree (id)
    "tbl_data_idx" btree (data)
```

### 3.2.1. 顺序扫描

顺序扫描的成本由cost_seqscan()函数估计。 在本小节中，我们将探讨如何估计以下查询的顺序扫描成本。

```sql
testdb=# SELECT * FROM tbl WHERE id < 8000;
```

在顺序扫描中，启动成本等于0，运行成本由以下公式定义：

​	‘run cost’ = ‘cpu run cost’ + ‘disk run cost’

​			= (cpu_tuple_cost + cpu_operator_cost) × $N_ {tuple}$ + seq_page_cost × $N_ {page}$


其中[seq_page_cost](https://www.postgresql.org/docs/current/static/runtime-config-query.html#GUC-SEQ-PAGE-COST), [cpu_tuple_cost](https://www.postgresql.org/docs/current/static/runtime-config-query.html#GUC-CPU-TUPLE-COST)和[cpu_operator_cost](https://www.postgresql.org/docs/current/static/runtime-config-query.html#GUC-CPU-OPERATOR-COST)在postgresql.conf文件中设置，默认值分别为1.0, 0.01和0.0025; $N_ {tuple}$和N$N_ {page}$分别是该表的所有元组和所有页面的编号，可以使用以下查询显示这些编号：

```sql
testdb=# SELECT relpages, reltuples FROM pg_class WHERE relname = 'tbl';
 relpages | reltuples 
----------+-----------
       45 |     10000
(1 row)
```


​	(1) $N_ {page}$=10000,
	(2) $N_ {page}$=45.		

从而,

​	‘run cost’  =  (0.01 + 0.0025) × 10000 + 1.0 × 45 = 170.0.

最终,

​	‘total cost’ = 0.0 + 170.0 = 170.

为了确认，以上查询的EXPLAIN命令的结果如下所示：


```sql
testdb=# EXPLAIN SELECT * FROM tbl WHERE id < 8000;
					QUERY PLAN                       
-------------------------------------------------------- 
Seq Scan on tbl  (cost=0.00..170.00 rows=8000 width=8)   
	Filter: (id < 8000)
(2 rows)
```

在第4行，我们可以发现启动和总成本分别是170.00和170.00， 据估计，将通过扫描所有行来选择8000行(tuple)。  在第5行中，显示了顺序扫描的过滤器‘Filter：(ID<8000)’。 更准确地说，它被称为*table level filter predicate*。 请注意，这种类型的过滤器是在读取表中的所有元组时使用的，并且它不会缩小page页的扫描范围。 

从运行成本估算中可以看出，PostgreSQL假设所有页面都将从存储中读取; 也就是说，PostgreSQL不会考虑扫描页是否在共享缓冲区中。

### 3.2.2. 索引扫描

尽管PostgreSQL支持许多[索引方法](https://www.postgresql.org/docs/current/static/indexes-types.html)，如BTree，[GiST](https://www.postgresql.org/docs/current/static/gist.html)，[GIN](https://www.postgresql.org/docs/current/static/gin.html)和[BRIN](https://www.postgresql.org/docs/current/static/brin.html)，但索引扫描的成本使用通用的成本函数cost_index()估算。

在本小节中，我们将探讨如何估计以下查询的索引扫描成本：

```sql
testdb=# SELECT id, data FROM tbl WHERE data < 240;
```

在估算成本之前，索引页和索引元组的数量, $N_{index,tuple}$ and $N_{index,page}$ 如下所示：

```sql
testdb=# SELECT relpages, reltuples FROM pg_class WHERE relname = 'tbl_data_idx';
 relpages | reltuples 
----------+-----------
       30 |     10000
(1 row)
```

​	(3) $N_{index,tuple}$=10000,
	(4) $N_{index,page}$=30.

#### 3.2.2.1. 启动成本(Start-Up Cost)

索引扫描的启动成本是读取索引页以访问目标表中第一个元组的开销，它由以下公式定义：

​	‘start-up cost’ = {ceil($log_{2}$($Nindex,tuple$)) + ($Hindex$ + 1) × 50} × cpu_operator_cost,

$H_{index}$是索引树的高度。

在这种情况下，根据(3)，$N_{index,tuple}$为10000; $H_{index}$是1; cpu_operator_cost为0.0025(默认)。 从而，

​	(5) ‘start-up cost’ = {ceil($log_{2}($10000)) + (1 + 1) × 50} × 0.0025 = 0.285

#### 3.2.2.2. 运行成本(Run Cost)

索引扫描的运行成本是表和索引的CPU成本和IO(输入/输出)成本的总和：

​	‘run cost’ = (‘index cpu cost’ + ‘table cpu cost’) + (‘index IO cost’ + ‘table IO cost’).

如果可以使用 [Index-Only Scans](https://www.postgresql.org/docs/10/static/indexes-index-only-scans.html)(第7章中描述)，则不估计'‘table cpu cost’和'table IO cost’。

前三项成本(i.e. index cpu cost, table cpu cost and index IO cost) 如下所示：

​	‘index cpu cost’ = Selectivity × $N_{index,tuple}$ × (cpu_index_tuple_cost + qual_op_cost),

​	‘table cpu cost’ = Selectivity × $N_{tuple}$ × cpu_tuple_cost,

​	‘index IO cost’ = ceil(Selectivity × $N_{index,page}$) × random_page_cost,

其中[cpu_index_tuple_cost](https://www.postgresql.org/docs/current/static/runtime-config-query.html#GUC-CPU-INDEX-TUPLE-COST)和[random_page_cost](https://www.postgresql.org/docs/current/static/runtime-config-query.html#GUC-RANDOM-PAGE-COST)在postgresql.conf文件中设置(默认值分别为0.005和4.0); 大致来说，qual_op_cost是索引的评估成本，这里没有给出太多解释：qual_op_cost=0.0025。Selectivity是指定WHERE子句的索引搜索范围的比例; 它是从0到1的浮点数，下面详细介绍。 例如，(Selectivity×$N_{tuple}$)表示要读取的表元组的数量，(Selectivity×$N_{index,page}$)表示要读取的索引页的数量等等。

*Selectivity*

查询谓词的selectivity是使用histogram_bounds或MCV(Most Common Value)估计的，它们都存储在统计信息[pg_stats](https://www.postgresql.org/docs/current/static/view-pg-stats.html)中。 这里，使用具体实例简要描述selectivity的计算。 更多细节见[官方文档](https://www.postgresql.org/docs/10/static/row-estimation-examples.html)。 

表中每列的MCV作为一对most_common_vals和most_common_freqs列存储在[pg_stats](https://www.postgresql.org/docs/10/static/view-pg-stats.html)视图中。

- most_common_vals是该列的MCV列表。
- most_common_freqs是MCV频率的列表。

一个简单的例子如下所示。["countries"](javascript:void(0)) 表有两列：存储国家名称的列‘country’和存储国家所属的大陆名称的列‘continent’。

```sql
testdb=# \d countries
   Table "public.countries"
  Column   | Type | Modifiers 
-----------+------+-----------
 country   | text | 
 continent | text | 
Indexes:
    "continent_idx" btree (continent)

testdb=# SELECT continent, count(*) AS "number of countries", 
testdb-#     (count(*)/(SELECT count(*) FROM countries)::real) AS "number of countries / all countries"
testdb-#       FROM countries GROUP BY continent ORDER BY "number of countries" DESC;
   continent   | number of countries | number of countries / all countries 
---------------+---------------------+-------------------------------------
 Africa        |                  53 |                   0.274611398963731
 Europe        |                  47 |                   0.243523316062176
 Asia          |                  44 |                   0.227979274611399
 North America |                  23 |                   0.119170984455959
 Oceania       |                  14 |                  0.0725388601036269
 South America |                  12 |                  0.0621761658031088
(6 rows)
```

让我们考虑一下WHERE子句'continent ='Asia''的查询：

```sql
testdb=# SELECT * FROM countries WHERE continent = 'Asia';
```

在这种情况下，优化器使用‘continent’列的MCV估算索引扫描成本。 下面显示了此列的most_common_vals和most_common_freqs：

```sql
testdb=# \x
Expanded display is on.
testdb=# SELECT most_common_vals, most_common_freqs FROM pg_stats 
testdb-#                  WHERE tablename = 'countries' AND attname='continent';
-[ RECORD 1 ]-----+-------------------------------------------------------------
most_common_vals  | {Africa,Europe,Asia,"North America",Oceania,"South America"}
most_common_freqs | {0.274611,0.243523,0.227979,0.119171,0.0725389,0.0621762}
```

most_common_vals的'Asia'对应的most_common_freqs的值为0.227979。 因此，在此估计中使用0.227979作为selectivity。

如果不能使用MCV，则使用目标列的histogram_bounds的值来估计成本。

- **histogram_bounds** 是将列值分成数量大致相等的值的列表。

一个具体的例子。 这是表'tbl'中列'data'的histogram_bounds的值：

```sql
testdb=# SELECT histogram_bounds FROM pg_stats WHERE tablename = 'tbl' AND attname = 'data';
        			     	      histogram_bounds
---------------------------------------------------------------------------------------------------
 {1,100,200,300,400,500,600,700,800,900,1000,1100,1200,1300,1400,1500,1600,1700,1800,1900,2000,2100,
2200,2300,2400,2500,2600,2700,2800,2900,3000,3100,3200,3300,3400,3500,3600,3700,3800,3900,4000,4100,
4200,4300,4400,4500,4600,4700,4800,4900,5000,5100,5200,5300,5400,5500,5600,5700,5800,5900,6000,6100,
6200,6300,6400,6500,6600,6700,6800,6900,7000,7100,7200,7300,7400,7500,7600,7700,7800,7900,8000,8100,
8200,8300,8400,8500,8600,8700,8800,8900,9000,9100,9200,9300,9400,9500,9600,9700,9800,9900,10000}
(1 row)
```

默认情况下，histogram_bounds被分成100个桶。 图3.7展示了这个例子中的桶和相应的histogram_bounds。 存储桶从0开始编号，每个存储桶(大约)存储相同数量的元组。 histogram_bounds的值是相应桶的边界。 例如，histogram_bounds的第0个值是1，这意味着它是存储在bucket_0中的元组的最小值; 第一个值是100，这是存储在bucket_1中的元组的最小值，依此类推。

**图. 3.7. Buckets and histogram_bounds.**

![Fig. 3.7. Buckets and histogram_bounds.](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch3/fig-3-07.png?raw=true)

接下来，将使用本小节中的示例计算selectivity。 查询有一个WHERE子句'data <240'，值'240'在第二个桶中。 在这种情况下，selectivity可以通过linear interpolation得出; 因此，使用以下等式来计算该查询中列'数据'的selectivity：

​	(6) Selectivity = $\frac{2+(240−hb[2])/(hb[3]−hb[2])}{100}$ = $\frac{2+(240−200)/(300−200)}{100}$ = $\frac{2+40/100}{100}$ = 0.024.

 因此，根据 (1),(3),(4) and (6),

​	(7) ‘index cpu cost’ = 0.024 × 10000 × (0.005 + 0.0025) = 1.8,

​	(8) ‘table cpu cost’ = 0.024 × 10000 × 0.01 = 2.4,

​	(9) ‘index IO cost’ = ceil(0.024 × 30) × 4.0 = 4.0.

‘table IO cost’ 由以下等式定义：

​	‘table IO cost’ = max_IO_cost + $indexCorrelation^{2}$ × (min_IO_cost − max_IO_cost).

max_IO_cost是IO成本最差的情况，即随机扫描所有表页的成本; 该成本由以下等式定义：

​	max_IO_cost = $N_{page}$ × random_page_cost.

这种情况下，根据 (2), $N_{page}$=45, 因此

​	(10) max_IO_cost = 45 × 4.0 = 180.0.

min_IO_cost是IO成本的最佳情况，即顺序扫描所选表页的成本; 该成本由以下等式定义：

​	min_IO_cost = 1 × random_page_cost + (ceil(Selectivity × $N_{page}$) − 1) × seq_page_cost.

这种情况下，

​	(11) min_IO_cost = 1 × 4.0 + (ceil(0.024 × 45)) − 1) × 1.0 = 5.0.

indexCorrelation在下面详细描述，在这个例子中，

​	(12) indexCorrelation = 1.0.

因此，根据(10),(11) 和 (12),

​	(13) ‘table IO cost’ = 180.0 + $1.0^{2}$ × (5.0 − 180.0) = 5.0.

最后，根据 (7),(8),(9) 和 (13),

​	(14) ‘run cost’ = (1.8 + 2.4) + (4.0 + 5.0) = 13.2.

索引相关性(Index Correlation)是列值的物理行排序与逻辑排序之间的统计相关性(引自官方文档)。 这范围从$-1$到 $+1$。 为了理解索引扫描和索引相关性之间的关系，下面给出一个具体示例。

表tbl_corr有五列：两列是文本类型，三列是整数类型。 三个整数列存储从1到12的数字。物理上，tbl_corr由三个page页组成，每个页有四个元组。 每个整数类型列都有一个名称为index_col_asc的索引，依此类推。

```sql
testdb=# \d tbl_corr
    Table "public.tbl_corr"
  Column  |  Type   | Modifiers 
----------+---------+-----------
 col      | text    | 
 col_asc  | integer | 
 col_desc | integer | 
 col_rand | integer | 
 data     | text    |
Indexes:
    "tbl_corr_asc_idx" btree (col_asc)
    "tbl_corr_desc_idx" btree (col_desc)
    "tbl_corr_rand_idx" btree (col_rand)
```

```sql
testdb=# SELECT col,col_asc,col_desc,col_rand 
testdb-#                         FROM tbl_corr;
   col    | col_asc | col_desc | col_rand 
----------+---------+----------+----------
 Tuple_1  |       1 |       12 |        3
 Tuple_2  |       2 |       11 |        8
 Tuple_3  |       3 |       10 |        5
 Tuple_4  |       4 |        9 |        9
 Tuple_5  |       5 |        8 |        7
 Tuple_6  |       6 |        7 |        2
 Tuple_7  |       7 |        6 |       10
 Tuple_8  |       8 |        5 |       11
 Tuple_9  |       9 |        4 |        4
 Tuple_10 |      10 |        3 |        1
 Tuple_11 |      11 |        2 |       12
 Tuple_12 |      12 |        1 |        6
(12 rows)
```

这些列的索引相关性如下所示：

```sql
testdb=# SELECT tablename,attname, correlation FROM pg_stats WHERE tablename = 'tbl_corr';
 tablename | attname  | correlation 
-----------+----------+-------------
 tbl_corr  | col_asc  |           1
 tbl_corr  | col_desc |          -1
 tbl_corr  | col_rand |    0.125874
(3 rows)
```

当执行以下查询时，PostgreSQL只读取第一页，因为所有目标元组都存储在第一页中。 参考图3.8(a)。

```sql
testdb=# SELECT * FROM tbl_corr WHERE col_asc BETWEEN 2 AND 4;
```

另一方面，当执行以下查询时，PostgreSQL必须读取所有页。 参考图3.8(b).

```sql
testdb=# SELECT * FROM tbl_corr WHERE col_rand BETWEEN 2 AND 4;
```

这样，索引相关性是一种统计相关性，它反映了在估计索引扫描成本时表中索引排序和表中物理元组排序之间的扭曲所引起的随机访问的影响。

**图. 3.8. 索引相关性**

![Fig. 3.8. Index correlation.](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch3/fig-3-08.png?raw=true)

 

#### 3.2.2.3. 总成本(Total Cost)

根据 (3) 和 (14),

​	(15) ‘total cost’ = 0.285 + 13.2 = 13.485.

为了确认，上面SELECT查询的EXPLAIN命令的结果如下所示： 

```sql
testdb=# EXPLAIN SELECT id, data FROM tbl WHERE data < 240;
							QUERY PLAN                                 	
---------------------------------------------------------------------------
Index Scan using tbl_data_idx on tbl  (cost=0.29..13.49 rows=240 width=8)   
	Index Cond: (data < 240)
(2 rows)
```

在第4行中，我们可以发现启动成本和总成本分别为0.29和13.49，并且估计将会扫描240行。

在第5行中，显示了索引扫描的索引条件‘Index Cond:(data < 240)’。 更确切地说，这个条件被称为*访问谓词access predicate*，它表达了索引扫描的开始和结束条件。

根据这篇[文章](http://use-the-index-luke.com/sql/explain-plan/postgresql/filter-predicates)，PostgreSQL中的EXPLAIN命令不区分访问谓词和索引过滤器谓词。 因此，如果您分析EXPLAIN的输出，不仅要注意索引条件，还要注意行的估计值。

*seq_page_cost and random_page_cost*

[seq_page_cost](https://www.postgresql.org/docs/10/static/runtime-config-query.html#GUC-SEQ-PAGE-COST)和[random_page_cost](https://www.postgresql.org/docs/10/static/runtime-config-query.html#GUC-RANDOM-PAGE-COST) 的默认值分别是1.0和4.0。 这意味着PostgeSQL假定随机扫描比顺序扫描慢四倍; 很明显，PostgreSQL的默认值是基于使用硬盘HDD。

另一方面，近几天，由于主要使用SSD，因此random_page_cost的默认值太大。 如果尽管使用了SSD，仍然使用了默认值random_page_cost，但优化器可能会选择无效的计划。 因此，使用SSD时，最好将random_page_cost的值更改为1.0。

此[博客](https://amplitude.engineering/how-a-single-postgresql-config-change-improved-slow-query-performance-by-50x-85593b8991b0)在使用random_page_cost的默认值时报告了此问题。

### 3.2.3. 排序(Sort)

排序路径用于排序操作，如ORDER BY，merge join操作的预处理和其他函数。排序的成本是使用cost_sort()函数估算的。 

在排序操作中，如果所有要排序的元组都可以存储在work_mem中，则使用quicksort算法。 否则，将创建一个临时文件并使用文件merge-sort算法。

排序路径的启动成本是排序目标元组的成本，因此，成本为O($N_{sort}$×$log_{2}$⁡($N_{sort}$))，其中$N_{sort}$是进行排序的元组数。 排序路径的运行成本是读取排序元组的成本，因此，成本为O($N_{sort}$)。

在本小节中，我们将探讨如何估计以下查询的排序成本。 假设这个查询将在work_mem中排序，而不使用临时文件。

```sql
testdb=# SELECT id, data FROM tbl WHERE data < 240 ORDER BY id;
```

在这种情况下，启动成本在以下公式中定义：

​	‘start-up cost’ = C + comparison_cost × $N_{sort}$ × $log_{2}$($N_{sort}$),

其中$C$是最后一次扫描的总成本，即索引扫描的总成本; 根据(15)，是13.48513.485;$N_{sort}$=240; compare_costcomparison_cost在2×cpu_operator_cost中定义。 从而，

​	‘start-up cost’ = 13.485+(2 × 0.0025) × 240.0 × $log_{2}$(240.0) = 22.973.

运行成本是读取内存中已排序的元组的成本; 从而，

​	‘run cost’ = cpu_operator_cost × $N_{sort}$ = 0.0025 × 240 = 0.6.

最终,

​	‘total cost’ = 22.973 + 0.6 = 23.573.

为了确认，上面的SELECT查询的EXPLAIN命令的结果如下所示：

```sql
testdb=# EXPLAIN SELECT id, data FROM tbl WHERE data < 240 ORDER BY id;                                   					QUERY PLAN                                    
--------------------------------------------------------------------------------- 
Sort  (cost=22.97..23.57 rows=240 width=8)   
	Sort Key: id   
		->  Index Scan using tbl_data_idx on tbl  (cost=0.29..13.49 rows=240 width=8)         
			Index Cond: (data < 240)
(4 rows)
```

在第4行中，我们可以发现启动和总成本分别为22.97和23.57。

## 3.3. 创建单表查询的计划树

由于的处理非常复杂，因此本节介绍了最简单的过程，即如何创建单表查询的计划树。 3.6节描述了更复杂的处理过程，即如何创建多表查询的计划树。

PostgreSQL中的优化器执行三个步骤，如下所示：

1. 进行预处理。
2. 通过估计所有可能的访问路径的成本，获得最优访问路径。
3. 从最优路径创建计划树。

访问路径是估计成本的处理单位; 例如，顺序扫描，索引扫描，排序和各种连接操作都有相应的路径。 访问路径仅在优化器内部用于创建计划树。 访问路径的最基本的数据结构是[relation.h](https://github.com/postgres/postgres/blob/master/src/include/nodes/relation.h)中定义的 [Path](javascript:void(0)) 结构体，它对应于顺序扫描。 所有其他访问路径都基于该结构。 详细内容将在后面介绍。

为了处理上述步骤，优化器在内部创建 [PlannerInfo](javascript:void(0)) 结构，并保存查询树、关于查询中包含的关系的信息、访问路径等。

在本节中，将使用特定示例描述如何从查询树创建计划树。

### 3.3.1. 预处理(Preprocessing)

在创建计划树之前，优化器对存储在PlannerInfo结构中的查询树进行一些预处理。

虽然预处理涉及很多步骤，但我们只讨论本小节中单表查询的主要预处理。 其他预处理操作在3.6节中描述。

1. 简化目标列表、限制子句等。 

   例如，'2 + 2'由[clauses.c](https://github.com/postgres/postgres/blob/master/src/backend/optimizer/util/clauses.c)中定义的eval_const_expressions()函数重写为'4'。

2. 规范布尔表达式。 

   例如， ‘NOT (NOT a)’ 被重写为'a'。

3. 展开AND/OR表达式.

   SQL标准中的AND和OR是二元运算符，但是在PostgreSQL内部，它们是n元运算符，优化器总是假定所有嵌套的AND和OR表达式都被展开。

   给出一个具体的例子。 考虑布尔表达式‘(id = 1) OR (id = 2) OR (id = 3)’ 。 图3.9(a)显示了使用二元运算符时查询树的一部分。 优化器通过使用三元运算符进行拼合来简化此树。 参考图3.9(b)。

**图. 3.9. 展开AND/OR表达式示例**

![Fig. 3.9. An example of flattening AND/OR expressions.](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch3/fig-3-09.png?raw=true)

### 3.3.2. 获取最优访问路径

为了获得最优的访问路径，优化器估计所有可能的访问路径的成本，并选择最优的访问路径。 更具体地说，优化器执行以下操作：

1. 创建 [RelOptInfo](javascript:void(0)) 结构来存储访问路径和相应的成本。

   RelOptInfo结构由make_one_rel()函数创建并存储在PlannerInfo结构的simple_rel_array中。 参考图3.10。 在初始状态下，如果存在相关索引，则RelOptInfo保存*baserestrictinfo* 和 *indexlist* ; *baserestrictinfo* 存储查询的WHERE子句，*indexlist*存储目标表的相关索引。

2. 估计所有可能的访问路径的成本，并将访问路径添加到RelOptInfo结构中。

   这个处理的细节如下：

   1. 创建路径，估计顺序扫描的成本，并将估计的成本写入路径中。 然后，路径被添加到RelOptInfo结构的*pathlist*中。
   2. 如果存在与目标表相关的索引，则创建索引访问路径，估计所有索引扫描成本，并将估计的成本写入路径。 然后，索引路径被添加到*pathlist*。
   3. 如果可以完成[位图扫描(bitmap scan)](https://wiki.postgresql.org/wiki/Bitmap_Indexes)，则创建位图扫描路径，估计所有位图扫描成本，并将估计的成本写入路径中。 然后，位图扫描路径被添加到*pathlist*。

3. 在RelOptInfo结构的*pathlist*中获取最优的访问路径。

4. 根据需要估算 LIMIT, ORDER BY 和 ARREGISFDD成本。

为了理解优化器的表现如何，下面给出了两个具体的例子。

#### 3.3.2.1. 示例1

首先，我们探索一个没有索引的简单单表查询; 这个查询包含WHERE和ORDER BY子句。

```sql
testdb=# \d tbl_1
     Table "public.tbl_1"
 Column |  Type   | Modifiers 
--------+---------+-----------
 id     | integer | 
 data   | integer | 

testdb=# SELECT * FROM tbl_1 WHERE id < 300 ORDER BY data;
```

图3.10和3.11描述了规划器在这个例子中的表现。

**图. 3.10. 如何获得示例1的最优路径**

![Fig. 3.10. How to get the cheapest path of Example 1.](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch3/fig-3-10.png?raw=true)

1. 创建一个RelOptInfo结构并将其存储在PlannerInfo的simple_rel_array中。

2. 在RelOptInfo的*baserestrictinfo*中添加一个WHERE子句。

   通过[initsplan.c](https://github.com/postgres/postgres/blob/master/src/backend/optimizer/plan/initsplan.c)中定义的distribute_restrictinfo_to_rels()函数将WHERE子句*‘id < 300’* 添加到*baserestrictinfo*。 另外，RelOptInfo的*indexlist*是NULL，因为目标表没有相关的索引。

3. 通过[planner.c](https://github.com/postgres/postgres/blob/master/src/backend/optimizer/plan/planner.c)中定义的standard_qp_callback() 函数将排序pathkey添加到PlannerInfo的*sort_pathkeys*中。

   *pathkey*是表示路径排序顺序的数据结构。 在这个例子中，列“data”作为一个*pathkey*被添加到sort_pathkeys，因为这个查询包含一个ORDER BY子句并且它的列是'data'。

4. 创建一个路径结构并使用cost_seqscan函数估计顺序扫描的成本，并将估计的成本写入路径。 然后，通过[pathnode.c](https://github.com/postgres/postgres/blob/master/src/backend/optimizer/util/pathnode.c)中定义的add_path() 函数将路径添加到RelOptInfo。

   如前所述， [Path](javascript:void(0)) 结构既包含由cost_seqscan函数估计的启动成本和总成本，等等。

在这个例子中，由于没有目标表的索引，优化器只估计顺序扫描成本; 因此，最优的访问路径是自动确定的。

**图. 3.11. 如何获得示例1的最优路径(从图3.10继续)**

![Fig. 3.11. How to get the cheapest path of Example 1 (continued from Fig. 3.10).](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch3/fig-3-11.png?raw=true)

5. 创建一个新的RelOptInfo结构来处理ORDER BY过程。

   请注意，新的RelOptInfo没有*baserestrictinfo*，即WHERE子句的信息。

6. 创建一个排序路径并将其添加到新的RelOptInfo; 然后将顺序扫描路径链接到排序路径的子路径。

   [SortPath](javascript:void(0)) 结构由两个路径结构组成：路径和子路径; 该路径存储有关排序操作本身的信息，并且子路径存储最优的路径。

   请注意，顺序扫描路径的项目‘parent’保存到旧RelOptInfo的链接，该旧RelOptInfo将WHERE子句存储在其*baserestrictinfo*中。 因此，在下一阶段，即创建计划树时，优化器可以创建包含WHERE子句作为 ‘Filter’的顺序扫描节点，即使新的RelOptInfo没有*baserestrictinfo*。

根据此处获得的最优的访问路径，生成一个查询树。 细节在3.3.3节中描述。

#### 3.3.2.2. 示例2

接下来，我们探索另一个带有两个索引的单表查询; 这个查询包含一个WHERE子句。

```sql
testdb=# \d tbl_2
     Table "public.tbl_2"
 Column |  Type   | Modifiers 
--------+---------+-----------
 id     | integer | not null
 data   | integer | 
Indexes:
    "tbl_2_pkey" PRIMARY KEY, btree (id)
    "tbl_2_data_idx" btree (data)

testdb=# SELECT * FROM tbl_2 WHERE id < 240;
```

图3.12至3.14描述了规划师在这个例子中的表现。

1. 创建一个*RelOptInfo*结构

2. 将WHERE子句添加到*baserestrictinfo*，并将目标表的索引添加到*indexlist*。

   在这个例子中，WHERE子句'id <240'被添加到*baserestrictinfo*，并且两个索引*tbl_2_pkey*和*tbl_2_data_idx*被添加到RelOptInfo的*indexlist*中。

3. 创建一个路径，估计顺序扫描的开销并将路径添加到*RelOptInfo*的*indexlist*中。

**图. 3.12. 示例2中如何获得最优路径**

![Fig. 3.12. How to get the cheapest path of Example 2.](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch3/fig-3-12.png?raw=true)

**图. 3.13. 示例2中如何获得最优路径(从图3.12继续)**

![Fig. 3.13. How to get the cheapest path of Example 2 (continued from Fig. 3.12).](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch3/fig-3-13.png?raw=true)

4. 创建 [IndexPath](javascript:void(0))，估算索引扫描的成本，并使用add_path()函数将IndexPath添加到RelOptInfo的pathlist中。

   在本例中，由于有两个索引tbl_2_pkey和tbl_2_data_idx，所以这些索引按顺序处理。 首先处理tbl_2_pkey。

   为tbl_2_pkey创建IndexPath，并估算启动成本和总成本。 在本例中，tbl_2_pkey是与'id'列相关的索引，而WHERE子句包含'id'列; 因此，WHERE子句存储在IndexPath的 *indexclauses*中。

   请注意，向pathlist添加访问路径时，add_path()函数会按照总成本的排序顺序添加路径。 在此示例中，此索引扫描的总成本小于顺序扫描总成本; 因此，该索引路径被插入在顺序扫描路径之前。

5. 创建另一个IndexPath，估算其他索引扫描的成本并将索引路径添加到RelOptInfo的pathlist。

   接下来，为tbl_2_data_idx创建IndexPath，估计成本并将此IndexPath添加到pathlist。 在这个例子中，没有与tbl_2_data_idx索引相关的WHERE子句; 因此，索引子句是NULL。

请注意，add_path()函数并不总是添加路径。 由于这种操作的复杂性，细节被省略。 有关详细信息，请参阅add_path()函数的注释。

6. 创建一个新的RelOptInfo结构

7. 将最优路径添加到新RelOptInfo的pathlist中

   在这个例子中，最优路径是使用索引tbl_2_pkey的索引路径; 因此，它的路径被添加到新的RelOptInfo的pathlist中。


**图. 3.14. 示例2中如何获取最优路径(从图3.13继续)**

![Fig. 3.14. How to get the cheapest path of Example 2 (continued from Fig. 3.13).](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch3/fig-3-14.png?raw=true)

### 3.3.3. 创建一个计划树

在最后阶段，优化器从最优路径生成计划树。

计划树的根是[plannodes.h](https://github.com/postgres/postgres/blob/master/src/include/nodes/plannodes.h)中定义的  [PlannedStmt](javascript:void(0)) 结构。 它包含19个元素，这里有4个代表性的元素。

- **commandType** 存储一个操作类型，如SELECT，UPDATE和INSERT。
- **rtable** 存储rangeTable条目。
- **relationOids** 存储该查询的相关表的oid。
- **plantree** 存储由计划节点组成的计划树，其中每个节点对应于特定操作，如顺序扫描、排序和索引扫描。 

如上所述，计划树由各种计划节点组成。 [PlanNode](javascript:void(0)) 结构是基本节点，其他节点始终包含它。 例如，用于顺序扫描的[SeqScanNode](javascript:void(0)) 由PlanNode和一个整数变量'scanrelid'组成。 PlanNode包含十四个元素。 以下是七个有代表性的元素。

- **start-up cost** and **total_cost** 是与该节点相对应的操作的估计成本。
- **rows** is the number of rows to be scanned which is estimated by the planner.
- **targetlist** 是优化器估计的要扫描的行数。
- **qual** 是一个存储qual条件的列表。
- **lefttree** and **righttree** 是添加子节点的节点。

在下文中，将描述两个计划树，它们将从前一小节的示例中所示的最优路径生成。

#### 3.3.3.1. 示例1

第一个例子是第3.3.2.1节中示例的计划树。 图3.11所示的最优路径是由排序路径和顺序扫描路径组成的树; 根路径是排序路径，子路径是顺序扫描路径。 尽管省略了详细的解释，但很容易理解，计划树几乎可以从最优路径生成。 在本例中， [SortNode](javascript:void(0)) 被添加到PlannedStmt结构的plantree，SeqScanNode被添加到SortNode的lefttree。 参考图3.15(a)。

**图. 3.15. 计划树示例**

![Fig. 3.15. Examples of plan trees.](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch3/fig-3-15.png?raw=true)

在SortNode中，lefttree指向SeqScanNode。 

在SeqScanNode中，qual保留WHERE子句‘id<300’。 

#### 3.3.3.2. 示例2

第二个例子是第3.3.2.2节中示例的计划树。 图3.14所示的最优路径是索引扫描路径; 因此，该计划树仅由一个 [IndexScanNode](javascript:void(0)) 结构组成。 参考图3.15(b)。

在这个例子中，WHERE子句'id <240'是一个访问谓词; 它因此存储在IndexScanNode的indexqual中。

## 3.4. 执行器如何执行

在单表查询中，执行器按照从计划树末尾到根的顺序取得计划节点，然后调用执行相应节点的处理函数。

每个计划节点都具有用于执行相应操作的函数，并且它们位于[src/backend/executor/](https://github.com/postgres/postgres/blob/master/src/backend/executor/) 目录中。 例如，执行顺序扫描(ScanScan)的函数在[nodeSeqscan.c](https://github.com/postgres/postgres/blob/master/src/backend/executor/nodeIndexscan.c)中定义; 执行索引扫描(IndexScanNode)的函数在[nodeIndexscan.c](https://github.com/postgres/postgres/blob/master/src/backend/executor/nodeIndexscan.c)中定义; 排序SortNode的函数在[nodeSort.c](https://github.com/postgres/postgres/blob/master/src/backend/executor/nodeSort.c)中定义等。

当然，了解执行器执行方式的最好方法是查看EXPLAIN命令的输出，因为PostgreSQL的EXPLAIN几乎按照原样展示了计划树。 这将在第3.3.3节中使用示例1进行解释。

```sql
testdb=# EXPLAIN SELECT * FROM tbl_1 WHERE id < 300 ORDER BY data;
					QUERY PLAN                           
--------------------------------------------------------------- 
Sort  (cost=182.34..183.09 rows=300 width=8)   
	Sort Key: data   
	->  Seq Scan on tbl_1  (cost=0.00..170.00 rows=300 width=8)         
		Filter: (id < 300)
(4 rows)
```

让我们来探讨执行器是如何执行的。从底行到顶行查看EXPLAIN命令的输出结果。 

- **第6行**: 首先，执行器使用[nodeSeqscan.c](https://github.com/postgres/postgres/blob/master/src/backend/executor/nodeSeqscan.c)中定义的函数执行顺序扫描操作。
- **第4行**: 接下来，执行器使用[nodeSort.c](https://github.com/postgres/postgres/blob/master/src/backend/executor/nodeSort.c)中定义的函数对顺序扫描的结果进行排序。



尽管执行器使用存储器分配的work_men和temp_buffers进行查询处理，但如果仅在内存中执行处理，它将使用临时文件。

使用ANALYZE选项，EXPLAIN命令实际上执行查询并显示实际的行数、运行时间和内存使用情况。 具体示例如下所示：

```sql
testdb=# EXPLAIN ANALYZE SELECT id, data FROM tbl_25m ORDER BY id;
						QUERY PLAN                                                        
-------------------------------------------------------------------------------------------------------------------------- 
Sort  (cost=3944070.01..3945895.01 rows=730000 width=4104) (actual time=885.648..1033.746 rows=730000 loops=1)   
	Sort Key: id   
	Sort Method: external sort  Disk: 10000kB   
	->  Seq Scan on tbl_25m  (cost=0.00..10531.00 rows=730000 width=4104) (actual time=0.024..102.548 rows=730000 loops=1) 
Planning time: 1.548 ms 
Execution time: 1109.571 ms
(6 rows)
```

在第6行中，EXPLAIN命令显示执行程序使用了一个大小为10000kB的临时文件。

临时文件暂时在base/pg_tmp子目录中创建，命名方法如下所示。

```shell
{"pgsql_tmp"} + {PID of the postgres process which creates the file} . {sequencial number from 0}
```

例如，临时文件‘pgsql_tmp8903.5’”是由PID为8903的Postgres进程创建的第6个临时文件。 

```shell
$ ls -la /usr/local/pgsql/data/base/pgsql_tmp*
-rw-------  1 postgres  postgres  10240000 12  4 14:18 pgsql_tmp8903.5
```

 

------

## 3.5. Join操作

PostgreSQL支持三种join操作：nested loop join，merge join和hash join。 PostgreSQL中的nested loop join和merge join有几个变化。

在下文中，我们假设读者熟悉这三种连接的基本行为。 如果您不熟悉这些术语，请参阅[[1](http://www.interdb.jp/pg/pgsql03.html#_3.ref.1), [2](http://www.interdb.jp/pg/pgsql03.html#_3.ref.2)]。但是，对于PostgreSQL支持的hybrid hash join with skew，没有太多的解释，在这里将进行更详细的解释。

请注意，PostgreSQL支持的三种连接方法可以执行所有的连接操作，不仅包括INNER JOIN，还包括LEFT/RIGHT OUTER JOIN，FULL OUTER JOIN等; 然而，为了简化，我们将重点放在本章的NATURAL INNER JOIN上。

### 3.5.1. Nested Loop Join

nested loop join是最基本的连接操作，它可以用于任何连接条件。 PostgreSQL支持nested loop join和它的五种变体。

#### 3.5.1.1. Nested Loop Join

nested loop join不需要任何启动操作; 从而，

​	‘start-up cost’=0.

nested loop join的运行成本与外表和内表的大小的乘积成比例; 即 ‘run cost’‘run cost’是O($N_{outer}$×$N_{inner}$)，其中$N_{outer}$ 和  $N_{inner}$分别是外表和内表的元组数。 更确切地说，它由以下等式定义：

​	‘run cost’ = (cpu_operator_cost + cpu_tuple_cost) × $N_{outer}$ × $N_{inner}$ + $C_{inner}$ × $N_{outer}$ + $C_{outer}$

其中$C_{outer}$和$C_{inner}$分别是外表和内表的扫描成本。

**图. 3.16. Nested loop join.**

![Fig. 3.16. Nested loop join.](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch3/fig-3-16.png?raw=true)

nested loop join的成本总是可以估计的，但这种连接操作很少使用，因为通常使用下面描述的更有效的变体。

#### 3.5.1.2. Materialized Nested Loop Join

上面描述的nested loop join必须在读取外表的每个元组时扫描内表的所有元组。因为扫描每个外表元组的整个内表是一个昂贵的过程，PostgreSQL支持*materialized nested loop join*，以减少内表的总扫描成本。 

在运行nested loop join之前，执行器通过使用下面描述的临时元组存储(*temporary tuple storage*)模块扫描内表一次，将内表元组写入work_mem或临时文件。与使用缓冲区管理器相比，它有可能更有效地处理内表元组，特别是如果至少所有的元组都写到work_mem中。 

图3.17演示了materialized nested loop join是如何执行的。 扫描物化元组(materialized tuples)在内部称为**重新扫描(rescan**)。

**图. 3.17. Materialized nested loop join.**

![Fig. 3.17. Materialized nested loop join.](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch3/fig-3-17.png?raw=true)

 

*临时元组存储 Temporary Tuple Storage*

PostgreSQL内部提供了一个临时元组存储模块，用于在hybrid hash join中创建batch等等。  这个模块由[tuplestore.c](https://github.com/postgres/postgres/blob/master/src/backend/utils/sort/tuplestore.c)中定义的函数组成，它们存储并从work_mem或临时文件中读取一系列元组。 是否使用work_mem或临时文件取决于要存储的元组的总大小。

我们将探究执行器如何处理materialized nested loop join的计划树，以及如何使用下面显示的特定示例估算成本。

```sql
testdb=# EXPLAIN SELECT * FROM tbl_a AS a, tbl_b AS b WHERE a.id = b.id;
						QUERY PLAN                               
----------------------------------------------------------------------- 
Nested Loop  (cost=0.00..750230.50 rows=5000 width=16)   
	Join Filter: (a.id = b.id)   
	->  Seq Scan on tbl_a a  (cost=0.00..145.00 rows=10000 width=8)   
	->  Materialize  (cost=0.00..98.00 rows=5000 width=8)         
		->  Seq Scan on tbl_b b  (cost=0.00..73.00 rows=5000 width=8)
(5 rows)
```

首先，给出了执行器的操作。执行器按以下方式处理显示的计划节点： 

- **第7行**: 执行器通过顺序扫描物化的内表tbl_b(第8行)。
- **第4行**: 执行器执行nested loop join操作; 外表是tbl_a，内表是物化的tbl_b。

在下文中，估计‘Materialize’(第7行)和 ‘Nested Loop’(第4行)的成本。 假设物化的内部元组存储在work_mem中。

**Materialize:**

没有启动成本；因此， 

​	‘start-up cost’ = 0.

运行成本由以下公式定义： 

​	‘run cost’ = 2 × cpu_operator_cost × $N_{inner}$;

因此，

​	‘run cost’ = 2 × 0.0025 × 5000 = 25.0.

另外，

​	‘total cost’ = (‘start-up cost’ + ‘total cost of seq scan’) + ‘run cost’;

因此，

​	‘total cost’ = (0.0 + 73.0) + 25.0 = 98.0.

**(Materialized) Nested Loop:**

没有启动成本；因此，

​	‘start-up cost’ = 0.

在估算运行成本之前，我们考虑*重新扫描成本(rescan cost)*。 该成本由以下等式定义：

​	‘rescan cost’ = cpu_operator_cost × $N_{inner}$.

在这种情况下，

​	‘rescan cost’ = (0.0025) × 5000 = 12.5.

运行成本由以下等式定义：

​	‘run cost’ = (cpu_operator_cost + cpu_tuple_cost) × $N_{inner}$ × $N_{outer}$

​			+ ‘rescan cost’ × ($N_{outer}$ − 1) + $C^{total}_{outer,seqscan}$ + $C^{total}_{materialize}$,

其中$C^{total}_{outer,seqscan}$是外表的总扫描成本，$C^{total}_{materialize}$是materialized的总成本; 因此，

​	‘run cost’ = (0.0025 + 0.01) × 5000 × 10000 + 12.5 × (10000 − 1) + 145.0 + 98.0 = 750230.5.

#### 3.5.1.3. Indexed Nested Loop Join

如果存在内表的索引，并且该索引可以查找满足连接条件的元组以匹配外表的每个元组，则优化器考虑使用该索引直接搜索内部元组而不是顺序扫描。 这种变化被称为**indexed nested loop join**; 参考图3.18。尽管它提到了索引化的‘nested loop join’，但该算法可以基于外表的单个循环进行处理; 因此，它可以高效地执行连接操作。

**图. 3.18. Indexed nested loop join.**

![Fig. 3.18. Indexed nested loop join.](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch3/fig-3-18.png?raw=true)

下面显示了indexed nested loop join的具体示例。

```sql
testdb=# EXPLAIN SELECT * FROM tbl_c AS c, tbl_b AS b WHERE c.id = b.id;
							QUERY PLAN                                   
-------------------------------------------------------------------------------- 
Nested Loop  (cost=0.29..1935.50 rows=5000 width=16)   
	->  Seq Scan on tbl_b b (cost=0.00..73.00 rows=5000 width=8)   
	->  Index Scan using tbl_c_pkey on tbl_c c  (cost=0.29..0.36 rows=1 width=8)         
		Index Cond: (id = b.id)
(4 rows)
```

在第6行中，显示访问内表的元组的成本。 如果元组满足第7行所示的索引条件(id = b.id)，这就是查找内表的成本。

在第7行的索引条件(id = b.id)中，'b.id'是连接条件中使用的外表属性的值。 每当通过顺序扫描检索外表的元组时，第6行中的索引扫描路径查找要连接的内部元组。 换句话说，每当将外表的值作为参数传递时，此索引扫描路径将查找满足连接条件的内部元组。 这种索引路径称为**parameterized (index) path**。 详细信息在[README](https://github.com/postgres/postgres/blob/master/src/backend/optimizer/README)中介绍。

此nested loop join的启动成本等于第6行中索引扫描的成本; 从而，

​	 ‘start-up cost’ = 0.285.

indexed nested loop join 的总成本由以下等式定义

​	 ‘total cost’ = (cpu_tuple_cost + $C^{total}_{inner,parameterized}$) × ${N_{outer}}$ + ${C^{run}_{outer,seqscan}}$, 

其中$C^{total}_{inner,parameterized}$是parameterized inner index scan的总成本。

在这种情况下，

​	 ‘total cost’ = (0.01 + 0.3625) × 5000 + 73.0 = 1935.5, 

运行成本是

​	‘run cost’ = 1935.5 − 0.285 = 1935.215.

综上所述，indexed nested loop join的总成本为O(${N_{outer}}$)。

#### 3.5.1.4. 其他变体

如果存在外表的索引并且它的属性参与了连接条件，则可以将其用于索引扫描，而不是外表的顺序扫描。 特别是，如果在WHERE子句中有一个索引的属性可以是访问谓词，则会缩小外表的搜索范围; 因此，nested loop join的成本可能会大大降低。

PostgreSQL支持带外部索引扫描的nested loop join的三种变体。参考图。3.19. 

**图. 3.19. nested loop join的三种变体与外部索引扫描**

![Fig. 3.19. The three variations of the nested loop join with an outer index scan.](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch3/fig-3-19.png?raw=true)

[这里](javascript:void(0)) 展示这些join的EXPLAIN结果

### 3.5.2. Merge Join

与nested loop join不同，merge join只能用于natural joins和[equi-joins.](https://en.wikipedia.org/wiki/Join_(SQL)#Equi-join)。

merge join的开销由initial_cost_mergejoin()和final_cost_mergejoin()函数估算。

由于确切的成本估算比较复杂，因此省略它仅显示merge join算法的运行时顺序。meger join 的启动成本是内表和外表的排序成本的总和; ($N_{outer}$$log_{2}$($N_{outer}$) + $N_{inner}$$log_{2}$($N_{inner}$))，其中$N_{outer}$和$N_{inner}$分别是外表和内表的元组数 。 运行成本是O($N_{outer}$r +  $N_{inner}$)。

与nested loop join类似，PostgreSQL中的merge join有四种变体。

#### 3.5.2.1. Merge Join

图 3.20 显示了merge join的概念图。

**图. 3.20. Merge join.**

![Fig. 3.20. Merge join.](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch3/fig-3-20.png?raw=true)

如果所有元组都可以存储在内存中，则排序操作将能够在内存中执行; 否则，使用临时文件。

下面给出merge join 的EXPLAIN命令的结果的一个具体示例。

merge join 的EXPLAIN命令结果如下。 

```sql
testdb=# EXPLAIN SELECT * FROM tbl_a AS a, tbl_b AS b WHERE a.id = b.id AND b.id < 1000;
							QUERY PLAN
------------------------------------------------------------------------- 
Merge Join  (cost=944.71..984.71 rows=1000 width=16)   
	Merge Cond: (a.id = b.id)   
	->  Sort  (cost=809.39..834.39 rows=10000 width=8)         
		Sort Key: a.id         
		->  Seq Scan on tbl_a a  (cost=0.00..145.00 rows=10000 width=8)   
	->  Sort  (cost=135.33..137.83 rows=1000 width=8)         
		Sort Key: b.id         
		->  Seq Scan on tbl_b b  (cost=0.00..85.50 rows=1000 width=8)               
			Filter: (id < 1000)
(9 rows)
```

- **第9行**: 执行器使用顺序扫描对内表tbl_b进行排序(第11行)。 
- **第6行**: 执行器使用顺序扫描对外表tbl_a进行排序(第8行)。 
- **第4行**: 执行器执行meger join操作; 外表是排序的tbl_a，内表是排序的tbl_b。

#### 3.5.2.2. Materialized Merge Join

与nested loop join相同，merge join也支持materialized merge join 物化内表，以使内表扫描更高效。

**图. 3.21. Materialized merge join.**

![Fig. 3.21. Materialized merge join.](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch3/fig-3-21.png?raw=true)

materialized merge join示例如下。很容易发现，与上面的merge join结果的不同之处在于第9行：‘Materialize’。

```sql
testdb=# EXPLAIN SELECT * FROM tbl_a AS a, tbl_b AS b WHERE a.id = b.id;
QUERY PLAN                                     
----------------------------------------------------------------------------------- 
Merge Join  (cost=10466.08..10578.58 rows=5000 width=2064)   
	Merge Cond: (a.id = b.id)   
	->  Sort  (cost=6708.39..6733.39 rows=10000 width=1032)         
		Sort Key: a.id         
		->  Seq Scan on tbl_a a  (cost=0.00..1529.00 rows=10000 width=1032)   
	->  Materialize  (cost=3757.69..3782.69 rows=5000 width=1032)         
		->  Sort  (cost=3757.69..3770.19 rows=5000 width=1032)               
			Sort Key: b.id               
			->  Seq Scan on tbl_b b  (cost=0.00..1193.00 rows=5000 width=1032)
(9 rows)
```

- **第10行**: 执行器使用顺序扫描对内表tbl_b进行排序(第12行)。
- **第9行**: 执行器物化排序后的表tbl_b的结果。
- **第6行**: 执行器使用顺序扫描对外表tbl_a进行排序(第8行)。
- **第4行**: 执行器执行merge join操作; 外表是排序后的tbl_a，内表是物化排序的tbl_b。

#### 3.5.2.3. 其他变体

与nested loop join类似，PostgreSQL中的merge也有各种变体，可以根据这些变体执行外表的索引扫描。 

图. 3.22. merge join的三个变体与外部索引扫描 

![Fig. 3.22. The three variations of the merge join with an outer index scan.](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch3/fig-3-22.png?raw=true)

[这里](javascript:void(0)) 展示这些join的EXPLAIN结果

### 3.5.3. Hash Join

与merge join类似，hash join只能用于natural joins和equi-joins。

PostgreSQL中的hash join根据表的大小而有所不同。 如果目标表足够小(更确切地说，内表的大小是work_mem的25％或更少)，它将是一个简单的两阶段内存中hash join; 否则，使用hybrid hash join with the skew方法。

在本小节中，描述了PostgreSQL中两种hash join的执行方式。

由于成本估算的复杂性，对成本估算的讨论被忽略了。粗略地说，如果在搜索和插入哈希表时没有冲突，则启动和运行成本是O($N_{outer}$+$N_{inner}$)。 

#### 3.5.3.1. In-Memory Hash Join

在本小节中，将描述in-memory hash join。 

in-memory hash join是在work_mem上处理的，在PostgreSQL中，哈希表区域称为**batch**。batch有散列槽，称为**buckets**，桶的数量由[nodeHash.c](https://github.com/postgres/postgres/blob/master/src/backend/executor/nodeHash.c);中定义的ExecChooseHashTableSize()函数决定；桶的数量总是$2^{n}$，其中$n$是整数。 

in-memory hash join有两个阶段：**build**阶段和**probe**阶段。 在build阶段，内表的所有元组都被插入batch; 在probe阶段，将外表的每个元组与batch中的内部元组进行比较，如果连接条件满足，则将其连接。 

举例描述这个操作。 假设下面显示的查询是使用hash join执行的。

```sql
testdb=# SELECT * FROM tbl_outer AS outer, tbl_inner AS inner WHERE inner.attr1 = outer.attr2;
```

下面显示hash join的操作。参考图3.23和3.24。 

**图. 3.23. in-memory hash join的build阶段**

![Fig. 3.23. The build phase in the in-memory hash join.](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch3/fig-3-23.png?raw=true)

(1) 在work_mem上创建一个batch。

​	在这个例子中，该batch有八个桶; 也就是说，桶的数量是$2^{3}$次方。

(2) 将内表的第一个元组插入batch的相应存储桶中。

  详情如下：

1. 计算连接条件中涉及的第一个元组属性的hash-key。 

   在这个例子中，由于WHERE子句是'inner.attr1 = outer.attr2'，因此使用内置hash函数计算第一个元组的属性'attr1'的hash-key。

2. 将第一个元组插入到相应的存储桶中。

   假设第一个元组的hash-key由二进制表示法为'0x000 ... 001'; 即最后三位是'001'。 在这种情况下，该元组被插入到key为'001'的桶中。

  在此文档中，构建batch的这种插入操作由以下运算符表示： $⊕$

(3) 插入内表的其余元组。

**图. 3.24. in-memory hash join的probe阶段**

![Fig. 3.24. The probe phase in the in-memory hash join.](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch3/fig-3-24.png?raw=true)

(4) Probe外表的第一个元组

   详情如下：

1. 计算第一个元组属性的hash-key，该属性涉及到外表的连接条件。 

   在这个例子中，假设第一个元组属性'attr2'的hash-key是'0x000 ... 100'; 也就是说，最后三位是'100'。

2. 将外表的第一个元组与batch中的内部元组进行比较，如果满足连接条件，则将其与连接元组进行比较。 

   因为第一个元组的hash-key的最后三位是“100”，所以执行器检索key为“100”的桶的元组，并比较连接条件(由WHERE子句定义)指定的表的各自属性值。 

   如果满足连接条件，则外表的第一个元组和内表的相应元组将被连接；否则，执行器将不执行任何操作。 

   在本例中，key为'100'的桶有Tuple_C。 如果Tuple_C的attr1等于第一个元组(Tuple_W)的attr2，则Tuple_C和Tuple_W将被连接并保存到内存或临时文件中。

  在这个文档中，构建batch的这种插入操作由以下运算符表示：$⊗$

(5) Probe 外表的其余元组。

#### 3.5.3.2. Hybrid Hash Join with Skew

当内表的元组无法存储到work_mem中的一个batch中时，PostgreSQL将使用hybrid hash join with skew，这是一种基于hybrid hash join的变体。

首先介绍了hybrid hash join的基本概念。在第一个build和probe阶段，PostgreSQL准备多个batch。batch数目与由ExecChooseHashTableSize()函数确定的桶数相同；它总是$2_{m}$，其中$m$是整数。在此阶段，在work_mem中只分配一个batch，而其他batch作为临时文件创建；属于这些batch的元组被写入相应的文件，并使用临时元组存储功能保存。 

图3.25说明了元组是如何存储在四个(=2222)batch中的。在这种情况下，存储每个元组的batch由元组hash-key最后5位的前两位决定，因为桶和batch的大小分别为2323和2222。Batch_0存储hash-key的最后5位介于‘00000’和‘00111’之间的元组，Batch_1存储hash-key的最后5位在‘01000’和‘01111’之间的元组，以此类推。 

**图. 3.25. hybrid hash join中多个batch**

![Fig. 3.25. Multiple batches in hybrid hash join.](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch3/fig-3-25.png?raw=true)

在hybrid hash join中，build和probe阶段的执行次数与batch数相同，因为内表和外表存储在相同数量的batch中。在第一轮build和probe阶段，不仅创建了每个batch，而且还处理了第一批内表和外表。另一方面，第二轮和后续各轮的处理需要写入临时文件并从临时文件中重新加载 ，所以这些过程成本很高。因此，PostgreSQL还准备了一个名为**skew**的特殊batch，以便在第一轮中更高效地处理许多元组。

skew batch存储将与外表元组连接的内表元组，这些外表元组参与连接条件的属性的MCV值相对较大。 但是，因为这个解释不容易理解，所以会用一个具体的例子来解释。

假设有两个表：customers和purchase_history。 顾客表由两个属性组成：name和address; purchase_history表由两个属性组成：customer_name和purchased_item。 顾客表有10,000行，purchase_history表有1,000,000行。 前10％的顾客购买了所有物品的70％。

在这些假设下，我们考虑执行下面的查询时，第一轮如何执行hybrid hash join with skew。

```sql
testdb=# SELECT * FROM customers AS c, purchase_history AS h WHERE c.name = h.customer_name;
```

如果客户表为内表，而purchase_history为外表，则前10％的客户将使用purchase_history表的MCV值存储在skew batch中。 请注意，引用了外表的MCV值，以将内表元组插入到skew batch中。 在第一轮的probe阶段，外表(purchase_history)的元组中的70％将与存储在skew batch中的元组连接。 这样，外表分布的不均匀性就越大，它可以在第一轮中处理外表的许多元组。

在下文中，展示hybrid hash join with skew的工作机制。 参考图3.26至3.29。

**图. 3.26. hybrid hash join第一轮build阶段**

![Fig. 3.26. The build phase of the hybrid hash join in the first round.](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch3/fig-3-26.png?raw=true)

(1) 在work_mem上创建一个batch和一个skew batch。

(2) 创建用于存储内表元组的临时batch文件。

​	在这个例子中，创建了三个batch文件，因为内表将被四个batch分开。

(3) 执行内表第一个元组的build操作。

   详情如下：

1. 如果第一个元组应该插入到skew batch中，请执行此操作; 否则，请继续执行2。

   在上面解释的例子中，如果第一个元组是前10％的客户之一，则将其插入到skew batch中。

2. 计算第一个元组的hash-key，然后插入相应的batch。

(4) 对内表的其余元组执行build操作。

**图 3.27. hybrid hash join第一轮probe阶段**

![Fig. 3.27. The probe phase of the hybrid hash join in the first round.](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch3/fig-3-27.png?raw=true)

(5) 创建用于存储外表元组的临时batch文件。

(6) 如果第一个元组的MCV值较大，请使用skew batch执行probe操作; 否则，跳到(7)。

​	在上面解释的例子中，如果第一个元组是前10％的客户的购买数据，则将其与skew batch中的元组进行比较。

(7) 执行第一个元组的probe操作。

​	根据第一个元组的hash-key值，执行以下过程：

​	如果第一个元组属于Batch_0，则执行probe操作。

​	否则，插入到相应的batch中。

(8) 从外表的其余元组中执行probe操作。 请注意，在该示例中，外表的元组中的70％已在第一轮中都经过skew处理。

**图. 3.28. 第二轮build和probe阶段**

![Fig. 3.28. The build and probe phases in the second round.](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch3/fig-3-28.png?raw=true)

(9) 删除skew batch并清除Batch_0以准备第二轮。

(10) 从batch文件'batch_1_in'执行build操作。

(11) 对存储在batch文件'batch_1_out'中的元组执行probe操作。

**图. 3.29. 第三轮和最后一轮build和probe阶段**

![Fig. 3.29. The build and probe phases in the third and the last rounds.](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch3/fig-3-29.png?raw=true)

(12) 使用batch文件'batch_2_in'和'batch_2_out'执行build和probe操作。

(13) 使用batch文件'batch_3_in'和'batch_3_out'执行build和probe操作。

### 3.5.4. Join Access Paths and Join Nodes

#### 3.5.4.1. Join Access Paths

nested loop join的访问路径是 [JoinPath](javascript:void(0)) 结构，其他连接访问路径 [MergePath](javascript:void(0)) 和 [HashPath](javascript:void(0)) 都是基于该结构的。

所有连接访问路径都将在下面进行说明，无需解释。 

**图. 3.30. 连接访问路径**

![Fig. 3.30. Join access paths.](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch3/fig-3-30.png?raw=true)

#### 3.5.4.2. Join Nodes

本小节显示三个没有解释的连接节点： [NestedLoopNode](javascript:void(0)), [MergeJoinNode](javascript:void(0)) 和 [HashJoinNode](javascript:void(0)) 。 它们基于 [JoinNode](javascript:void(0)) 结构。

## 3.6. 创建多表查询的计划树

在本节中，将描述创建多表查询的计划树的过程。

### 3.6.1. 预处理

调用[planner.c](https://github.com/postgres/postgres/blob/master/src/backend/optimizer/plan/planner.c)中定义的subquery_planner() 函数进行预处理。 单表查询的预处理已在第3.3.1节中描述。 在本小节中，将描述多表查询的预处理; 然而，虽然相关内容很多，但只介绍一部分。

1. 计划和转换 CTE

   如果有WITH列表，则优化器通过SS_process_ctes() 函数处理每个WITH查询。

2. 拉取子查询

   如果FROM子句具有子查询并且它没有GROUP BY，HAVING，ORDER BY，LIMIT和DISTINCT子句，并且它不使用INTERSECT或EXCEPT，则优化器将通过pull_up_subqueries() 函数转换为join形式。 例如，下面显示的包含FROM子句中的子查询的查询可以转换为natural join查询。 不用说，这个转换是在查询树中完成的。

```sql
testdb=# SELECT * FROM tbl_a AS a, (SELECT * FROM tbl_b) as b WHERE a.id = b.id;
   	 	       	     ↓
testdb=# SELECT * FROM tbl_a AS a, tbl_b as b WHERE a.id = b.id;
```

4. 将外连接转换为内连接

   如果可能，优化器将外连接查询转换为内连接查询。


### 3.6.2. 获得最优路径

为了得到最优的计划树，优化器必须考虑所有索引的组合和连接方法的可能性。 这是一个非常昂贵的过程，如果由于组合爆炸导致表的数量超过某个水平，这将是不可行的。 幸运的是，如果表的数量少于12个左右，优化器可以通过动态规划来获得最优计划。 否则，优化器使用遗传算法。 请参阅下面的内容。

*遗传查询优化器 Genetic Query Optimizer*

当执行连接多个表的查询时，需要大量的时间来优化查询计划。 为了处理这种情况，PostgreSQL实现了一个有趣的功能：[Genetic Query Optimizer](http://www.postgresql.org/docs/current/static/geqo.html)。 这是一种在合理时间内确定合理计划的近似算法。 因此，在查询优化阶段，如果连接表的数量高于参数[geqo_threshold](http://www.postgresql.org/docs/current/static/runtime-config-query.html#GUC-GEQO-THRESHOLD)指定的阈值(缺省值为12)，PostgreSQL使用遗传算法生成查询计划。 

以下步骤描述通过动态规划确定最优计划树：

*Level = 1*

​	获取每张表的最优路径; 最优路径存储在相应的RelOptInfo中。

*Level = 2*

​	为每个组合选择最优路径，从所有表中选择两个。

​	例如，如果有两个表A和B，则获得表A和B的最优连接路径，这是最终答案。

​	下面，两个表的RelOptInfo由{A，B}表示。

​	如果有三个表，则为{A，B}，{A，C}和{B，C}中的每一个找得最优路径。

*Level = 3 and higher*

​	继续相同的处理，直到level等于表的数量。

这样，部分问题的最优路径在每个级别获得并且被用于获得上级的计算。这使得有效地计算最优计划树成为可能。 

**图. 3.31. 使用动态规划获得最优访问路径**

![Fig. 3.31. How to get the cheapest access path using dynamic programming.](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch3/fig-3-31.png?raw=true)

在下文中，描述优化器获取以下查询的最优计划的过程。

```sql
testdb=# \d tbl_a
     Table "public.tbl_a"
 Column |  Type   | Modifiers 
--------+---------+-----------
 id     | integer | not null
 data   | integer | 
Indexes:
    "tbl_a_pkey" PRIMARY KEY, btree (id)

testdb=# \d tbl_b
     Table "public.tbl_b"
 Column |  Type   | Modifiers 
--------+---------+-----------
 id     | integer | 
 data   | integer | 

testdb=# SELECT * FROM tbl_a AS a, tbl_b AS b WHERE a.id = b.id AND b.data < 400;
```

#### 3.6.2.1. Level 1 预处理

在Level 1中，优化器创建RelOptInfo结构并估计查询中每个关系的最低成本。 在这，RelOptInfo结构被添加到该查询的PlannerInfo的simple_rel_arrey。

**图. 3.32. Level 1预处理后的PlannerInfo 和 RelOptInfo**

![Fig. 3.32. The PlannerInfo and RelOptInfo after processing in Level 1.](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch3/fig-3-32.png?raw=true)

tbl_a的RelOptInfo有三条访问路径，它们被添加到RelOptInfo的pathlist中，并且链接到三个最低成本路径，即*cheapest start-up (cost) path*，*cheapest total (cost) path*和*cheapest parameterized (cost) path*。 由于最优启动和总成本路径是显而易见的，因此将描述cheapest parameterized index scan path的成本。

如第3.5.1.3节所述，优化器考虑使用indexed nested loop join的parameterized path(并且很少将indexed merge join与outer index scan一起使用)。 cheapest parameterized cost是估计parameterized path的最低成本。

tbl_b的RelOptInfo只具有顺序扫描访问路径，因为tbl_b没有相关的索引。

#### 3.6.2.2. Level 2 预处理

在Level 2中，创建RelOptInfo结构并将其添加到PlannerInfo的join_rel_list中。 然后，估计所有可能的join路径的成本，并选择总成本最低的最优访问路径。RelOptInfo将最优访问路径存储为最优总成本路径。 参考图3.33。

**图. 3.33. Level 2预处理后的PlannerInfo 和 RelOptInfo**

![Fig. 3.33. The PlannerInfo and RelOptInfo after processing in Level 2.](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch3/fig-3-33.png?raw=true)

表3.1展示这个示例中所有连接访问路径的组合。示例的查询是一个equi-join类型; 因此，估计所有三种连接方法。 为了方便起见，引入了一些访问路径的名词：

- *SeqScanPath(table)* 表示表的顺序扫描路径。
- *Materialized->SeqScanPath(table)* 表示表的物化顺序扫描路径。
- *IndexScanPath(table, attribute)* 表示由表的属性指定的索引扫描路径。
- *ParameterizedIndexScanPath(table, attribute1, attribute2)* 由表的属性1指定的参数化索引路径，并且由外表的属性2进行参数化。

**表 3.1: 本例中所有组合的访问路径** 

|                  | Outer Path              | Inner Path                                      |                                                     |
| ---------------- | ----------------------- | ----------------------------------------------- | --------------------------------------------------- |
| Nested Loop Join |                         |                                                 |                                                     |
| 1                | SeqScanPath(tbl_a)      | SeqScanPath(tbl_b)                              |                                                     |
| 2                | SeqScanPath(tbl_a)      | Materialized->SeqScanPath(tbl_b)                | Materialized nested loop join                       |
| 3                | IndexScanPath(tbl_a,id) | SeqScanPath(tbl_b)                              | Nested loop join with outer index scan              |
| 4                | IndexScanPath(tbl_a,id) | Materialized->SeqScanPath(tbl_b)                | Materialized nested loop join with outer index scan |
| 5                | SeqScanPath(tbl_b)      | SeqScanPath(tbl_a)                              |                                                     |
| 6                | SeqScanPath(tbl_b)      | Materialized->SeqScanPath(tbl_a)                | Materialized nested loop join                       |
| 7                | SeqScanPath(tbl_b)      | ParametalizedIndexScanPath(tbl_a, id, tbl_b.id) | Indexed nested loop join                            |
| Merge Join       |                         |                                                 |                                                     |
| 1                | SeqScanPath(tbl_a)      | SeqScanPath(tbl_b)                              |                                                     |
| 2                | IndexScanPath(tbl_a,id) | SeqScanPath(tbl_b)                              | Merge join with outer index scan                    |
| 3                | SeqScanPath(tbl_b)      | SeqScanPath(tbl_a)                              |                                                     |
| Hash Join        |                         |                                                 |                                                     |
| 1                | SeqScanPath(tbl_a)      | SeqScanPath(tbl_b)                              |                                                     |
| 2                | SeqScanPath(tbl_b)      | SeqScanPath(tbl_a)                              |                                                     |

例如，在nested loop join中，估计七条连接路径。 第一个表示外路径和内路径分别是tbl_a和tbl_b的顺序扫描路径; 第二个表示外路径是tbl_a的顺序扫描路径，而内路径是tbl_b的物化顺序扫描路径; 等等。

优化器最终从估计的连接路径中选择最优访问路径，并将最优路径添加到RelOptInfo {tbl_a，tbl_b}的pathlist中。 参考图3.33。

在这个例子中，如下面EXPLAIN的结果所示，优化器选择内表和外表为tbl_b和tbl_c的hash join。

```sql
testdb=# EXPLAIN  SELECT * FROM tbl_b AS b, tbl_c AS c WHERE c.id = b.id AND b.data < 400;
						QUERY PLAN                              
---------------------------------------------------------------------- 
Hash Join  (cost=90.50..277.00 rows=400 width=16)   
	Hash Cond: (c.id = b.id)   
	->  Seq Scan on tbl_c c  (cost=0.00..145.00 rows=10000 width=8)   
	->  Hash  (cost=85.50..85.50 rows=400 width=8)         
		->  Seq Scan on tbl_b b  (cost=0.00..85.50 rows=400 width=8)               
			Filter: (data < 400)
(6 rows)
```

### 3.6.3. 获取三表查询的最优路径 

获取涉及三个表的查询的最优路径如下：

```sql
testdb=# \d tbl_a
     Table "public.tbl_a"
 Column |  Type   | Modifiers 
--------+---------+-----------
 id     | integer | 
 data   | integer | 

testdb=# \d tbl_b
     Table "public.tbl_b"
 Column |  Type   | Modifiers 
--------+---------+-----------
 id     | integer | 
 data   | integer | 

testdb=# \d tbl_c
     Table "public.tbl_c"
 Column |  Type   | Modifiers 
--------+---------+-----------
 id     | integer | not null
 data   | integer | 
Indexes:
    "tbl_c_pkey" PRIMARY KEY, btree (id)

testdb=# SELECT * FROM tbl_a AS a, tbl_b AS b, tbl_c AS c 
testdb-#                WHERE a.id = b.id AND b.id = c.id AND a.data < 40;
```

 **Level 1:**

优化器估计所有表的最优路径，并将这些信息存储在相应的RelOptInfos：{tbl_a}，{tbl_b}和{tbl_c}中。

**Level 2:**

优化器选取三张表中的所有组合，并估计每个组合的最优路径; 优化器然后将信息存储在相应的RelOptInfos：{tbl_a，tbl_b}，{tbl_b，tbl_c}和{tbl_a，tbl_c}中。

**Level 3:**

优化器最终使用已获得的RelOptInfos获得最优路径。 更准确地说， 规划器考虑了RelOptInfos的三个组合：{tbl_a，{tbl_b，tbl_c}，{tbl_b，{tbl_a，tbl_c}和{tbl_c，{tbl_a，tbl_b}，因为{tbl_a，tbl_b，tbl_c}是由它们组成的。 

​	{tbl_a,tbl_b,tbl_c} = {tbl_a,{tbl_b,tbl_c}} + {tbl_b,{tbl_a,tbl_c}} + {tbl_c,{tbl_a,tbl_b}}.

然后，优化器会估计其中所有可能的连接路径的成本。

在RelOptInfo {tbl_c，{tbl_a，tbl_b}}中，优化器估计tbl_c和{tbl_a，tbl_b}的最优路径的所有组合，tbl_a，tbl_b是其内表和外表分别为tbl_a和tbl_b的hash join， 这个例子。 估计的连接路径将包含三种连接路径及其变体，如前一小节所示，即nested loop join及其变体，merge join及其变体以及hash join。

优化器以相同的方式处理RelOptInfos {tbl_a，{tbl_b，tbl_c}}和{tbl_b，{tbl_a，tbl_c}}，最后从所有估计的路径中选择最优访问路径。

下面显示了该查询的EXPLAIN命令的结果：

![img](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch3/fig-3-34.png?raw=true)

最外层的连接是indexed nested loop join(第5行)；inner parameterized index scan显示在第13行中，第7-12行是hash join的结果，其内表和外表分别是tbl_b和tbl_a。因此，执行器首先执行tbl_a和tbl_b的hash join，然后执行indexed nested loop join。 

## 参考

- [1] Abraham Silberschatz, Henry F. Korth, and S. Sudarshan, "[Database System Concepts](https://www.amazon.com/dp/0073523321)", McGraw-Hill Education, ISBN-13: 978-0073523323
- [2] Thomas M. Connolly, and Carolyn E. Begg, "[Database Systems](https://www.amazon.com/dp/0321523067)", Pearson, ISBN-13: 978-0321523068