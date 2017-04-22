---
title: PostgreSQL性能分析
comments: true
date: 2016-10-13 12:49:37
tags: ["PostgreSQL","database","c"]
---

> 面对客户抱怨诸如“很慢”、“卡顿”等问题的时候，我们该如何找出性能瓶颈？从研发角度来说，做一下性能分析(profiling)可能有所帮助。性能分析能找出来最慢的函数，从而发现性能问题的瓶颈。再从瓶颈入手，优化代码从而解决性能问题。

常用的方法是gdb或pstack堆栈跟踪，比如postgreSQL经常卡在一个特定的地方很长时间，往往都有相同的堆栈跟踪信息。我们要做的就是用gdb附加到进程，将所有进程堆栈打出来，然后利用一些简单的脚本将信息做汇总，再利用sort|uniq|sort的方法排序统计出最多的堆栈信息，或最多的函数调用信息。

其实这种profile工具已经有了，其中一个简洁有效的工具是[poor man's profiler](https://poormansprofiler.org/)，Google, Facebook, Wikipedia, Intel的工程师都在用这个工具来分析程序的性能。在这里我对其改造一番，使得其更方便的分析postgreSQL数据库。

## 使用方法

```
# ./poormansprofile.sh  
usage: promansprofile username samples sleeptime
```
username是postgreSQL登陆的用户名，脚本里来找到对应用户的pid
samples是采样点数
sleeptime是每采样点数间隔
比如我们分析`yshen`用户进程，采样点300个，那么如下命令启动分析：
```
# ./poormansprofile.sh  yshen 300 0
1/300 completed.
2/300 completed.
3/300 completed.
4/300 completed.
5/300 completed.
.
.
.
300/300 completed.
```
PS: 在有些环境下，需要用root用户。

## 分析结果
这个脚本可以分析

- 函数排名：即哪个函数调用最占时间。
- 堆栈排名：即种调用堆栈最占用时间。

#### 函数排名

如下是跑回归测试的函数排名分析。
```
    266 ServerLoop
    266 PostmasterMain
    266 PostgresMain
    266 main
    266 BackendStartup
    266 BackendRun
    259 exec_simple_query
    134 CommitTransactionCommand
    134 CommitTransaction
    132 XLogFlush
    132 RecordTransactionCommit
    129 finish_xact_command
    128 XLogWrite
    128 pg_fdatasync
    128 issue_xlog_fsync
    128 __fdatasync_nocancel
    120 PortalRun
    101 standard_ProcessUtility
    101 ProcessUtility
     99 PortalRunMulti
     96 PortalRunUtility
     75 ProcessUtilitySlow
     74 smgrimmedsync
     74 pg_fsync_no_writethrough
     74 pg_fsync
     74 mdimmedsync
     74 __fsync_nocancel
     74 FileSync
     73 index_build
     69 btbuild

```
可以看到ServerLoop这个函数最占用时间，很正常，因为这个函数在等待用户连接。下面几个函数其中有个函数XLogFlush，FileSync，smgrimmedsync，pg_fdatasync等等都是IO相关，可见IO非常耗时。以上是正常状态下的函数排名，如果有异常应该就可以发现，比如一些阻塞行为，死锁，IO，网络问题都能在这里发现。

## 堆栈排名
堆栈排名更加详细，说了哪种堆栈状态最耗时。
```
     45 __fdatasync_nocancel,pg_fdatasync,issue_xlog_fsync,XLogWrite,XLogFlush,RecordTransactionCommit,CommitTransaction,CommitTransactionCommand,finish_xact_command,exec_simple_query,PostgresMain,BackendRun,BackendStartup,ServerLoop,PostmasterMain,main
      7 __fsync_nocancel,pg_fsync_no_writethrough,pg_fsync,FileSync,mdimmedsync,smgrimmedsync,_bt_load,_bt_leafbuild,btbuild,index_build,index_create,DefineIndex,ProcessUtilitySlow,standard_ProcessUtility,ProcessUtility,PortalRunUtility,PortalRunMulti,PortalRun,exec_simple_query,PostgresMain,BackendRun,BackendStartup,ServerLoop,PostmasterMain,main
      4 __fsync_nocancel,pg_fsync_no_writethrough,pg_fsync,FileSync,mdimmedsync,smgrimmedsync,_bt_load,_bt_leafbuild,btbuild,index_build,index_create,create_toast_table,CheckAndCreateToastTable,NewRelationCreateToastTable,ProcessUtilitySlow,standard_ProcessUtility,ProcessUtility,PortalRunUtility,PortalRunMulti,PortalRun,exec_simple_query,PostgresMain,BackendRun,BackendStartup,ServerLoop,PostmasterMain,mai
```
分析第一个堆栈，出现了45次，堆栈的内容是CommitTransaction提交事务带来的刷盘持久化操作，可见很耗时啊，因为带来了IO。同样的，如果发现什么异常行为也可以在这里发现。

## 源码1 ——只分析用户进程
源码如下：
如果去掉
```
sed 's/,/\n/g' | \
uniq | \
```
这两行就是显示的堆栈排名，否则是函数排名。

<!--more-->
```
#!/bin/bash

if [ $# != 3 ] ; then
        echo "usage: promansprofile username samples sleeptime"
        exit 1
fi

username=$1
nsamples=$2
sleeptime=$3

if [ $(ps aux | grep "postgres: $username" | wc -l) -eq 1 ] ; then
        echo "user process is not running"
        exit 1
fi

for x in $(seq 1 $nsamples)
  do
    ps_info=$(ps aux | grep "postgres: $username" )
    pid=$(echo $ps_info | sed -n "1, 1p" | awk '{print $2}')
    gdb -ex "set pagination 0" -ex "thread apply all bt" -batch -p $pid  2>/dev/null
    sleep $sleeptime
    echo "$x/$nsamples completed." >&2
  done | \
awk '
  BEGIN { s = ""; } 
  /^Thread/ { print s; s = ""; } 
  /^\#/ { if (s != "" ) { s = s "," $4} else { s = $4 } } 
  END { print s }' | \
sed 's/,/\n/g' | \
uniq | \
sort | \
uniq -c | \
sort -r -n -k 1,1

```

## 源码2 ——分析所有postgreSQL进程
以下源码能分析所有PostgreSQL进程，比如vacuum、log、stat、checkpoint进程。
```
#!/bin/bash

if [ $# != 2 ] ; then                                                                                                                                  
    echo "usage: promansprofile samples sleeptime"
    exit 1
fi

nsamples=$1
sleeptime=$2


for x in $(seq 1 $nsamples)
  do  
    ps_info=$(ps aux)
    ps_cnt=$(echo $ps_info | wc -l)
    for p in $(seq 1 $ps_cnt) ; do
        pid=$(echo $ps_info | sed -n "$p, 1p" | awk '{print $2}')
        gdb -ex "set pagination 0" -ex "thread apply all bt" -batch -p $pid  2>/dev/null 
    done
    sleep $sleeptime
    echo "$x/$nsamples completed." >&2
  done | \ 
awk '
  BEGIN { s = ""; } 
  /^Thread/ { print s; s = ""; } 
  /^\#/ { if (s != "" ) { s = s "," $4} else { s = $4 } } 
  END { print s }' | \ 
sed 's/,/\n/g' | \ 
uniq  |\  
sort |\
uniq -c |\
sort -r -n -k 1,1 

```


## 其他

如果有人说gdb这样不断attach到进程，是否对数据库的性能有影响，大概会拖慢0.2秒。但是为了找出性能问题所在，这样做是值得的，你的付出将会得到加倍回报。稍后我再介绍一个比gdb更快的堆栈工具，只需要1ms就能打出来堆栈。
