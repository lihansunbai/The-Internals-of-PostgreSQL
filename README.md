# PostgreSQL的内部结构

##### 针对数据库管理员和系统开发人员

![Copyright Diego Zazzeo](imgs/ch0/puestas-fauna-mecanica-c.png)

------

- 作者： [Hironobu SUZUKI](http://www.interdb.jp/)
- 原书名称：[《The Internals of PostgreSQL》](http://www.interdb.jp/pg/index.html)
- 译者：[杨杰](https://yonj1e.github.io/young/) （[yonj1e@163.com](mailto:yonj1e@163.com) ）

## 法律声明

译者纯粹出于**学习目的**与**个人兴趣**翻译本书，不追求任何经济利益。

译者保留对此版本译文的署名权，其他权利以原作者和出版社的主张为准。

本译文只供学习研究参考之用，不得公开传播发行或用于商业用途。

## 目录

- [简介](ch0.md)
- [第一章.](ch1.md) 数据库集群、数据库和表
- [第二章.](ch2.md) 进程和内存结构
- [第三章.](ch3.md) 查询处理
- [第四章.](ch4.md) 外部表(FDW)和并行查询
- [第五章.](ch5.md) 并发控制
- [第六章.](ch6.md) VACUUM
- [第七章.](ch7.md) HOT(Heap-Only-Tuple)技术和仅索引扫描(Index-Only Scans)
- [第八章.](ch8.md) 缓冲管理器
- [第九章.](ch9.md) 预写式日志 (WAL)
- [第十章.](ch10.md) 基础备份和时间点恢复 (PITR)
- [第十一章.](ch11.md) 流复制

## 翻译计划

工作业余时间翻译学习，无聊一起造作啊~

![wechat](imgs/ch0/me.jpg)

## LICENSE

CC-BY 4.0
