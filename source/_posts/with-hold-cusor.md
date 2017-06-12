---
title: with hold cusor
comments: true
date: 2017-06-12 11:42:26
tags: [数据库, PostgreSQL接口, PostgreSQL, ECPG]
categories: [技术分享]
---


> 最近在协助光大银行研发国产化平台，遇到了一个问题：客户这边需要fetch一条数据，然后根据取到的数据做update等等（不一定是对这刚取到的数据进行update），然后再fetch下一条，然后下一条取不出来。经过看客户这边的代码是因为多个fetch操作之间有update语句进行commit操作。commit会把cursor删掉，之后的fetch无法使用cursor。这时候with hold cursor就派上用场了。


重现用例：
```
#include<stdio.h>
#include<stdlib.h>

int main()
{
	EXEC SQL BEGIN DECLARE SECTION;
		int dboid;
		char dbname[256] = {0};
	EXEC SQL END   DECLARE SECTION;

	EXEC SQL CONNECT TO postgres USER yshen;

	EXEC SQL PREPARE stmt1 FROM "SELECT oid,datname FROM pg_database WHERE oid > ?";
	EXEC SQL DECLARE foo_bar CURSOR FOR stmt1;

	/* when end of result set reached, break out of while loop */
	EXEC SQL WHENEVER NOT FOUND DO BREAK;
	EXEC SQL WHENEVER SQLERROR STOP;

	EXEC SQL OPEN foo_bar USING 100;
	while (1)
	{
		    EXEC SQL FETCH NEXT FROM foo_bar INTO :dboid, :dbname;
			printf("%d %s\n", dboid, dbname);
		    EXEC SQL COMMIT;
	}
	EXEC SQL CLOSE foo_bar;
	EXEC SQL DEALLOCATE PREPARE stmt1;
}
```

这是数据库日志：

```
LOG:  statement: begin transaction
LOG:  execute <unnamed>: declare foo_bar cursor for SELECT oid,datname FROM sys_database WHERE oid > $1
DETAIL:  parameters: $1 = '100'
LOG:  statement: fetch next from foo_bar
LOG:  statement: commit
LOG:  statement: begin transaction
LOG:  statement: fetch next from foo_bar
ERROR:  cursor "foo_bar" does not exist
STATEMENT:  fetch next from foo_bar
```

可以看到，commit会删掉定义的cursor，所以服务器报`ERROR:  cursor "foo_bar" does not exist`错误。

加上WITH HOLD 关键字即可解决这个问题：

EXEC SQL DECLARE foo_bar CURSOR `WITH HOLD` FOR stmt1;

```
LOG:  statement: begin transaction
LOG:  execute <unnamed>: declare foo_bar cursor with hold for SELECT oid,datname FROM pg_database WHERE oid > $1
DETAIL:  parameters: $1 = '100'
LOG:  statement: fetch next from foo_bar
LOG:  statement: commit
LOG:  statement: begin transaction
LOG:  statement: fetch next from foo_bar
LOG:  statement: commit
LOG:  statement: begin transaction
LOG:  statement: fetch next from foo_bar
LOG:  statement: close foo_bar
LOG:  statement: deallocate "stmt1"
LOG:  statement: commit
```

可以看到数据是成功取出来了，服务器并没有报错。





如果还想看到更多此类文章，请移步到[小宇的博客](http://shenyu.wiki)。
