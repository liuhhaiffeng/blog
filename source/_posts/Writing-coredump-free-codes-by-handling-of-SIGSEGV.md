---
title: Writing coredump free codes by handling of SIGSEGV
comments: true
date: 2017-10-29 09:15:57
tags: [c]
categories: [技术分享]
---


> Invalied memory access is always a headache for C programmers. The default way for the OS to handle Invalied memory access is send a SIGSEGV to the program and later shutdown the program and generate a coredump file. But very often we want to see if a memory address is safe to write and read. Or simply sometime we just want to avoid SIGSEGV to happen. The trick is to catch SIGSEGV and handle it by ourself.

The below C code is trying to access an invalied memory address `0x00`, the OS will send SIGSEGV to the program and generate a coredump file, finally the program is shut down. This is the default behavior for the OS to handle invalied memory access.
```c
    int *a = 0;
    *a = 1;
```
But suppose we put the above code inside a try...catch? If the try failed, we will catch it and print an error message?
```c
    try
    {
        int *a = 0;
        *a = 1;
    }
    catch
    {
        printf("catched SIGSEGV\n");
    }
    endtry;
```

The `try...catch` here set up two things: 
1. a signal handler to catch `SIGSEGV`.
2. a long jump to catch when `SIGSEGV` is raised.

Here is a full source code:
```
#include <stdlib.h>
#include <stdio.h>
#include <setjmp.h>
#include <signal.h>

typedef void (*SignalHandlerPtr)(int);
SignalHandlerPtr savedSIGSEGV = (SignalHandlerPtr)(0);
sigjmp_buf SEGSEGVjmp;
static void SEGSEGVhandler(int sig);

#define try  \
	do { \
	    savedSIGSEGV = signal(SIGSEGV, SEGSEGVhandler); \
		if (sigsetjmp(SEGSEGVjmp, 1) == 0) \
		{ 

#define catch	\
		} \
		else \
		{
		 

#define endtry  \
        signal(SIGSEGV, savedSIGSEGV); \
		} \
	} while (0)

static void SEGSEGVhandler(int sig)
{
    sig=sig;
    siglongjmp(SEGSEGVjmp, 1);
}

int main()
{
    try
    {
        int *a = 0;
        *a = 1;
    }
    catch
    {
        printf("catched SIGSEGV\n");
    }
    endtry;

    
	printf("exited normally\n");
    return 0;
}
```




This program will not cause a coredump, instead will catch SIGSEGV and exit normally.
```c
$ ./main 
catched SIGSEGV
exited normally
```

If we know how to try...catch a SIGSEGV, we may as well know how to write a function to test whether an address is safe or not. Now we write a function called `bool IsByteSafe( void *add )` for that.


```c
bool IsByteSafe( void *add )
{
	unsigned char c;

	/* is safe to read/write ? */
    try
    {
		c = *(unsigned char *)add;
		*(unsigned char *)add = c;
    }
    catch
    {
		return false;
    }
    endtry;

	return true;
}


```

More, we can also test a memory region by using `bool IsByteSafe( void *add )`. So we come up with a function `bool IsSafeAddress(void *begin, size_t len)`.
```c
bool IsSafeAddress(void *begin, size_t len)
{
	while(--len)
	{
		if(!IsByteSafe(begin+len))
			return false;
	}

	return true;
}
```

Now, let's put all above into test.
```
int main()
{
	int c;
	int *a = &c;
	int *b = 0x00;
#define SIZE (100*sizeof(int))
	int *d = (int *)malloc(SIZE);

	if(IsSafeAddress(a, sizeof(int)))
	{
		printf("%p is safe\n", a);
	}
	else
	{
		printf("%p is not safe\n", a);
	}


	if(IsSafeAddress(b, sizeof(int)))
	{
		printf("%p is safe\n", b);
	}
	else
	{
		printf("%p is not safe\n", b);
	}

	if(IsSafeAddress(d, SIZE))
	{
		printf("%p---%p is safe\n", d, d+100);
	}
	else
	{
		printf("%p---%p is not safe\n", d, d+100);
	}

	printf("exited normally\n");
    return 0;
}

```

```
$ ./main 
0x7ffd2001e1b4 is safe
(nil) is not safe
0x7a6e8fd010---0x7a6e8fd1a0 is safe
exited normally
```

Okay, all is well.



