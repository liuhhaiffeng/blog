---
title: linux中的僵尸进程
comments: true
date: 2016-10-09 07:13:11
tags: linux
---

> 如果一个进程已经退出或者被杀死，但是它的父进程尚未执行wait操作，那么该进程进入僵尸(zombie)状态。这种进程不再参与调度，它的内存也会被释放，但系统不会把它从进程表中删除(top 命令中显示状态为Z)。僵尸在进程等待父进程回收它的退出状态。

这篇文章会告诉你：
1. 僵尸进程产生原理。
2. 如何产生僵尸进程。
3. 如何回收僵尸进程。

<!--more-->

## 产生僵尸进程
根据僵尸进程的定义，我们不难用c语言来产生一个僵尸进程

```c
#include<unistd.h>
#include<stdio.h>
#include<stdlib.h>
#include<string.h>

#define MB 1024*1024

int main()
{

    int pid;

    printf("new process PID=%d\n", getpid());
    pid = fork();
    if(pid==0)
    {   
        /* child */
        printf("new process PID=%d\n", getpid());
        char * p = malloc(512*MB);
        memset(p,0,512*MB);
        exit(0);
    }   
    else
    {   
        /* parent */
        sleep(60);
    }   
}      
```
编译并执行该程序，然后再看该程序进程的状态：
```
$ ./main 
new process PID=25810
new process PID=25811

$ ps aux | grep main
yshen    25810  0.0  0.0   4200   620 pts/31   S+   14:30   0:00 ./main
yshen    25811  0.0  0.0      0     0 pts/31   Z+   14:30   0:00 [main] <defunct>

```

可以看到，主进程的状态为S+即休眠状态，子进程的状态为Z+即僵尸状态。此时僵尸进程等待父进程读取其退出状态。在该程序中等待父进程退出之后，子进程也将退出（因为子进程的状态在父进程退出之后已经没什么用了）。

用top命令来查看内存占用情况，可以看到实僵尸进程的内存占用等于0.
```
$ top
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND 
25810 yshen     20   0    4200    792    712 S   0.0  0.0   0:00.00 main
25811 yshen     20   0       0      0      0 Z   0.0  0.0   0:00.67 main
```

## 处理僵尸进程
根据以上分析，我们不难得出僵尸进程退出的2种情况：
1. 父进程用wait/waitpid来获取子进程的状态。
2. 父进程退出。

子进程退出，也会给父进程发送SIGCHLD信号来通知。在实际编码中，一般应SIGCHLD信号进行处理。（如：重新fork拉起子进程或进行垃圾回收，记录日志等等）。如下例子我们注册SIGCHLD信号处理函数，等待子进程退出时由父进程调用wait来获取其退出状态。
```
#include<unistd.h>
#include<stdio.h>
#include<stdlib.h>
#include<signal.h>

void handler(int code)
{
	int status;
	pid_t pid =	wait(&status);
	printf("parent get SIGCHLD, child %d returned %d\n",
									pid, WEXITSTATUS(status));
	sleep(60); /* 加这行是因为父进程收到SIGCHLD信号打断了原来的sleep，如果不加这个父进程就会马上退出 */
}

int main()
{

	int pid;
	signal(SIGCHLD,handler);
	
	printf("new process PID=%d\n", getpid());
	pid = fork();

	if(pid==0)
	{
		/* child */
		printf("new process PID=%d\n", getpid());
		exit(123);
	}
	/* parent */
	sleep(60);
	exit(0);
}
```
编译并执行。
```
$ ./main 
new process PID=26559
new process PID=26560
parent get SIGCHLD, child 26560 returned 123

$ ps aux | grep main
yshen    26604  0.0  0.0   4200   624 pts/31   S+   15:07   0:00 ./main
```
可以看到，已经没有僵尸进程了。


