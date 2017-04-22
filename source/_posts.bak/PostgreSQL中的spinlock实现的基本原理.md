---
title: PostgreSQL中的spinlock实现的基本原理
comments: true
date: 2016-11-13 02:48:32
tags: [linux,c,PostgreSQL]
categories:
---

> linux中锁实现方法有文件锁、mutex、semaphore等。在PostgreSQL设计过程中，考虑到mutex、semaphore具有一定的限制，为了性能和可移植性考虑，对于比较短时间的等待，spinlock性能更好。那么spinlock是怎么实现的呢？

spinlock依赖于test and set，即先测试一个变量的值是否为0，这一步叫test；如果为0则现在没有加锁，那么接下来设置为1加锁，这一步叫set；如果加锁之后，接下来再次加锁test为1则用一个while循环不断等待，从而实现了加锁的功能。
```c
void spin_lock(lock_t *lock)
{
	while(tas(lock)==1);
}
```
解锁相对简单，直接把变量设为0即可。
```c
void spin_unlock(lock_t *lock)
{
	*lock = 0;
}
```

首先需要我们实现一个满足上面要求的tas函数，即函数检查一个内存中的值，看是否为0，如果为0，那么set设置为1，并返回0。如果为1则直接返回1。由于并发原因，必须确保set的原子性，即set的指令必须全部一次执行成功，中间不能出现中断。比如：
要把寄存器al=1的值放入内存(%rcx)，并把内存(%rcx)放入寄存器al其实是一个exchange交换操作，一般需要3条指令，其中用到了一个临时寄存器，比如
```asm
mov (%rcx) ah
mov al     (%rcx)
mov ah     al
```
现在要求如上3条汇编具有原子性，即可以用一条汇编指令来实现，这个操作可以在汇编中用`lock xchg`来实现。

现在我们查看在PostgreSQL中一段加锁的汇编代码：
```asm
...
  40073e:   cmpb   $0x0,(%rcx)  ;test是不是为0
  400741:   jne    400746 <TAS_LOCK+0x20> ;如果不是0，那么直接返回
  400743:   lock xchg %al,(%rcx) ;如果是0，那么设置为1
  400746:   mov    %eax,%ebx ;返回结果
  400748:   movzbl %bl,%eax
  40074b:   pop    %rbx
  40074c:   pop    %rbp
  40074d:   retq
```
按照其原理，通过对其进行反汇编，得到如下的c语言函数：
```c
static __inline__ int
tas(volatile lock_t *lock)
{
    register lock_t _res = 1;

    __asm__ __volatile__(
        "   cmpb    $0,%1   \n"
        "   jne     1f      \n"
        "   lock            \n"
        "   xchgb   %0,%1   \n"
        "1: \n"
:       "+q"(_res), "+m"(*lock)
:       /* no inputs */
:       "memory", "cc");
    return (int) _res;
}
```
下面通过一段并发程序，来检验一下。下面的程序首先在共享内存中创建了一个变量count，接下来fork了两个进程分别对count递增了1000000次，在递增的时候用我们上面描述的spinlock来加锁保护。最后打出了结果，正确的结果是2000000.

<!--more-->
```c
#include<unistd.h>
#include<stdio.h>
#include<stdlib.h>
#include<string.h>
#include<sys/mman.h>
#include<errno.h>
#include<sys/types.h>
#include<sys/wait.h>

typedef unsigned char lock_t;

static __inline__ int
tas(volatile lock_t *lock)
{
    register lock_t _res = 1;

    __asm__ __volatile__(
        "   cmpb    $0,%1   \n"
        "   jne     1f      \n"
        "   lock            \n"
        "   xchgb   %0,%1   \n"
        "1: \n"
:       "+q"(_res), "+m"(*lock)
:       /* no inputs */
:       "memory", "cc");
    return (int) _res;
}

void spin_lock(lock_t *lock)
{
	while(tas(lock)==1);
}
void spin_unlock(lock_t *lock)
{
	*lock = 0;
}

void *shmalloc(ssize_t size)
{
	void *mem = (void *)mmap(NULL,size,PROT_READ|PROT_WRITE,MAP_SHARED|MAP_ANONYMOUS,-1,0);

	if (mem == (void *)-1)
	{
		printf("mmap error: %s\n",strerror(errno));
		exit(1);
	}

}

int main()
{
	int *count = (int *)shmalloc(sizeof(int));
	lock_t *lock = (lock_t *)shmalloc(sizeof(lock_t));
	*lock = 0;

	*count = 0;

	pid_t pid = fork();

	if (pid < 0)
	{
		printf("fork failed!\n");
		exit(1);
	}

	if (pid == 0)
	{
		/* child */
		int i;
		for (i=0; i<1000000; i++)
		{
			spin_lock(lock);
				(*count)++;
			spin_unlock(lock);
		}
		exit(1);
	}

	if (pid > 0)
	{	
		/* parent */
		int i;
		for (i=0; i<1000000; i++)
		{
			spin_lock(lock);
				(*count)++;
			spin_unlock(lock);
		}
		wait(NULL);
		printf("%d\n",*count);
		exit(1);
	}
}
```

```
 $ ./tas 
2000000
```
