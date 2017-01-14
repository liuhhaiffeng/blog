---
title: PostgreSQL日志分析工具——pgBadger
comments: true
date: 2016-10-13 07:29:26
tags: [数据库, "PostgreSQL", PostgreSQL使用, 调优]
categories: 技术分享
---


> 面对客户抱怨诸如“很慢”、“卡顿”等问题的时候，常用的办法就是查看服务器的日志。但是一头扎进几百M甚至几个G的日志里查看并不现实。那么如何快速回答：最慢的查询有哪些？查询相应时间分布？等等问题，并对最突出的问题着手优化呢？显然我们需要借助自动化工具来完成这个任务。

pgBadger就是为分析`PostgreSQL全日志`为目的诞生的一个工具，能够分析日志并生成分析报告，[这里](/uploads/out.html)给出了一个报告样本。怎么样，报告是不是很详细？


## pdBadger安装
pgBadger代码托管在github上，可以到[这里](https://github.com/dalibo/pgbadger/releases)进行下载，最新的版本是[v9.0](https://github.com/dalibo/pgbadger/archive/v9.0.tar.gz)
安装步骤页十分简单：
```
wget https://github.com/dalibo/pgbadger/archive/v9.0.tar.gz
tar xzf pgbadger-9.0.tar.gz
cd pgbadger-9.x/
perl Makefile.PL
make && sudo make install
```
安装完后，在终端执行`pgbadger --version`显示版本信息后，表明安装成功。
```
$ pgbadger --version
pgBadger version 9.0
```

## 收集PostgreSQL日志

要利用pgbadger来分析日志，那么我们首先要打开PostgreSQL记录日志的参数，如下是一个典型的配置：
```
log_min_duration_statement = 0 
log_checkpoints = on
log_connections = on
log_disconnections = on
log_lock_waits = on
log_temp_files = 0 
log_autovacuum_min_duration = 0 
log_line_prefix = '%t [%p]: [%l-1] db=%d,user=%u,app=%a,client=%h'
log_destination = 'stderr'  
logging_collector = on    
log_directory = 'pg_log'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_file_mode = 0600
log_rotation_age = 1d
log_rotation_size = 1000MB
lc_messages='C'
```
接着打开数据库，如果数据库正在运行那么需要发送一个SIGHUP信号来让数据库重新读取配置。过段时间，我们就能够在`pg_log`文件夹下看到日志产生了：
```
$ ls
postgresql-2016-10-13_113844.log
```

## 生成分析报告
生成分析报告非常简单只需要一步
```
$ pgbadger ./postgresql-2016-10-13_113844.log
[========================>] Parsed 5642240 bytes of 5642240 (100.00%), queries: 19436, events: 3002
LOG: Ok, generating html report...
```
稍等片刻就会输出分析报告[out.html](/uploads/out.html)

当然了，pgbadger还有一些高端玩法，比如

-  分析压缩文件
```
pgbadger /var/log/postgres.log.1.tar.gz 
```
-  分析多个文件
```
pgbadger /var/log/postgresql/postgresql-2012-05-*
```
-  排除某些类型查询
```
pgbadger --exclude-query="^(COPY|COMMIT)" /var/log/postgresql.log
```
-  指定起始时间
```
pgbadger -b "2012-06-25 10:56:11" -e "2012-06-25 10:59:11" 
                       /var/log/postgresql.log
```
-  从管道读取
```
cat /var/log/postgres.log | pgbadger -
```
-  开启多进程
```
perl pgbadger -j 8 /pglog/postgresql-9.1-main.log
```

也可以把pgbadger加入crontab计划中，来实现生成每周报告等等。


## 报告分析

报告包括了很多方面的内容：

总体信息

- 服务器traffic信息，比如select、update、insert情况
- 连接情况信息，比如每个库、用户连接情况
- 会话信息，比如会话时长
- checkpoint检查点信息
- 临时文件信息
- vacuum信息，比如做vacuum时间分布，做vacuum表情况，删除元组、页面信息
- 等待锁的信息
- 查询信息，insert/update/delete比例分布
- 慢查询排名，查询耗时分布
- 报错信息

下面介绍一下主要的几个内容

### 基本信息

![image_1auubla73147a1j8r1sf53a11ejq9.png-27.5kB](http://static.zybuluo.com/shenyuflying/f65eqmftm1egi0k8vkm68ub4/image_1auubla73147a1j8r1sf53a11ejq9.png)

### QPS

![image_1auubnceb1ah08qorkt1hct1tf3m.png-46.6kB]( http://static.zybuluo.com/shenyuflying/qtlgqigexi461t1ixoppuxkd/image_1auubnceb1ah08qorkt1hct1tf3m.png)

这里可以看到有关服务器压力的信息。

### 查询时间分布 Histogram of query times

![image_1auubrt19117m130akqdhgbcd513.png-38.5kB](http://static.zybuluo.com/shenyuflying/n5eyqk5866jo00x082uubo63/image_1auubrt19117m130akqdhgbcd513.png)

这里可以看到查询时间分布的信息。
### 慢查询排名 Slowest individual queries

![image_1auubtkfr1kvfuke1dqiknii6p1g.png-161.8kB](http://static.zybuluo.com/shenyuflying/b608kg30v8brh5m4mfupw7xt/image_1auubtkfr1kvfuke1dqiknii6p1g.png)
这里可以看到慢查询信息。
另外还有
Time Consuming queries (N) —— 最耗时的查询
Most frequent queries (N) ——　最频繁的查询
Normalized slowest queries　——　归一化后最慢的查询

PS：这里的归一化，就是排除参数不同的干扰。
### 其他

另外报告中还有关于报错信息的统计和详细情况。有助于发现服务器的故障。

## 总结

通过pgbadger可以快速分析几百M甚至几个G的服务器日志。掌握服务器运行情况并快速回答：最慢的查询有哪些？查询相应时间分布？等等问题。有助于对最突出的问题着手优化。从而快速解决客户的抱怨。
