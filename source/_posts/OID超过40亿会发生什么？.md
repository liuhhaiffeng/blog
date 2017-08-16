---
title: OID超过40亿会发生什么？
comments: true
date: 2017-08-16 17:04:40
tags: [数据库, PostgreSQL]
categories: [技术分享]
---



> OID是32位无符号int类型，最大取值是2^32=42亿。如果超过了会发生什么呢？

对于加了`with oids`属性的普通用户表，比较消耗oid，因为每行都要生成一个oid。一旦超过42亿，那么会重新从FirstNormalObjectId=16384开始。这就有可能发生表中的两行有重复的现象发生。（表A中两行oid一样，表A和表B两行oid一样。）

对于系统表，都是有`with oids`属性的。那么会不会在一个表中发生重复呢？答案是不会。默认数据库为系统表建了一个OID索引来保证其唯一性的。比如`pg_class_oid_index`。那么插入系统表的一行时，会先用`GetNewObjectId()`新生成一个OID，再往索引里面找，如果找到了那么继续往下找，直到这个新生成的OID在索引里面没有找到，那么就保证了这个OID在该表中是唯一的。但是这种机制只能保证该OID在一个系统表内部是不重复的，不能保证该OID没有在别的系统表或者用户表中用过。


冲突矩阵：
| 会不会冲突     | 全局 | 用户表中行的oid | 系统表A中行的oid |系统表B中行的oid |
| -------------- |:----:|:---------------:|:---------------:|:---:|
| 全局           | 会   | 会              | 会              |会|
| 用户表中行的oid| 会   | 会              | 会              |会|
| 系统表A中行的oid| 会   | 会              | 不会          |会|
| 系统表B中行的oid| 会   | 会              | 会              |不会|


所以假设OID是唯一是不明智的。

内部原理
-----------

![image_1bnl2ggcv158q10s52hbh581eb99.png-122kB][1]


所有的oid都是由`GetNewObjectId()`来生成的，其内部是维护一个全局的counter。
```c
Oid
GetNewObjectId(void)
{
	LWLockAcquire(OidGenLock, LW_EXCLUSIVE);

	if (ShmemVariableCache->nextOid < ((Oid) FirstNormalObjectId))
	{
			/* wraparound in normal environment */
			ShmemVariableCache->nextOid = FirstNormalObjectId;
	}

	result = ShmemVariableCache->nextOid;

	(ShmemVariableCache->nextOid)++;

	LWLockRelease(OidGenLock);

	return result;
}
```
如果超出42亿那么就会重新从FirstNormalObjectId=16384开始。
在`heap_insert()`插入一条数据的时候，会调用`GetNewOid()`来生成一个OID，逻辑如下：
```c
Oid
GetNewOid(Relation relation)
{
	oidIndex = RelationGetOidIndex(relation);

	if (!OidIsValid(oidIndex))
	{
		return GetNewObjectId();
	}

	indexrel = index_open(oidIndex, AccessShareLock);
	newOid = GetNewOidWithIndex(relation, indexrel);
	index_close(indexrel, AccessShareLock);

	return newOid;
}
```
如果有oid索引(一般是系统表)，那么就会调用`GetNewOidWithIndex()`用索引来保证一致性。否则直接调用`GetNewObjectId()`来返回一个OID.

所以以上机制只能保证一个系统表内部的OID不会重复。




  [1]: http://static.zybuluo.com/shenyuflying/2jto93ju1ntvvowq13hypg4o/image_1bnl2ggcv158q10s52hbh581eb99.png


如果还想看到更多此类文章，请移步到[小宇的博客](http://shenyu.wiki)。
