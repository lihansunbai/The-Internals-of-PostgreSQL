# PostgreSQL的内部结构 

##### 针对数据库管理员和系统开发人员 

![Copyright Diego Zazzeo](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch0/puestas-fauna-mecanica-c.png?raw=true)

## 简介

在本文档中，针对数据库管理员和系统开发人员描述了PostgreSQL的内部结构。 

PostgreSQL是一个开源的多用途关系型数据库系统，在世界各地广泛使用。 它是一个集成多个子系统的庞大系统，每个子系统都有其独特的复杂特征，互相协同工作。 尽管理解内部机制对于使用PostgreSQL进行管理和集成至关重要，但其庞大性和复杂性决定了其难度。 本文档的主要目的是解释每个子系统是如何工作的，并且描述PostgreSQL的全貌。

本文档基于2012年以日文撰写的第二部分（ISBN-13：978-4774153926），它由七个部分组成，涵盖了版本10和更早的版本。 计划完成11章作为一个整体，现在，你可以阅读以下10章的翻译：

- [第一章.](ch1.md) 数据库集群、数据库和表
- [第二章.](ch2.md) 进程和内存结构
- [第三章.](ch3.md) 查询处理
- [第五章.](ch5.md) 并发控制
- [第六章.](ch6.md) VACUUM
- [第七章.](ch7.md) HOT(Heap-Only-Tuple)技术和仅索引扫描(Index-Only Scans)
- [第八章.](ch8.md) 缓冲管理器
- [第九章.](ch9.md) 预写式日志 (WAL)
- [第十章.](ch10.md) 基础备份和时间点恢复 (PITR)
- [第十一章.](ch11.md) 流复制

我已经暂停翻译剩下的章节一段时间了，因为我很忙。 

- 第四章. 外部表(FDW)和并行查询

导图

 

## 作者

Hironobu SUZUKI

我毕业于信息工程研究生院（M.S. in Information Engineering），曾在多家公司担任过软件开发人员和技术经理/主管。 我在数据库和系统集成领域出版了[7本书](https://www.amazon.co.jp/s/ref=dp_byline_sr_book_1?ie=UTF8&field-author=%E9%88%B4%E6%9C%A8+%E5%95%93%E4%BF%AE&search-alias=books-jp&text=%E9%88%B4%E6%9C%A8+%E5%95%93%E4%BF%AE&sort=relevancerank)（3本PostgreSQL书籍和3本MySQL书籍）。

作为日本PostgreSQL用户组(2010-2016年)的负责人，我在日本组织了六年多来最大的PostgreSQL(非商业)技术[研讨会/讲座](http://www.postgresql.jp/wg/shikumi/)，并在2008年和2009年担任了日本PostgreSQL会议的项目委员会成员，并在2013年担任了该委员会主席。 

年轻时曾在南美住过几年。 最近，有时会回到那里。

从2018年5月起，将在爱尔兰都柏林工作；目前正准备从日本搬迁。 

[Github](https://github.com/s-hironobu/)   [Blog](http://www.interdb.jp/blog/)

## 联系

在发送消息之前请先阅读此[常见问题](http://www.interdb.jp/pg/faq.html)解答疑惑。 

[email](mailto:info@interdb.jp) or [email](mailto:interdb.mx@gmail.com)  / [twitter](http://twitter.com/suzuki_hironobu)

## 版权

**© Copyright ALL Right Reserved, Hironobu SUZUKI.**

如果您想使用本文档的任何部分和/或任何图形 ，请与我联系。 **例外：教育机构可以自由使用该文件。**
