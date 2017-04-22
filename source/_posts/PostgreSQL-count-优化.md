---
title: PostgreSQL count(*) 优化
date: 2016-10-02 12:50:10
tags: [数据库, "PostgreSQL","PostgreSQL使用"]
categories: 技术分享
---

在PostgreSQL数据库中，count(*)默认是全表扫描，那么对于数据量很大的表，因为需要把表的所有页面读入到内存，需要较多IO，耗费的时间很长。在很多应用中，需要频繁的count(*)来统计行数，那么在此类应用中如何优化count(*)的效率呢？

## 建立一个表

我们先建立一个100万行的表，来做实验。

```sql
postgres=# create table tb ( a int, b text);
CREATE TABLE
postgres=# insert into tb values ( generate_series(1,1000000), 'xxxxxx');
INSERT 0 1000000
postgres=# select count(*) from tb;
  count  
---------
 1000000
(1 row)
Time: 652.431 ms
```

实际执行下来，花费时间是652ms
下面我们来看一下执行计划

```
postgres=# explain select count(*) from tb;
                            QUERY PLAN                            
------------------------------------------------------------------
 Aggregate  (cost=17906.00..17906.01 rows=1 width=0)
   ->  Seq Scan on tb  (cost=0.00..15406.00 rows=1000000 width=0)
(2 rows)
```
可以看到，count(*)是采用顺序扫描的执行计划。

## analyze会不会有改善？

那么analyze会不会起作用呢？

```
postgres=# analyze tb;
ANALYZE
Time: 201.853 ms
postgres=# select count(*) from tb;
  count  
---------
 1000000
(1 row)
Time: 649.252 ms
```

可以看到analyze基本没什么作用。

## 主键会不会有改善？
```
postgres=# alter table tb add  primary key(a);
ALTER TABLE
Time: 2353.671 ms
postgres=# select count(*) from tb;
  count  
---------
 1000000
(1 row)

Time: 654.886 ms
postgres=# alter table tb add  primary key(a);
ERROR:  multiple primary keys for table "tb" are not allowed
Time: 1.603 ms
postgres=# explain select count(*) from tb;
                            QUERY PLAN                            
------------------------------------------------------------------
 Aggregate  (cost=17906.00..17906.01 rows=1 width=0)
   ->  Seq Scan on tb  (cost=0.00..15406.00 rows=1000000 width=0)
(2 rows)

Time: 1.246 ms

```
可以看到，加了索引之后还是在走顺序扫描，时间没有什么改善。

## 索引会不会有改善？

那么建立一个索引有没有作用？

```
postgres=# create index tb_idx on tb(a);
CREATE INDEX
Time: 2009.193 ms
postgres=# select count(*) from tb;
  count  
---------
 1000000
(1 row)

Time: 654.997 ms
postgres=# explain select count(*) from tb;
                            QUERY PLAN                            
------------------------------------------------------------------
 Aggregate  (cost=17906.00..17906.01 rows=1 width=0)
   ->  Seq Scan on tb  (cost=0.00..15406.00 rows=1000000 width=0)
(2 rows)
```

可以看到，建立了索引之后，时间依然没有明显改善，查看执行计划，可以看到依然是走的顺序扫描。
为什么不走索引呢？我们把顺序扫描关闭之后。
```
postgres=# set enable_seqscan = off;
SET
postgres=# explain (analyze,buffers) select count(*) from tb;
                              QUERY PLAN                     
                                           
----------------------------------------------------------------------
 Aggregate  (cost=33889.43..33889.44 rows=1 width=0) (actual time=1968.386..1968.387 rows=1 loo
ps=1)
   Buffers: shared hit=8141
   ->  Index Only Scan using tb_pkey on tb  (cost=0.42..31389.42 rows=1000000 width=0) (actual 
time=0.073..1542.385 rows=1000000 loops=1)
         Heap Fetches: 1000000
         Buffers: shared hit=8141
 Planning time: 0.203 ms
 Execution time: 1968.514 ms
(7 rows)
Time: 1969.929 ms
```
顺序扫描之后看到时间反而变多了！查看执行计划可以看到，用的是Index Only Scan，读取到的页面是8141，而且还要读Heap上的100万条数据来查看事务信息，所以在该例子中采用索引扫描并不能有多大提升。



## 优化方法——建触发器
如果这个表更新不那么频繁，可以建一个表用来缓存count的结果，再建一个触发器来更新缓存表。查的时候，查缓存count的那个表。
举个例子：

```sql
--建立count缓存表
 create table tb_count(col int);
--初始化count缓存表
 insert into tb_count ( select count(*) from tb);
--创建更新函数
CREATE OR REPLACE FUNCTION tb_func()  RETURNS TRIGGER AS $$
        BEGIN
                update  tb_count  set col = col + 1;
                RETURN null;
        END;
$$ LANGUAGE plpgsql; 
--创建触发器
CREATE TRIGGER tb_insert
    AFTER INSERT ON tb
    FOR EACH ROW
    EXECUTE PROCEDURE tb_func();
    
--类似的创建对于如下2个的触发器
    DELETE
    TRUNCATE
```

下面我们来看一下，优化之后的效率：

```
postgres=# select count(*) from tb;
  count  
---------
 1000001
(1 row)

Time: 898.974 ms
postgres=# select * from tb_count;
   col   
---------
 1000001
(1 row)

Time: 2.463 ms
postgres=# 

```

可以看到，优化之后时间从原来的898ms降低为2ms。




