---
title: linux中进程共享内存的2种方法
comments: true
date: 2016-11-12 11:27:10
tags: [c,linux,PostgreSQL]
---

> 共享内存可以在两个或多个进程间共享一个给定的内存区域，因为数据不需要在进程之间复制，相比于pipe、socket、file等共享通信方式，共享内存是最快的一种共享机制。linux中共享内存一般有2种方法即:shared memory和mmap。


下面两个例子都是首先在主进程里创建一段共享内存，然后通过fork创建一个子进程。子进程在共享内存里写入Hello world，父进程等待子进程退出之后读取共享内存中的内容并显示出来。
下面通过两个实例源码来介绍。

<!--more-->

shmat
------

```
#include<unistd.h>
#include<stdio.h>
#include<stdlib.h>
#include<string.h>
#include<errno.h>
#include<sys/types.h>
#include<sys/wait.h>
#include<sys/ipc.h>
#include<sys/shm.h>

int main()
{
	int shmid = shmget(IPC_PRIVATE,100,IPC_CREAT|IPC_EXCL|0600);
	char *mem = (char *)shmat(shmid,0,0);
	
	if (shmid == -1)
	{
		printf("shmget error:%s\n",strerror(errno));
		exit(1);
	}

	pid_t pid = fork();

	if (pid < 0)
	{
		printf("fork failed!\n");
		exit(1);
	}

	if (pid == 0)
	{
		char *str = "Hello world!\n";
		memcpy(mem,str,strlen(str)+1);
		exit(1);
	}

	if (pid > 0)
	{	
		wait(NULL);
		printf("%s",mem);
		exit(1);
	}
}
```

```
$ ./shm
Hello world!
```

mmap
--------




```c
#include<unistd.h>
#include<stdio.h>
#include<stdlib.h>
#include<string.h>
#include<sys/mman.h>
#include<errno.h>
#include<sys/types.h>
#include<sys/wait.h>

int main()
{
	char *mem = (char *)mmap(NULL,sizeof(char)*100,PROT_READ|PROT_WRITE,MAP_SHARED|MAP_ANONYMOUS,-1,0);

	if ((void *)mem == (void *)-1)
	{
		printf("mmap error: %s\n",strerror(errno));
		exit(1);
	}

	pid_t pid = fork();

	if (pid < 0)
	{
		printf("fork failed!\n");
		exit(1);
	}

	if (pid == 0)
	{
		char *str = "Hello world!\n";
		memcpy(mem,str,strlen(str)+1);
		exit(1);
	}

	if (pid > 0)
	{	
		wait(NULL);
		printf("%s",mem);
		exit(1);
	}
}
```

```
$ ./mmap
Hello world!

```
