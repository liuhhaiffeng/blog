---
title: 【小宇带你学PostgreSQL内核】第二课：开发环境
comments: true
date: 2016-12-11 10:23:14
tags:
categories:
---

<center><iframe height=498 width=510 src='http://player.youku.com/embed/XMTg1OTI5NTQ4NA==' frameborder=0 'allowfullscreen'></iframe></center>

<!--more-->

2. 开发环境
-----------------
### 2.1 编译环境
linux
make  3.80
gcc/llvm-clang
Flex 2.5.31 
Bison 1.875 
Perl 5.8
libreadline
zlib
git
lcov
dtrace
### 2.2 获取代码
历史版本：wget https://ftp.postgresql.org/pub/source/vx.x.x/postgresql-x.x.x.tar.gz
git clone 	https://git.postgresql.org/git/postgresql.git
### 2.3 编辑器

**vim**
**emacs**
**SourceInsight**

### 2.4 编译

```
./configure \
CC='gcc -O0 -gdwarf-2 -g3 ' \
--prefix=`pwd`/release \
--enable-debug \
--enable-coverage \
--enable-profiling \
--enable-cassert \
--enable-depend \
--enable-dtrace
```

```
$ make
$ make install
$ ./initdb -D ../data
$ ./postgres -D ../data
$ ./psql  postgres
psql (9.6.0)
Type "help" for help.
postgres=# 
```

### 2.5 调试
terminator：一个很好用的终端
gdb
ddd





如果还想看到更多此类文章，请移步到[小宇的博客](http://shenyu.wiki)。
