---
title: '好文推荐：《Spanner: Google''s Globally-Distributed Database》'
comments: true
date: 2016-10-05 11:23:57
tags: ["读书","数据库"]
categories: 读书
---

> Spanner是谷歌公司研发的、可扩展的、多版本、全球分布式、同步复制数据库。它是第一个把数据分布在全球范围内的系统，并且支持外部一致性的分布式事务。本文描述了Spanner的架构、特性、不同设计决策的背后机理和一个新的时间API，这个API可以暴露时钟的不确定性。这个API及其实现，对于支持外部一致性和许多强大特性而言，是非常重要的，这些强大特性包括：非阻塞的读、不采用锁机制的只读事务、原子模式变更。

附上原文[Spanner: Google's Globally-Distributed Database](/uploads/spanner.pdf)
