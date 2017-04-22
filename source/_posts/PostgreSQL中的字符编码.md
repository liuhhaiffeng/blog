---
title: PostgreSQL中的字符编码
comments: true
date: 2016-10-27 12:02:21
tags: [数据库,PostgreSQL, PostgreSQL使用]
categories: 技术分享
---


字符编码（Character encoding）就是把某种或多种字符（比如英语字母，中文）按照某种形式编码为比特，以便于在计算机中存储和通过网络进行传递。常见的字符编码有ASCII、UTF-8、GBK等等。


PostgreSQL支持多种编码字符集，对各种语言有很好的支持。每个数据库有一个字符集，字符集在初始化数据库`initdb -E UTF-8`或新建数据库`CREATE DATABASE chinese WITH ENCODING 'UTF-8'`时确定。编码信息保存在系统表`pg_database`中。你可以在psql中用-l命令来查看数据库和对应的字符集编码。

服务器端不是对所有编码都支持，比如不支持GBK，但是PostgreSQL支持服务器发给客户端时自动转码，比如可以转码为GBK。转码规则在系统表`pg_conversion`里确定。
```
postgres=# select  conname, conproc from pg_conversion where conname like '%gbk%';
   conname   |   conproc   
-------------+-------------
 gbk_to_utf8 | gbk_to_utf8
 utf8_to_gbk | utf8_to_gbk
(2 rows)

```
比如如下例子，服务器的默认编码是UTF8
```
postgres=# \l
                               List of databases
    Name    | Owner | Encoding |   Collate   |    Ctype    | Access privileges 
------------+-------+----------+-------------+-------------+-------------------
 postgres   | yshen | UTF8     | zh_CN.UTF-8 | zh_CN.UTF-8 | 
```
客户端的通过`set client_encoding = GBK;`设为GBK
```
postgres=# set client_encoding = GBK;
SET
postgres=# show client_encoding ;
 client_encoding 
-----------------
 GBK
(1 row)
```
在查询过程中，服务器把UTF8编码转换为GBK发送到客户端
```
postgres=# select * from student;
   sno   | sname  | gender | age | nation | classno 
---------+--------+--------+-----+--------+---------
 2016001 | 王二小 | 男     |  20 | 中国   | 1-1
 2016002 | 刘胡兰 | 女     |  20 | 中国   | 1-1
 2016003 | 李小明 | 男     |  20 | 中国   | 1-2
 2016004 | 李小花 | 女     |  20 | 中国   | 1-2
(4 rows)
```

那么编码是如何在内部实现的呢？窥探一下内部函数`utf8_to_gbk`
```
Datum
utf8_to_gbk(PG_FUNCTION_ARGS)
{
	unsigned char *src = (unsigned char *) PG_GETARG_CSTRING(2);
	unsigned char *dest = (unsigned char *) PG_GETARG_CSTRING(3);
	int			len = PG_GETARG_INT32(4);

	CHECK_ENCODING_CONVERSION_ARGS(PG_UTF8, PG_GBK);

	UtfToLocal(src, len, dest,
			   ULmapGBK, lengthof(ULmapGBK),
			   NULL, 0,
			   NULL,
			   PG_GBK);

	PG_RETURN_VOID();
}

```
其实就是根据输入的字节流寻找一种编码到另一种编码的映射关系，映射关系保存在ULmapGBK这个里面,找到映射关系之后再把另一种编码对应的字节输出。

有兴趣可以移步[官方手册](https://www.postgresql.org/docs/9.5/static/multibyte.html)进一步了解
