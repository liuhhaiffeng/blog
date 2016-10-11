---
title: 数据库性能测试标准TPC
comments: true
date: 2016-10-11 07:31:30
tags: 数据库
---



> 经常有人在自己机器上跑了一个简单的查询，就发布到网上说哪个数据性能如何如何。数据库性能的好坏影响因素很多，比如业务特点、数据量、测试时间、硬件配置等等。那么有没有一个标准来度量数据库性能的好呢？

有这么一个组织：TPC，即事务处理性能委员位TPC(Transaction Processing Performance Council)，成立于88年，制定了一系列用来测试数据库性能的标准，在业内广泛应用。
TPC成员有很多软硬件厂商组成，比如IBM，INTEL传统的硬件厂商；还有ORALCE、SAP、REDHAT等软件厂商等。国内的HUAWEI也在其中。可以说TPC组织覆盖了数据库的整个生态系统。
他们的官网是：http://www.tpc.org

TPC有很多测试标准，见下图。

![TPC测试标准](http://static.zybuluo.com/shenyuflying/le1aa8mmq6jnh0191fscinj7/image_1aup9fil01m0qsda13k0mi51jmp9.png)

其中最有名的是TPC-C和TPC-H标准。TPC-C标准是TPC推出的最早的一个标准，主要用来测试交易处理的的场景。TPC-H标准主要用来测试决策支持场景。

TPC-C
------
TPC-C全称为TPC Benchmark C，主要用来测试交易处理场景下数据库的性能。该测试标准模拟了一个批发商管理订单的场景。包括了输入订单，发货，收钱，查询订单状态。性能指标有2个分别是：

- new-order txn rate (tpmC)
- price/performance ($/tpmC)

根据字面意思不难理解，tmpC是每分钟处理订单的数量，$/tmpC是一个经济指标说的是每tmpC花费。

TPC-H
-------
TPC-H全称The TPC Benchmark™H ，主要用来测试决策支持场景下数据库的性能。该测试标准模拟了商业环境中一些即席查询（Ad-hoc queries）和并发修改。这些场景都需要访问大量的数据，执行的查询非常复杂。是不是可以理解成为OLAP场景测试？

