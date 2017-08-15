---
title: postgres keepalive
comments: true
date: 2017-08-15 16:15:18
tags: [技术分享, 数据库, PostgreSQL]
categories: 技术分享
---

> 在客户端连接服务器后，由于各种异常情况，如客户端出现断网、死机、系统崩溃等情况，服务器可能并不知道对端的情况而一直维护这个连接，造成资源的浪费，因此设置了keepalive后，服务器可以探知对端发生的意外情况而释放这个连接。

参数说明
---------

有三个参数可以配置：

| 参数                  | 说明           |  单位           | 
| ---------------------- |:-------------:|:---------------:|
| tcp_keepalives_idle      |  最后一次数据交换到发送第一个探测包的间隔 |  秒|
| tcp_keepalives_interval    |  没有接收到对方确认，继续发送探测包的次数         |  次|
| tcp_keepalives_count     | 没有接收到对方确认，继续发送探测包的频率    | 秒|




即在经过最后一次数据交换后，服务器会等待tcp_keepalive_idle秒的时间，然后发送第一个探测包，若没有收到确认就每隔tcp_keepalive_intvl秒发送下一个，当第tcp_keepalive_probes个探测包没有收到确认后，服务器就会释放这个连接


配置方法
------------
在服务器的配置文件中加入三行：
```
tcp_keepalives_idle = 180
tcp_keepalives_interval = 30
tcp_keepalives_count = 3
```   
在上例中，服务器先等待180秒后会每隔30秒发送一个探测包，若三个探测包均无确认，服务器会在总计240秒后断开连接。

**注意**：以上配置需要重启服务器。

接下来登陆山
```
$ ./isql -USYSTEM -WMANAGER -h192.168.2.121 -p61123 TEST
TEST=# show tcp_keepalives_idle;
 tcp_keepalives_idle 
---------------------
 180
(1 行)

TEST=# show tcp_keepalives_count;
 tcp_keepalives_count 
----------------------
 3
(1 行)

TEST=# show tcp_keepalives_interval;
 tcp_keepalives_interval 
-------------------------
 30
(1 行)

```

内部实现
------------
keepalive是通过`pq_setkeepalive`函数来设置的，该函数的调用关系如下图。
![image_1bniabaj2bp1ms11ni33pc43s9.png-26.2kB][1]
可以看到是在连接ConnCreate的时候来设置的。具体的看`pg_setkeepalivesidle`可以看到：
```c
int
pq_setkeepalivesidle(int idle, Port *port)
{
	if (port == NULL || IS_AF_UNIX(port->laddr.addr.ss_family))
		return STATUS_OK;

	if (idle == port->keepalives_idle)
		return STATUS_OK;

	if (setsockopt(port->sock, IPPROTO_TCP, TCP_KEEPIDLE,
				   (char *) &idle, sizeof(idle)) < 0)
	{
		elog(LOG, "setsockopt(TCP_KEEPIDLE) failed: %m");
		return STATUS_ERROR;
	}

	port->keepalives_idle = idle;
	return STATUS_OK;
}
```
1. 首先会判断是不是空端口（比如本地fork出来的进程），和是不是本地的端口（IS_AF_UNIX）。如果是这两种情况就不会设。
2. 如果已经设上那么直接返回。
3. 接下来用`setsockopt`来设置。

**注意**：以上代码是简化版本。

其他说明
-----------
有反应show keepalive出来的值是0. show是利用getsockopt来获取端口实际的值。如果连接是本机的话，其实是没设上idle interval count，所以show出来是0. 连接是远程的话，才能设上，show也能显示出来。

根据如上代码分析：本机的连接不需要设keepalive，只有远程的连接才会设。
show的时候是用getsockopt来获取当前连接的keepalive值。所以本机的getsockopt获取的是0。其实是没设上。远程的正常。


  [1]: http://static.zybuluo.com/shenyuflying/b0a8768gu9v61tsvc83lsv2f/image_1bniabaj2bp1ms11ni33pc43s9.png

如果还想看到更多此类文章，请移步到[小宇的博客](http://shenyu.wiki)。
