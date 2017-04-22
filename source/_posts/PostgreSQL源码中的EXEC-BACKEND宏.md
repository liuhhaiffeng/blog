---
title: PostgreSQL源码中的EXEC_BACKEND宏
comments: true
date: 2016-11-09 08:04:10
tags: [数据库, PostgreSQL, PostgreSQL内核]
categories: 技术分享
---

看pg源码的时候经常会见到一个EXEC_BACKEND宏，在源码中并没有给出解释，那么它是什么意思呢？从http://postgresql.nabble.com/EXEC-BACKEND-td1991462.html这里找到了解释：

It exists because Windows doesn't have fork(), only the equivalent of
fork-and-exec.  Which means that no state variables will be inherited
from the postmaster by its child processes, and any state that needs to
be carried across has to be handled explicitly.  You can define
EXEC_BACKEND in a non-Windows build, for the purpose of testing code
to see if it works in that environment.

regards, tom lane 

意思是说用`EXEC_BACKEND`宏是因为Windows没有fork(),只好用别的方法来实现fork-and-exec。但是这样以来，生成的子进程没办法继承父进程的一些状态变量，所以状态变量必须显式的传给所有的子进程。在非Windows平台上编译的时候（比如linux）也可以定义`EXEC_BACKEND`宏来做调试。

可见`EXEC_BACKEND`宏是为了在win32上用来实现变量传递机制。
