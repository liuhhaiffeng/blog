---
title: hanoi
comments: true
date: 2017-06-24 12:44:47
tags: [c, programming, algorithm]
categories: 技术分享
---


> a visualized hanoi program.

<center> ![hanoi.gif-5.9kB][1] </center>

<!--more-->
```c

#include<unistd.h>
#include<stdio.h>
#include<stdlib.h>


#define N 10
int bars[3][N];

void printBar(int *onebar)
{
    int i;
    for(i = 0; i<N;i++)
    {
        if(onebar[i]!=0)
            printf("%d", onebar[i]);
        else
            printf("-");
    }
    printf("\n");
}

void print()
{
    int i;
    for(i=0;i<3;i++)
        printBar(bars[i]);
    printf("\n");
}

int find(int n)
{
    int which;
    int i;
    int j;
    for(i=0;i<3;i++)
    for(j=0;j<N;j++)
        if(bars[i][j] == n)
            return i;
}
void delete(int which, int n)
{
    int i;
    for(i=0;i<N;i++)
        if(bars[which][i] == n)
            bars[which][i] = 0;
}

void insert(int which, int n)
{
    int i;

    if(which == -1)
        which = 2;
    if(which == 3)
        which = 0;

    for(i=0;i<N;i++)
        if(bars[which][i] == 0)
        {
            bars[which][i] = n;
            return;
        }

    return;
}

void shift(int n, int d)
{
    int which;

    system("clear");
    printf("\nshift %d %s\n", n,d>0 ? "right" : "left");

    which = find(n);
    delete(which, n);
    insert(which+d,n);
    print();
    usleep(300000);

}

void hanoi(int n, int d)
{
    if(n==0) return;
    hanoi(n-1, -d);
    shift(n, d);
    hanoi(n-1, -d);
}

void init(int n)
{
    int i;
    for(i=0;i<n;i++)
    {
        bars[0][i] = n - i;
    }
}

int main(int argc, char *argv[])
{
    int n = atoi(argv[1]);

    init(n);
    print();
    hanoi(n,-1);
    print();

    return 0;
}

```


如果还想看到更多此类文章，请移步到[小宇的博客](http://shenyu.wiki)。
