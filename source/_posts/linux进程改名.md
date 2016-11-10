---
title: linux进程改名
comments: true
date: 2016-11-10 08:16:26
tags: [linux,c]
---

> 在多进程程序中，用ps命令能看到进程的名字。这样能够方便管理，不会因为看到好多同样的进程而不知道他们在干什么而茫然，同时也能避免管理员kill掉错误的进程。

比如PostgreSQL刚启动时共有7个进程，通过ps可以清楚的看到每个进程是干什么的。

```
$ ps ax | grep postgres
26300 pts/26   S      0:00 ./postgres -D ../data
26301 ?        Ss     0:00 postgres: logger process  
26303 ?        Ss     0:00 postgres: checkpointer process  
26304 ?        Ss     0:00 postgres: writer process  
26305 ?        Ss     0:00 postgres: wal writer process  
26306 ?        Ss     0:00 postgres: autovacuum launcher process  
26307 ?        Ss     0:00 postgres: stats collector process  

```
那么在linux下是如何做到进程改名的呢？

方法1--修改argv[0]
------


```c
#include<unistd.h>                                                                             #include<stdlib.h>
#include<stdio.h>
#include<string.h>

int main(int argc, char *argv[])
{
    memcpy(argv[0],"hello world",sizeof(char)*strlen(argv[0]));
    sleep(100);
    return 0;
}

```

```
yshen@yshen-office:~/test/procname$ ./main &
[2] 26903
yshen@yshen-office:~/test/procname$ ps
  PID TTY          TIME CMD
26903 pts/14   00:00:00 main
yshen@yshen-office:~/test/procname$ ps ax | grep hello
26903 pts/14   S      0:00 hello 
```
问题：ps不起效果，ps ax才能看到。不知到为啥。
其实PostgreSQL中就是用的这种方法，详见ps_status.c这个文件。

方法2--用prctl
---------
需要Linux 2.1.57版本以上。

```c
#include<unistd.h>
#include<stdlib.h>
#include<stdio.h>
#include<sys/prctl.h>
#include<string.h>
#include<errno.h>
#include<sys/types.h>
#include<sys/wait.h>


char *progname = NULL;

void setproctitle(char *title)
{
	char buf[256];

	sprintf(buf,"%s: %s", progname, title);
	if (0 != prctl(PR_SET_NAME,buf,0,0,0,0))
		printf("error prctl, errno=%d, errstr=%s\n", errno, strerror(errno));
}
char *getproctitle(void)
{
	char *ret = (char *)malloc(sizeof(char)*16);
	if (0 != prctl(PR_GET_NAME,ret,0,0,0,0))
	{
		printf("error prctl, errno=%d, errstr=%s\n", errno, strerror(errno));
		free(ret);
		ret = NULL;
		
	}
	return ret;

}
int main(int argc, char *argv[])
{
	pid_t pid;
	progname = strdup(argv[0]);
	
	switch(pid = fork())
	{
		case 0:
		{
			/* child */
			setproctitle("child");
			sleep(100);
			break;
		}
		case -1:
		{
			printf("fork falied!\n");
			exit(1);
			break;
		}
		default:
		{
			/* parent */
			setproctitle("parent");
			sleep(100);
			wait(NULL);
			break;
		}
	}

	return 0;


}
```

```
$ ./main  &
[1] 26917
$ ps
  PID TTY          TIME CMD
16399 pts/14   00:00:00 bash
25668 pts/14   00:00:02 gedit
26917 pts/14   00:00:00 ./main2: parent
26918 pts/14   00:00:00 ./main2: child
$ ps ax | grep main
26917 pts/14   S      0:00 ./main
26918 pts/14   S      0:00 ./main

```


问题：ps ax不起效果，ps 看到。不知到为啥。

看来综合用这两种方法才行啊。

