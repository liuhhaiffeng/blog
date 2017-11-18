---
title: 深入理解postgres嵌入式sql
comments: true
date: 2017-11-18 21:08:46
tags: [技术分享,数据库]
categories: 技术分享
---


# 简介

## What?

ESQL（Embedded SQL），嵌入式SQL：是一种将[SQL语句](https://baike.baidu.com/item/SQL%E8%AF%AD%E5%8F%A5/5714895)直接写入宿主语言的方法。宿主语言可以是C语言等等语言。


| 数据库      | 嵌入式SQL名称 | 文档                                       |
| -------- | -------- | ---------------------------------------- |
| Oracle   | Pro*C    | https://docs.oracle.com/cd/E11882_01/appdev.112/e10825/toc.htm |
| Kingbase | ESQL     | 联机帮助手册                                   |
| Postgres | ECPG     | https://www.postgresql.org/docs/10/static/ecpg-concept.html |

## Why?

这样就可以在C语言里面直接书写SQL语句，比如我们对比如下两段代码：

**C语言调用libpq**


```c
int main() {
    PGconn     *conn;
	PGresult   *res;
    int v1;
    ...
    res = PQexec(conn, "select a from test limit 1;");
	v1 = atoi(PQgetvalue(res, 0, 0));
	...
}
```

**嵌入式SQL**
```sql
int main() {
	EXEC SQL BEGIN DECLARE SECTION;
	int v1;
	EXEC SQL END DECLARE SECTION;
	...	
	EXEC SQL SELECT a INTO :v1 FROM test LIMIT 1;
	...
}
```

**优点：**

给应用研发人员带来了很大方便。这样使得研发人员可以不关注底层实现（比如上例子中的变量输出），较初级的程序员也可以有较高产出。

- 用法简单
- 检查语法

**缺点：**

- 受限制于ESQL本身的语法，不够灵活。适用于比较简单的应用。
- ESQL底层基于libpq，所以ESQL功能只是libpq的一个子集。
- 性能一般。

| 接口    | 兼容性  |  性能  | 易用性  |  功能  |
| ----- | :--: | :--: | :--: | :--: |
| ESQL  |  中   |  中   |  高   |  少   |
| libpq |  低   |  高   |  中   |  多   |
| ODBC  |  高   |  中   |  低   |  中   |

## Who?

- 应用研发人员
- 数据库研发人员

## Where?

- 新开发应用
- 迁移的应用


## When?

- 规模相对较小的应用
- 程序原型验证
- 初级研发人员


## How?

本文希望通过以下内容来让读者具备基本的ESQL研发能力

- 架构
- 客户端编程
- 代码概貌
- 近期主要工作




# 客户端编程

![](/uploads/step.png)



## hello world

### 预编译

```sql
#include<stdio.h>
#include<stdlib.h>

int main()
{
    EXEC SQL BEGIN DECLARE SECTION;
        char msg[256] = {0};
    EXEC SQL END   DECLARE SECTION;

    EXEC SQL WHENEVER SQLERROR sqlprint;
    EXEC SQL CONNECT TO postgres@localhost;

    EXEC SQL select * into :msg from helloworld;

    printf("%s\n", msg);

	EXEC SQL DISCONNECT;

}
```

```shell
$(ECPG)  helloworld.pgc -I $(ECPG_INCLUDE) -o helloworld.c
```

```c
/* Processed by ecpg (11devel) */
/* These include files are added by the preprocessor */
#include <ecpglib.h>
#include <ecpgerrno.h>
#include <sqlca.h>
/* End of automatic include section */

#line 1 "helloworld.pgc"
#include<stdio.h>
#include<stdlib.h>

int main()
{
    /* exec sql begin declare section */
           
    
#line 7 "helloworld.pgc"
 char msg [ 256 ] = { 0 } ;
/* exec sql end declare section */
#line 8 "helloworld.pgc"


    /* exec sql whenever sqlerror  sqlprint ; */
#line 10 "helloworld.pgc"


    { ECPGconnect(__LINE__, 0, "postgres@localhost" , NULL, NULL , NULL, 0); 
#line 12 "helloworld.pgc"

if (sqlca.sqlcode < 0) sqlprint();}
#line 12 "helloworld.pgc"


    { ECPGdo(__LINE__, 0, 1, NULL, 0, ECPGst_normal, "select * from helloworld", ECPGt_EOIT, 
	ECPGt_char,(msg),(long)256,(long)1,(256)*sizeof(char), 
	ECPGt_NO_INDICATOR, NULL , 0L, 0L, 0L, ECPGt_EORT);
#line 14 "helloworld.pgc"

if (sqlca.sqlcode < 0) sqlprint();}
#line 14 "helloworld.pgc"


    printf("%s\n", msg);

    { ECPGdisconnect(__LINE__, "CURRENT");
#line 18 "helloworld.pgc"

if (sqlca.sqlcode < 0) sqlprint();}
#line 18 "helloworld.pgc"


}
```
###  编译
```shell
gcc -g -O0  -I $(ECPG_INCLUDE) -L $(LIB)  -o helloworld helloworld.c -lecpg
```

###  执行


```shell
yshen@yshen-PC:~/coding/esql$ ./helloworld 
hello world
```


```sh
yshen@yshen-PC:~/coding/esql$ ldd ./helloworld
	linux-vdso.so.1 (0x00007ffe29bcd000)
	libecpg.so.6 => /home/yshen/coding/postgresql/release/lib/libecpg.so.6 
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007fe4cfd6b000)
	libpgtypes.so.3 => /home/yshen/coding/postgresql/release/lib/libpgtypes.so.3 
	libpq.so.5 => /home/yshen/coding/postgresql/release/lib/libpq.so.5 
	libpthread.so.0 => /lib/x86_64-linux-gnu/libpthread.so.0 
	/lib64/ld-linux-x86-64.so.2 (0x00007fe4d0624000)
	libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007fe4cf3ea000)
```



### Makefile

```shell
POSTGRES="/home/yshen/coding/postgresql/release"
ECPG=$(POSTGRES)"/bin/ecpg"
ECPG_INCLUDE=$(POSTGRES)"/include"
LIB=$(POSTGRES)"/lib"

execs=$(patsubst %.pgc,%,$(wildcard *.pgc))

all: $(execs)

%: %.c
	gcc -g -O0  -I $(ECPG_INCLUDE) -L $(LIB)  -o $@ $< -lecpg

%.c: %.pgc
	$(ECPG)  $< -I $(ECPG_INCLUDE) -o $@

clean:
	-rm  $(execs)
```

make

make clean

make helloworld.c





## 综合例子：备份工具

```sql
/*
 * dump utility written by ESQL
 *              yshen@2017
 */
#include<stdio.h>
#include<stdlib.h>


/* public interfaces */
void Connect(char *dbname);
void Disconnect();
void Dump();

/* internal functions */
void dump_one_table(char *tbname);
char * get_type_string(int type);

/* execution start from here */
int main()
{
    Connect("postgres");
    Dump();
    Disconnect();
}

/* connect to db */
void Connect(char *dbname)
{
    EXEC SQL BEGIN DECLARE SECTION;
        char connstr[256];
    EXEC SQL END   DECLARE SECTION;

    sprintf(connstr, "%s", dbname);
    EXEC SQL WHENEVER SQLERROR sqlprint;
    EXEC SQL CONNECT TO :connstr;
}

/* disconnect from db */
void Disconnect()
{
    EXEC SQL DISCONNECT;
}

/* dump start here */
void Dump()
{
    EXEC SQL BEGIN DECLARE SECTION;
    char schemaname[256] = {0};
    char tablename[256]  = {0};
    char fullname[256] = {0};
    EXEC SQL END   DECLARE SECTION;

    /*
     * query all tables from pg_tables view
     * FIXME: currently only support postgres
     */
    EXEC SQL DECLARE cur2 CURSOR  FOR
        select schemaname, tablename 
        from pg_tables
        where schemaname!='pg_catalog'
        and schemaname!='information_schema';

    EXEC SQL OPEN cur2;

    while (1)
    {
        EXEC SQL FETCH cur2 INTO :schemaname, :tablename;
        if (sqlca.sqlcode == ECPG_NOT_FOUND) break;
        sprintf(fullname,"%s.%s", schemaname, tablename);
        /* do the acutal dump for this table */
        dump_one_table(fullname);
    }
    EXEC SQL CLOSE cur2;
}

/* dump one table's defination and data */
void dump_one_table(char *tbname)
{
    EXEC SQL BEGIN DECLARE SECTION;
        char sql[256] = {0};
        int rows;
        int cols;
        char data[256] = {0};
        char name[256] = {0};
        int type;
        int firstrow=1;
        int i;
    EXEC SQL END   DECLARE SECTION;

    sprintf(sql, "select * from %s;", tbname);

    EXEC SQL ALLOCATE DESCRIPTOR desc1;
    EXEC SQL PREPARE stmt1 FROM :sql;
    EXEC SQL DECLARE cur1 CURSOR FOR stmt1;
    EXEC SQL OPEN cur1;
    EXEC SQL DESCRIBE stmt1 INTO SQL DESCRIPTOR desc1;
    EXEC SQL GET DESCRIPTOR desc1 :cols = COUNT;

    printf("create table %s (", tbname);

    /* dump table's defination into create stmt
     *      create table xxx (col1 type1, col2, type2...);
     */
    for (i=0; i<cols; i++)
    {
        i++;
        EXEC SQL GET DESCRIPTOR desc1 VALUE :i :name = NAME;
        EXEC SQL GET DESCRIPTOR desc1 VALUE :i :type = TYPE;
        i--;
        if(i==0)
            printf("%s %s", name, get_type_string(type));
        else
            printf(", %s %s", name, get_type_string(type));
    }

    printf(");\n");

    /* dump table's data into a insert stmt
     *      insert into tbname (col1 type ...) values (row1),(row2)..(rown);
     */
    printf("insert into %s (", tbname);
    for (i=0; i<cols; i++)
    {
        i++;
        EXEC SQL GET DESCRIPTOR desc1 VALUE :i :name = NAME;
        i--;
        if(i==0)
            printf("%s", name);
        else
            printf(", %s", name);
    }

    printf(") values ");


    /* print values */
    while(1)
    {
        EXEC SQL FETCH cur1 INTO SQL DESCRIPTOR desc1; 
        if (sqlca.sqlcode == ECPG_NOT_FOUND) break;

        if(firstrow)
            printf("(");
        else
            printf(",(");
        for (i=0; i<cols; i++)
        {
            i++;
            EXEC SQL GET DESCRIPTOR desc1 VALUE :i :data = DATA;
            i--;

            if(i==0)
                printf("\'%s\'", data);
            else
                printf(", \'%s\'", data);
        
        }
        printf(") ");
        firstrow=0;
    }
    printf(";\n");

    EXEC SQL CLOSE cur1;
    EXEC SQL DEALLOCATE DESCRIPTOR desc1;
}

char * get_type_string(int type)
{

    switch(type)
    {
        case 1:
            return "text";
            break;
        case 4:
            return "int";
            break;
        default:
         return "text";
            
    }
}
```

```shell
yshen@yshen-PC:~/coding/esql$ ./dump 
create table public.helloworld (a text);
insert into public.helloworld (a) values ('hello world') ;
create table public.tb1 (a int);
insert into public.tb1 (a) values ('1') ,('1') ,('2') ,('3');
create table public.tb2 (a int, b text);
insert into public.tb2 (a, b) values ('1', 'yshen') ,('2', 'hxwang') ;
```






# 软件架构



## 逻辑架构

![](/uploads/prep.png)

![](/uploads/exec.png)



## 开发架构

### 代码目录

源码位于src/interface/ecpg目录下：

- preproc      --预处理
- ecpglib        --运行时
- pgtypeslib  --类型输入输出
- compatlib   --兼容性
- include        --头文件
- test  --测试



### 主要流程

（回到开头的hello world）


```c
ECPGdo(__LINE__, /* 行号 */
    0,           /* compatible level， 比如兼容infomix */
    1,           /* 强制输出isnull，如果没有提供indicator变量就会报错。 */
    NULL,        /* 用于执行语句的数据库连接 */
    0,           /* 是用?来标志入参么，还是用$1 $2 ...*/
    ECPGst_normal,  /* 语句类型 */
    "select * from helloworld", 
    ECPGt_EOIT,  /* 输入结束标志 */
	ECPGt_char,  /* 数据类型 */
    (msg),       /* 指向变量的指针 */
    (long)256,   /* char或varchar数组的大小 */
    (long)1,     /* 数组中元素的个数（用于取多条记录） */
	(256)*sizeof(char),  /* 数组下一维度的起始位置（用于取多条记录） */ 
	ECPGt_NO_INDICATOR,  /* 有没有indicator来输出isnull */
     NULL ,   /* 指向indicator的指针 */
     0L,      /* */
     0L,      /* indicator的个数（用于取多条记录） */
	 0L,      /* indicator下一维度的其实位置（用于取多条记录） */
     ECPGt_EORT /* 输出结束标志 */
      );
```



# 常见问题

## 语法解析

parser如果用的是esql自己的parser，这样的问题是：1. 服务器的语法esql可能不支持。所以postgres后来改用服务器的parser经过处理之后生成自己的parser。



## 自动提交

默认是手动提交的，也就是需要加上COMMIT语句：

```c
EXEC SQL CREATE TABLE TB1(A INT);
EXEC SQL COMMIT;
```

如果需要自动提交：

```sql
EXEC SQL SET AUTOCOMMIT = ON;
```




## 在esql中用结构体问题

esql中如果用结构体，那么需要把结构体定义放到declare section中。客户的结构体都是定义在头文件中。但是declare section不认头文件中的#ifndef ... define ... endif 这些预处理指令。
解决方案是把结构体定义单独放在一个头文件中，比如struct.h里面。原来的头文件变为：

```c
#ifndef _HEADER_H_
#define _HEADER_H_
#include "struct.h"
#endif
```
esql程序中可以把struct.h加在declare section 中了：

```
EXEC SQL BEGIN DECLARE SECTION;
EXEC SQL INCLUDE struct.h
EXEC SQL END DECLARE SECTION;
```
这样就可以在esql中正常使用结构体了。



## 调试

```sql
#include<stdio.h>
#include<stdlib.h>

int main()
{
    ECPGdebug(1,stdout);

	EXEC SQL BEGIN DECLARE SECTION;
        char msg[256] = {0};
    EXEC SQL END   DECLARE SECTION;

    EXEC SQL CONNECT TO postgres@localhost;
...
    EXEC SQL DISCONNECT;
}
```

```c
yshen@yshen-PC:~/coding/esql$ ./debug 
[11785]: ECPGdebug: set to 1
[11785]: ECPGconnect: opening database postgres on localhost port <DEFAULT>  
[11785]: ecpg_execute on line 15: query: select * from helloworld; with 0 parameter(s) on connection postgres
[11785]: ecpg_execute on line 15: using PQexec
[11785]: ecpg_process_output on line 15: correctly got 1 tuples with 1 fields
[11785]: ecpg_get_data on line 15: RESULT: hello world offset: 256; array: no
hello world
[11785]: ecpg_finish: connection postgres closed
```





## 终极方案

如果遇到ESQL不支持的情况，可以直接调用`ECPGget_PGconn`来返回一个libpq连接，接下来可以用libpq来处理。比如：

```c
    EXEC SQL CONNECT TO testdb AS con1;

    conn = ECPGget_PGconn("con1");

    /* create */
    loid = lo_create(conn, 0);
    if (loid &lt; 0)
        printf("lo_create() failed: %s", PQerrorMessage(conn));
```


如果还想看到更多此类文章，请移步到[小宇的博客](http://shenyu.wiki)。
