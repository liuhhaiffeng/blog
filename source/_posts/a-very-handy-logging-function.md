---
title: a very handy logging function
comments: true
date: 2017-08-15 16:56:35
tags:  [c]
categories: [技术分享]
---



> print a log prefixed with a timestamp.

sample
---------
```
2017-08-10 16:00:28.1050: hello
2017-08-10 16:00:28.1050: 123
```

todo
---------
1. add thread/process info
2. add log level
3. add strerror info "%m"


code
---------
```
#include <stdlib.h>
#include <stdio.h>
#include <sys/timeb.h>
#include <stdarg.h>
#include <time.h>
#include <string.h>

char *systime();
void logging(char *fmt, ...);

void
logging(char *fmt, ...)
{
	va_list		ap;

	fprintf(stderr, "%s: ", systime());
	va_start(ap, fmt);
	vfprintf(stderr, fmt, ap);
	va_end(ap);
	fprintf(stderr, "\n");
	fflush(stderr);
}

char * 
systime()
{
	static char DataTime[25];                           /* "%Y-%m-%d %H:%M:%S" is just nineteen number */
	char Millis[8];                                     /* Millisecond */

	time_t rawtime;
	struct tm *timeinfo;
	struct timeb timebuffer;

	ftime(&timebuffer);
	rawtime=timebuffer.time;
	timeinfo=localtime(&rawtime);                       /* get localtime */
	
	strftime(DataTime,25,"%Y-%m-%d %H:%M:%S",timeinfo); /* Customize the system time format */
	sprintf(Millis,".%+3.3hu0",timebuffer.millitm);
	strncpy(DataTime+19,Millis,5);
	
	return DataTime;
}

int main()
{
	logging("hello");
	logging("%d",123);

}

```





如果还想看到更多此类文章，请移步到[小宇的博客](http://shenyu.wiki)。
