---
title: yap--yet another program profiling tool
comments: true
date: 2016-10-14 07:59:13
tags: [编程语言, Shell]
categories: 开源社区
---


> Sampling tools like oprofile or dtrace's profile provider don't really provide methods to see what [multithreaded] programs are blocking on - only where they spend CPU time. Though there exist advanced techniques (such as systemtap and dtrace call level probes), it is overkill to build upon that. Poor man doesn't have time. Poor man needs food.  --quoted from [poor man's profiler](https://poormansprofiler.org/)

Just as the name implies, this is an enhenced version of poor man's profiler with the following new features added:

1. support command line argument, which make it easier to use and easier to integrate to your program.
2. support multi process profiling, which is good for PostgreSQL profiling.
3. user can choose how many stack level to print, which make the result set shorter and easier to read.
4. when stack level is set to 1 , instead of the stack ranking, the function ranking is printed
4. the result set is neater and much more pretty.


INSTALL
--------
Download it from `https://github.com/shenyuflying/yap.git`
```
git clone https://github.com/shenyuflying/yap.git
```
Type `yap -h` to show the help message. 
```
# ./yap -h
================================================================
                          YAP    ver0.1                         
================================================================
  yap -- yet another profiling tool based on poor man's profiler
         to generate stack rank of your program.                
                                                                
                             yshen 2016 see http://shenyu.wiki  
================================================================
                                                         
usage: yap options=values
        -s | --samples=n      how many samples to collect
        -t | --sleeptime=n    how many time to sleep during each sample
        -f | --stackframe=n   how deep the stack frame to output
        -p | --pid=pid        which pid to analyze
        -n | --progname=name  which program to analyze, good for multi-process profiling
        -h | --help           show this help
```
if the message is shown, the yap is ready to work.


HOW TO USE
-----------

You need at least specify a progname or a pid to let yap attach to your program.
```
# ./yap  --progname=postgres
```
will be enough for the yap to run. after you have choosen a program to profile, you will be run with the default configurations, and `yap` is start working.
the default configurations are
```
samples=100   # take 100 samples of your program
sleeptime=0   # no sleep during each iteration
stackframe=5  # the stack frame depth is 5
```
it will take a while and there will be a progress indicator on the screen, so take your time.
```
# ./yap  --progname=postgres
1/100 completed.
2/100 completed.
3/100 completed.
4/100 completed.
5/100 completed.
...
100/100 complete.
```
when done, the stack ranking is printed.
```
40 __epoll_wait_nocancel	WaitEventSetWaitBlock	WaitEventSetWait	WaitLatchOrSocket	WaitLatch
10 __select_nocancel	ServerLoop	PostmasterMain	main
10 __epoll_wait_nocancel	WaitEventSetWaitBlock	WaitEventSetWait	WaitLatchOrSocket	SysLoggerMain
10 __epoll_wait_nocancel	WaitEventSetWaitBlock	WaitEventSetWait	WaitLatchOrSocket	PgstatCollectorMain
1 
```
if you want to seek the function ranking rather than stack ranking, use `--stackframe=1` and re-run yap

```
     60 __epoll_wait_nocancel
     10 __select_nocancel
      1 
```
the function ranking is much sorter and easier to find the slowest functions.

TODO
-------

1. shall be run in a time duration, add -b | --begin and -e | --end and -d | --duration
2. shall be exit when the profiling program exit
3. ...


report bugs to shenyufly@163.com




