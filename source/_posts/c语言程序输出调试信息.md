---
title: c语言程序输出调试信息
comments: true
date: 2016-11-17 13:51:58
tags: [linux,c,PostgreSQL,debug]
categories:
---

> 随着服务器的内存逐渐增大，从16GB、32GB逐步拓展到64GB、128GB甚至256GB或更多。数据库在遇到一些致命错误（比如SIGSEGV段错误、stack overflow）时，受限于磁盘空间大小、IO速度、恢复时间等限制，往往不具备条件把内存映像（coredump）转储到磁盘上，这为后续排查出错原因带来了很大困难。这时候就需要程序在遇到致命错误时，能够及时输出诊断信息。一些语言的运行时，比如java、golang等内建了错误堆栈输出的功能。那么c语言程序如何实现错误堆栈输出的功能呢？


先睹为快
---------

启动程序，然后发送SIGSEGV信号来模拟程序遇到非法内存访问错误的情形。
```
$ ./main &
[new process 2018]
$ kill -11 2018
#0  0x00007f30d7487680 in __read_nocancel () at ../sysdeps/unix/syscall-template.S:84
#1  0x0000000000400b6b in system_ex ()
#2  0x0000000000400c87 in run ()
#3  0x0000000000400dcb in run_gdb ()
#4  0x0000000000400f37 in GetCrashInfo ()
#5  <signal handler called>
#6  0x0000000000400f9a in main ()
```
我们看到了函数的堆栈，进一步还可以打出诸如本地变量，局部变量，寄存器等等信息。
<!--more-->

基本思路
--------
运行中的程序输出堆栈的方法有2种。
第一种：借助backtrace函数。
第二种：借助gdb。

### backtrace函数

backtrace用法比较简单，但是功能也比较有限，只能输出基本的堆栈。
比如：
```
$ cc -rdynamic prog.c -o prog
$ ./prog 3
backtrace() returned 8 addresses
./prog(myfunc3+0x5c) [0x80487f0]
./prog [0x8048871]
./prog(myfunc+0x21) [0x8048894]
./prog(myfunc+0x1a) [0x804888d]
./prog(myfunc+0x1a) [0x804888d]
./prog(main+0x65) [0x80488fb]
/lib/libc.so.6(__libc_start_main+0xdc) [0xb7e38f9c]
./prog [0x8048711]

```
```c
#include <execinfo.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

void
myfunc3(void)
{
    int j, nptrs;
#define SIZE 100
    void *buffer[100];
    char **strings;

    nptrs = backtrace(buffer, SIZE);
    printf("backtrace() returned %d addresses\n", nptrs);

    /* The call backtrace_symbols_fd(buffer, nptrs, STDOUT_FILENO)
       would produce similar output to the following: */

    strings = backtrace_symbols(buffer, nptrs);
    if (strings == NULL) {
        perror("backtrace_symbols");
        exit(EXIT_FAILURE);
    }

    for (j = 0; j < nptrs; j++)
        printf("%s\n", strings[j]);

    free(strings);
}
static void   /* "static" means don't export the symbol... */
myfunc2(void)
{
    myfunc3();
}

void
myfunc(int ncalls)
{
    if (ncalls > 1)
        myfunc(ncalls - 1);
    else
        myfunc2();
}

int
main(int argc, char *argv[])
{
    if (argc != 2) {
        fprintf(stderr, "%s num-calls\n", argv[0]);
        exit(EXIT_FAILURE);
    }

    myfunc(atoi(argv[1]));
    exit(EXIT_SUCCESS);
}


```

### 借助gdb

gdb是专为调试诞生的工具，更为强大。但是如何在程序中调用gdb？系统已经提供了system调用，但是该调用不能把程序的信息输出给调用者。需要我们自己写一个system调用，能够把子进程输出的信息发送给主进程。
基本思路是fork一个进程，在子进程execv运行gdb，attach到主进程，运行一系列调试命令来输出调试信息。把子进程输出的信息发送给主进程的方法，涉及到了多线程通信的问题。一个比较简单的解决方法是在主进程和子进程之间建立管道，在子进程里面把标准输出重定向到管道的写入端，主进程从管道读取信息。为此专门写了一个`system_ex`函数来完成这个工作。
```c
/*
 * system_ex -- system with something extra
 *
 * like original system but can save the
 * cmd output value to buf. the memory
 * used by buf is allocated by system_ex
 * using malloc and realloc, make sure
 * you free the buf memory after the
 * function returns.
 *
 *                       yshen@2016
 * Argument:
 *       cmd: the command as a null terminated
 *       string to execute
 *       buf: the buffer to save the stdout
 *       printed by the command
 * Return:
 *       on sucess system_ex returns the
 *       command exit code
 *       on faliure system_ex returns -1
 *       and buf set to NULL
 * Usage:
 *       char *buf = NULL;
 *       if (system_ex("ls",&buf) != -1)
 *       {
 *           printf("%s", buf == NULL ? "(no output)" : buf);
 *           if (buf != NULL)
 *               free(buf);
 *       }
 *       else
 *       {
 *           ...error handling...
 *       }
 */
static int
system_ex(char* cmd, char **buf)
{

	int   fd[2];
	pid_t pid;
	int   n, count;
	int wait_val = 0;

	if (cmd == NULL)
		return -1;

	/* no need to save to buf, use original system() */
	if (buf == NULL)
		return system(cmd);

	/*
	 *         write                    read
	 * child ---------> fd[1] fd[0] ------------> parent
	 *
	 */
	if (pipe(fd) < 0)
	{
		*buf = NULL;
		return -1;
	}

	if ((pid = fork()) < 0)
	{
		/* fork failed, close pipe*/
		close(fd[0]);
		close(fd[1]);
		*buf = NULL;
		return -1;
	}
	else if (pid > 0)
	{
		/* parent */
		int allocsz = 1024;
		close(fd[1]);
		count = 0;
		/* 1k shall be enough for most case */
		*buf = (char *)malloc(sizeof(char)*allocsz);
		/* not enough memory */
		if (*buf == NULL)
			return -1;

		/* parent block here waiting child to write */
		while (n = read(fd[0], (*buf) + count, 1024))
		{
			count += n;
			if (count >= allocsz)
			{
				/* double the buffer each time */
				allocsz *= 2;
				*buf = (char *)realloc(*buf, allocsz);

				/* not enough memory */
				if (*buf == NULL)
					return -1;
			}
		}

		/*terminate the string */
		if (count > 0)
			(*buf)[count - 1] = '\0';
		else
		{
			/* opps, the child exit abnormally, nothing read */
			free(*buf);
			*buf = NULL;
		}

		close(fd[0]);
		/* get the return value of command */
		if (wait4(pid, &wait_val, 0, 0) == -1)
		{
			wait_val = -1;
		}
		return wait_val;
	}
	else
	{
		/* child */
		/* close read side */
		close(fd[0]);

		/* close stdout */
		close(STDOUT_FILENO);
		/* dup fd1 now fd1 equals to stdout */
		dup(fd[1]);
		/* close it again */
		close(fd[1]);
		/* execute the command */
        execl("/bin/sh", "sh", "-c", cmd, (char*)0);
		/* should not reach here */
        exit(127);
	}

	/* should not reach here */
	*buf = NULL;

	return -1;
}
```
好了准备工作完毕。下面从`main`函数开始讲解。
第一步在程序入口注册信号处理函数，在程序发生非法内存访问，会收到SIGSEGV信号，这时候调用GetCrashInfo。
```c
int main()
{

	signal(SIGSEGV, GetCrashInfo);

	while(1)；

}
```
接下来`GetCrashInfo`调用`run_gdb`来完成详细调试信息的输出。
```c
/*
 * MAIN entrance of Crash Info Collector
 * run a batch of gdb command to collect
 * neccessary infomation useful for debug
 */
void
GetCrashInfo(int a)
{
	int  level;
	int  i;
	
	printf("get singal %d\n", a);

	run_gdb("bt", -1);
	run_gdb("thread apply all bt", -1);

	level = GetStackLevel();

	/* general infos */
	for (i = 0; i < level ; i++)
	{
		run_gdb("info args", i);
		run_gdb("info locals", i);
	}
	/* detailed asm info */
	for (i = 0; i < level; i++)
	{
		run_gdb("info frame", i);
		run_gdb("info registers", i);
		run_gdb("disas", i);
	}

	exit(1);
}
```

`run_gdb`调用gdb来完成入参指定的命令。方法是拼了一个命令串`cmd_str`，包含要执行的gdb命令，pid等信息。接下来调用`run`来执行这个命令并返回结果。

```c
/*
 * run a gdb command under frame n
 * the output of gdb is printed using ereport LOG
 * Arguments:
 *      cmd: the command to run by gdb, such as "bt"
 *      frame: under which frame to run the command
 *             values from 0-N, any value less than 0
 *             means no need to go to that frame
 */
static void run_gdb(char *cmd, int frame)
{
	char cmd_str[1024] = {0};
	char arg_f[256] = {0};
	char *buf = NULL;

#define IGNORE_STRING 	"| grep -v \"\\[New Thread\" "\
						"| grep -v \"No symbol table info available\" "\
						"| grep -v \"in read () from\" "\
						"| grep -v \"\\[Thread debugging using\" "\

	if (frame >= 0)
		sprintf(arg_f,"-ex \"f %d\"",frame);
	else
		sprintf(arg_f," ");

	sprintf(cmd_str,"gdb -batch "
				"-ex \" set pagination 0 \" "
				"%s " /* -ex f ?*/
				"-ex \"%s\" "
				"-p %d "
				IGNORE_STRING,
				arg_f,
				cmd,
				getpid());

	buf = run(cmd_str);
	if (buf != NULL)
	{
		printf("\n=====================================================================\n"
			"\n            frame:%d   command:%s                                    \n"
			"\n---------------------------------------------------------------------\n",
			frame < 0 ? 0 : frame, cmd);
		printf("%s", buf);
		free(buf);
	}
}

```
`run`函数调用了上述的`system_ex`来实际执行命令并返回结果。对错误进行了简单的处理。
```c
/*
 * run a command and return the output
 * if the command failed report the
 * error message using ereport DEBUG1
 */
static char *
run(char *cmd)
{

    char *buf = NULL;
    int status;



    if ((status = system_ex(cmd, &buf)) == 0)
    {
        if (buf != NULL)
				return buf;
    }
	else
	{
		printf("command %s failed with status %d msg %s",
			cmd, WEXITSTATUS(status), buf == NULL ? "no" : buf);
	}

	return buf;
}
```



完整的源码
------------
```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/time.h>
#include <sys/resource.h>
#include <sys/wait.h>
#include <signal.h>



static int system_ex(char* cmd, char **buf);
static char *run(char *cmd);
static void run_gdb(char *cmd, int frame);
static int GetStackLevel(void);
void GetCrashInfo(int);

/*
 * system_ex -- system with something extra
 *
 * like original system but can save the
 * cmd output value to buf. the memory
 * used by buf is allocated by system_ex
 * using malloc and realloc, make sure
 * you free the buf memory after the
 * function returns.
 *
 *                       yshen@2016
 * Argument:
 *       cmd: the command as a null terminated
 *       string to execute
 *       buf: the buffer to save the stdout
 *       printed by the command
 * Return:
 *       on sucess system_ex returns the
 *       command exit code
 *       on faliure system_ex returns -1
 *       and buf set to NULL
 * Usage:
 *       char *buf = NULL;
 *       if (system_ex("ls",&buf) != -1)
 *       {
 *           printf("%s", buf == NULL ? "(no output)" : buf);
 *           if (buf != NULL)
 *               free(buf);
 *       }
 *       else
 *       {
 *           ...error handling...
 *       }
 */
static int
system_ex(char* cmd, char **buf)
{

	int   fd[2];
	pid_t pid;
	int   n, count;
	int wait_val = 0;

	if (cmd == NULL)
		return -1;

	/* no need to save to buf, use original system() */
	if (buf == NULL)
		return system(cmd);

	/*
	 *         write                    read
	 * child ---------> fd[1] fd[0] ------------> parent
	 *
	 */
	if (pipe(fd) < 0)
	{
		*buf = NULL;
		return -1;
	}

	if ((pid = fork()) < 0)
	{
		/* fork failed, close pipe*/
		close(fd[0]);
		close(fd[1]);
		*buf = NULL;
		return -1;
	}
	else if (pid > 0)
	{
		/* parent */
		int allocsz = 1024;
		close(fd[1]);
		count = 0;
		/* 1k shall be enough for most case */
		*buf = (char *)malloc(sizeof(char)*allocsz);
		/* not enough memory */
		if (*buf == NULL)
			return -1;

		/* parent block here waiting child to write */
		while (n = read(fd[0], (*buf) + count, 1024))
		{
			count += n;
			if (count >= allocsz)
			{
				/* double the buffer each time */
				allocsz *= 2;
				*buf = (char *)realloc(*buf, allocsz);

				/* not enough memory */
				if (*buf == NULL)
					return -1;
			}
		}

		/*terminate the string */
		if (count > 0)
			(*buf)[count - 1] = '\0';
		else
		{
			/* opps, the child exit abnormally, nothing read */
			free(*buf);
			*buf = NULL;
		}

		close(fd[0]);
		/* get the return value of command */
		if (wait4(pid, &wait_val, 0, 0) == -1)
		{
			wait_val = -1;
		}
		return wait_val;
	}
	else
	{
		/* child */
		/* close read side */
		close(fd[0]);

		/* close stdout */
		close(STDOUT_FILENO);
		/* dup fd1 now fd1 equals to stdout */
		dup(fd[1]);
		/* close it again */
		close(fd[1]);
		/* execute the command */
        execl("/bin/sh", "sh", "-c", cmd, (char*)0);
		/* should not reach here */
        exit(127);
	}

	/* should not reach here */
	*buf = NULL;

	return -1;
}

/*
 * run a command and return the output
 * if the command failed report the
 * error message using ereport DEBUG1
 */
static char *
run(char *cmd)
{

    char *buf = NULL;
    int status;



    if ((status = system_ex(cmd, &buf)) == 0)
    {
        if (buf != NULL)
				return buf;
    }
	else
	{
		printf("command %s failed with status %d msg %s",
			cmd, WEXITSTATUS(status), buf == NULL ? "no" : buf);
	}

	return buf;
}

/*
 * run a gdb command under frame n
 * the output of gdb is printed using ereport LOG
 * Arguments:
 *      cmd: the command to run by gdb, such as "bt"
 *      frame: under which frame to run the command
 *             values from 0-N, any value less than 0
 *             means no need to go to that frame
 */
static void run_gdb(char *cmd, int frame)
{
	char cmd_str[1024] = {0};
	char arg_f[256] = {0};
	char *buf = NULL;

#define IGNORE_STRING 	"| grep -v \"\\[New Thread\" "\
						"| grep -v \"No symbol table info available\" "\
						"| grep -v \"in read () from\" "\
						"| grep -v \"\\[Thread debugging using\" "\

	if (frame >= 0)
		sprintf(arg_f,"-ex \"f %d\"",frame);
	else
		sprintf(arg_f," ");

	sprintf(cmd_str,"gdb -batch "
				"-ex \" set pagination 0 \" "
				"%s " /* -ex f ?*/
				"-ex \"%s\" "
				"-p %d "
				IGNORE_STRING,
				arg_f,
				cmd,
				getpid());

	buf = run(cmd_str);
	if (buf != NULL)
	{
		printf("\n=====================================================================\n"
			"\n            frame:%d   command:%s                                    \n"
			"\n---------------------------------------------------------------------\n",
			frame < 0 ? 0 : frame, cmd);
		printf("%s", buf);
		free(buf);
	}
}

static int
GetStackLevel(void)
{
	char cmd[1024] = {0};
	int  level;
	char *buf = NULL;

	sprintf(cmd,"gdb -batch "
				"-ex \"set pagination 0\" "
				"-ex \"bt\" "
				"-p %d"
				"| grep \"#\" "
				"| wc -l",
				getpid());
	buf = run(cmd);
	if (buf != NULL)
	{
		level = atoi(buf);
		free(buf);
		return level;
	}

	return -1;
}

/*
 * MAIN entrance of Crash Info Collector
 * run a batch of gdb command to collect
 * neccessary infomation useful for debug
 */
void
GetCrashInfo(int a)
{
	int  level;
	int  i;
	
	printf("get singal %d\n", a);

	run_gdb("bt", -1);
	run_gdb("thread apply all bt", -1);

	level = GetStackLevel();

	/* general infos */
	for (i = 0; i < level ; i++)
	{
		run_gdb("info args", i);
		run_gdb("info locals", i);
	}
	/* detailed asm info */
	for (i = 0; i < level; i++)
	{
		run_gdb("info frame", i);
		run_gdb("info registers", i);
		run_gdb("disas", i);
	}

	exit(1);
}

int main()
{

	signal(SIGSEGV, GetCrashInfo);

	while(1)；

}
```
