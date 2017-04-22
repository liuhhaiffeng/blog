---
title: a nice progressbar with time estimation
comments: true
date: 2017-04-12 19:51:14
tags: [linux, c]
categories: [技术分享]
---


> Recently I've been making a progressbar for database server. I took a full day's time to carefully implement a linux shell based progressbar which not only can print progress on the terminal but also can estimate the remaining time. 

a screen shot of the progressbar.

<center> ![main.gif-47kB][1] </center>

It is implement by C programming language. The source code is below:


```c
#include<unistd.h>
#include<stdio.h>
#include<stdlib.h>
#include<time.h>


/*
 * print a progress bar like this
 *
 * LOG:  [========>                    ] 20%  (1h 30m 5s remaining)
 *
 * Argument:
 *      precentage range [0,100]
 *
 */
void
elog_progress(int precentage)
{
	char          str[1024]           = {0}; /* the progress str to print, should be enough */
	char         *fmt                 = "LOG:  [%-100s] %3d%%  %s\r";
	char          bars[100+1]         = {0};
	int           i;
	/* start time will be used the next call, make it static */
	static time_t start_time          = 0;
	/* we start estimate after 5 seconds */
	const  int    estimate_start_time = 5;
	char          time_str[1024]      = {0};
	char         *time_fmt            = "( %dh %2dm %2ds remaining )      ";
	int                             h = 0,
	                                m = 0,
	                                s = 0;

	/* if not within [0,100], return now */
	if (precentage < 0 || precentage > 100)
		return;

	/* fillin the progress bar */
	for (i = 0; i < precentage; i++)
	{
		bars[i] = '=';
	}

	/* fill the head of the progress bar */
	if (precentage > 0 && precentage < 100)
		bars[precentage] = '>';

	/* estimate time */
	if (start_time != 0)
	{
		time_t time_remaining = 0;
		time_t time_cost = 0;
		/* time cost since start */
		time_cost = time(NULL) - start_time;

		/* estimate time remaining */
		if(precentage > 0 && time_cost >= estimate_start_time)
		{
			time_remaining  = time_cost * (100 - precentage) / precentage;
			h = time_remaining/3600;
			m = time_remaining/60%60;
			s = time_remaining%60;
			sprintf(time_str, time_fmt, h, m, s);
		}
	}

	/* init start time on first run */
	if (start_time == 0)
		start_time = time(NULL);

	/* ok, we have everything we need , ready to output*/
	sprintf(str, fmt, bars, precentage, time_str);

	printf("%s",str);
	fflush(stdout);

	/* start a new line when 100% */
	if (precentage == 100)
	{
		printf("\n");
		fflush(stdout);
		/* reset time estimate */
		start_time = 0;
	}

	return;
}
int main()
{
	int precentage = 0;

	while(precentage<=100)
	{
		elog_progress(precentage);
		precentage++;
		usleep(100000);/* sleep for 10sec */
	}

}
```

  [1]: http://static.zybuluo.com/shenyuflying/50vqgj0i2dr4anyg5sdndqx2/main.gif


如果还想看到更多此类文章，请移步到[小宇的博客](http://shenyu.wiki)。
