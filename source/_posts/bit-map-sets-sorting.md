---
title: 'bitmap sets sorting'
comments: true
date: 2017-04-03 10:32:06
tags: [编程]
categories: 技术分享
---

> BitMapSet can be a very powerful when used for sorting/finding. Because work can be done in O(N) time for sorting and O(1) for finding. Compared to quick sort algorithm, the data is stored in BitMapSet once the data is read into memory. Compared to hash finding, the data is sorted in BitMapSet. In my test, 1,000,000 numbers can be sorted in less than 0.09s on my respberripi and only occupy 125k of memory.


<!--more-->


```c
#include<stdio.h>
#include<stdlib.h>
#include<assert.h>
#include<time.h>


#define bool int 
#define true !(0) 
#define false 0 

/* start of BitMapSet implementation */
typedef struct BitMapSet {
    long int   bits;  /* how many bit are there ? */
    void      *data; /* data repersenting the bit 0 or 1 */
} BitMapSet;

BitMapSet *bitmapset_new(long int max_number)
{
    BitMapSet *res = (BitMapSet *)malloc(sizeof(*res));
    int        bytes; /* for memalloc  */

    bytes = max_number/8 + 1;

    res->bits  = max_number;
    res->data = (void *)calloc(bytes, sizeof(char));
    printf("mem:%d\n",bytes);

    return res;
}

static bool 
bitmapset_set_or_get(BitMapSet *bms, long int location, bool is_set)
{
    char *ptr;
    int pos; /* which char in the bms data ? */
    int i;
    int value = 1;
    char res=0;

    if (location >= bms->bits)
    {
        fprintf(stderr, "value %d exceeds the capcity of bitmapset(%d)\n",location,bms->bits);
        exit(1);
    }

    ptr = (char *)bms->data;
    pos = location/8;

    ptr+=pos;
    location-=(8*pos);

    value=value<<(location);

    if(is_set)
        *ptr = (*ptr)|value;
    else
        res = (*ptr)&value;

    return (bool)res;
}
void
bitmapset_set(BitMapSet *bms, long int location)
{
    bitmapset_set_or_get(bms,location,true);

    return;
}
bool
bitmapset_get(BitMapSet *bms, long int location)
{

    return  bitmapset_set_or_get(bms,location,false);
}

BitMapSet *bitmapset_add(BitMapSet *bms, int value)
{
    bitmapset_set(bms, value);
    return bms;
}
void
bitmapset_print(BitMapSet *bms)
{
    int i;
    printf("cap = %d\n",bms->bits);
    for(i=1;i < bms->bits;i++)
    {
        if(bitmapset_get(bms,i))
          printf("%d\t",i);
    }
}
/* end of BitMapSet implementation */

int main()
{
    int max=1000000;
    BitMapSet *bms = bitmapset_new(max);
    do{
        bitmapset_add(bms,--max);
    }while(max);

    bitmapset_print(bms);

}


```