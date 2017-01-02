---
title: PostgreSQL内存盘性能测试
comments: true
date: 2016-11-29 12:53:53
tags: [PostgreSQL]
---

> 磁盘IO是数据库性能的瓶颈。如果把数据放在内存盘上性能能提升多少呢？我也很好奇，于是用pg自带的pgbench测试了一下，结果表明在SSD硬盘上性能为207tps；在内存盘上性能为1897tps；性能提升了9倍。

基本配置
-------------
电脑品牌：Thinkpad X201
CPU:Intel(R) Core(TM) i7 CPU       M 620  @ 2.67GHz
SSD磁盘性能：
随机写：4k，162M/s，iops=40656
随机读：4k，178M/s，iops=55240
顺序写：362M/s
内存盘性能： 2.0 GB/s
内存：8G
数据库采用默认配置

<!--more-->

```
$ cat /proc/cpuinfo 
processor	: 0
vendor_id	: GenuineIntel
cpu family	: 6
model		: 37
model name	: Intel(R) Core(TM) i7 CPU       M 620  @ 2.67GHz
stepping	: 2
microcode	: 0xe
cpu MHz		: 2659.889
cache size	: 4096 KB

$ free -g
              total        used        free      shared  buff/cache   available
Mem:              7           1           2           2           3           3
Swap:             7           0           7

$ fio -direct=1 -iodepth=128 -rw=randwrite -ioengine=libaio -bs=4k -size=2G -numjobs=1 -runtime=10 -group_reporting -name=./tmp
./tmp: (g=0): rw=randwrite, bs=4K-4K/4K-4K/4K-4K, ioengine=libaio, iodepth=128
fio-2.2.10
Starting 1 process
Jobs: 1 (f=1): [w(1)] [100.0% done] [0KB/158.6MB/0KB /s] [0/40.6K/0 iops] [eta 00m:00s]
./tmp: (groupid=0, jobs=1): err= 0: pid=16914: Tue Nov 29 20:24:40 2016
  write: io=1591.0MB, bw=162626KB/s, iops=40656, runt= 10018msec
    slat (usec): min=3, max=9846, avg=13.45, stdev=65.57
    clat (usec): min=596, max=33299, avg=3131.62, stdev=1837.53
     lat (usec): min=718, max=33324, avg=3145.41, stdev=1839.00
    clat percentiles (usec):
     |  1.00th=[  828],  5.00th=[ 1048], 10.00th=[ 1288], 20.00th=[ 1672],
     | 30.00th=[ 1976], 40.00th=[ 2352], 50.00th=[ 2800], 60.00th=[ 3280],
     | 70.00th=[ 3760], 80.00th=[ 4320], 90.00th=[ 5216], 95.00th=[ 6176],
     | 99.00th=[ 9408], 99.50th=[11072], 99.90th=[16320], 99.95th=[20352],
     | 99.99th=[28032]
    bw (KB  /s): min=134968, max=178976, per=100.00%, avg=162657.65, stdev=8446.26
    lat (usec) : 750=0.13%, 1000=3.92%
    lat (msec) : 2=26.45%, 4=43.78%, 10=24.95%, 20=0.72%, 50=0.05%
  cpu          : usr=10.98%, sys=52.95%, ctx=134871, majf=0, minf=11
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.1%, 32=0.1%, >=64=100.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.1%
     issued    : total=r=0/w=407296/d=0, short=r=0/w=0/d=0, drop=r=0/w=0/d=0
     latency   : target=0, window=0, percentile=100.00%, depth=128

Run status group 0 (all jobs):
  WRITE: io=1591.0MB, aggrb=162625KB/s, minb=162625KB/s, maxb=162625KB/s, mint=10018msec, maxt=10018msec

Disk stats (read/write):
    dm-0: ios=0/403504, merge=0/0, ticks=0/998476, in_queue=999688, util=95.90%, aggrios=0/406834, aggrmerge=0/1774, aggrticks=0/1005596, aggrin_queue=1006352, aggrutil=95.37%
  sda: ios=0/406834, merge=0/1774, ticks=0/1005596, in_queue=1006352, util=95.37%





 $ fio -direct=1 -iodepth=128 -rw=randread -ioengine=libaio -bs=4k -size=2G -numjobs=1 -runtime=10 -group_reporting -name=./tmp
./tmp: (g=0): rw=randread, bs=4K-4K/4K-4K/4K-4K, ioengine=libaio, iodepth=128
fio-2.2.10
Starting 1 process
Jobs: 1 (f=1): [r(1)] [90.9% done] [178.3MB/0KB/0KB /s] [45.7K/0/0 iops] [eta 00m:01s]
./tmp: (groupid=0, jobs=1): err= 0: pid=17067: Tue Nov 29 20:26:26 2016
  read : io=2048.0MB, bw=220962KB/s, iops=55240, runt=  9491msec
    slat (usec): min=1, max=6358, avg= 9.30, stdev=49.59
    clat (usec): min=2, max=28539, avg=2305.34, stdev=1747.77
     lat (usec): min=4, max=28556, avg=2314.95, stdev=1750.45
    clat percentiles (usec):
     |  1.00th=[  354],  5.00th=[  382], 10.00th=[  394], 20.00th=[  442],
     | 30.00th=[ 1176], 40.00th=[ 1608], 50.00th=[ 2024], 60.00th=[ 2512],
     | 70.00th=[ 3056], 80.00th=[ 3632], 90.00th=[ 4384], 95.00th=[ 5216],
     | 99.00th=[ 7776], 99.50th=[ 9152], 99.90th=[13504], 99.95th=[15296],
     | 99.99th=[19584]
    bw (KB  /s): min=174240, max=184504, per=80.94%, avg=178856.28, stdev=3158.59
    lat (usec) : 4=0.01%, 10=0.01%, 20=0.01%, 50=0.01%, 100=0.01%
    lat (usec) : 250=0.02%, 500=21.63%, 750=1.12%, 1000=4.08%
    lat (msec) : 2=22.54%, 4=35.98%, 10=14.26%, 20=0.35%, 50=0.01%
  cpu          : usr=12.77%, sys=50.96%, ctx=127960, majf=0, minf=138
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.1%, 32=0.1%, >=64=100.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.1%
     issued    : total=r=524288/w=0/d=0, short=r=0/w=0/d=0, drop=r=0/w=0/d=0
     latency   : target=0, window=0, percentile=100.00%, depth=128

Run status group 0 (all jobs):
   READ: io=2048.0MB, aggrb=220962KB/s, minb=220962KB/s, maxb=220962KB/s, mint=9491msec, maxt=9491msec

Disk stats (read/write):
    dm-0: ios=407296/3, merge=0/0, ticks=958808/12, in_queue=959388, util=93.58%, aggrios=406755/3, aggrmerge=541/1, aggrticks=956908/12, aggrin_queue=959268, aggrutil=92.23%
  sda: ios=406755/3, merge=541/1, ticks=956908/12, in_queue=959268, util=92.23%

$ dd if=/dev/zero of=./tmp bs=8k count=300000
300000+0 records in
300000+0 records out
2457600000 bytes (2.5 GB, 2.3 GiB) copied, 6.793 s, 362 MB/s

$ dd if=/dev/zero of=./tmp bs=8k count=300000
300000+0 records in
300000+0 records out
2457600000 bytes (2.5 GB, 2.3 GiB) copied, 1.24375 s, 2.0 GB/s

```

data目录在磁盘上
-----------
```
$ ./pgbench --client=16 --jobs=16  --time=20 --progress=1  postgres
starting vacuum...end.
progress: 1.0 s, 204.0 tps, lat 69.831 ms stddev 57.034
progress: 2.0 s, 206.0 tps, lat 79.884 ms stddev 64.072
progress: 3.0 s, 207.0 tps, lat 79.170 ms stddev 55.301
progress: 4.0 s, 204.0 tps, lat 76.850 ms stddev 43.849
progress: 5.0 s, 207.0 tps, lat 77.856 ms stddev 44.732
progress: 6.0 s, 190.0 tps, lat 84.452 ms stddev 70.720
progress: 7.0 s, 211.0 tps, lat 75.539 ms stddev 49.628
progress: 8.0 s, 210.0 tps, lat 75.229 ms stddev 51.632
progress: 9.0 s, 211.0 tps, lat 75.075 ms stddev 49.451
progress: 10.0 s, 207.0 tps, lat 79.154 ms stddev 63.283
progress: 11.0 s, 212.0 tps, lat 76.071 ms stddev 47.042
progress: 12.0 s, 209.9 tps, lat 74.868 ms stddev 59.072
progress: 13.0 s, 210.1 tps, lat 77.152 ms stddev 51.673
progress: 14.0 s, 210.0 tps, lat 76.003 ms stddev 51.179
progress: 15.0 s, 210.0 tps, lat 75.999 ms stddev 62.302
progress: 16.0 s, 210.0 tps, lat 74.284 ms stddev 44.574
progress: 17.0 s, 210.0 tps, lat 78.569 ms stddev 54.181
progress: 18.0 s, 207.0 tps, lat 77.143 ms stddev 57.839
progress: 19.0 s, 207.0 tps, lat 75.662 ms stddev 52.170
progress: 20.0 s, 210.0 tps, lat 75.412 ms stddev 60.447
transaction type: TPC-B (sort of)
scaling factor: 1
query mode: simple
number of clients: 16
number of threads: 16
duration: 20 s
number of transactions actually processed: 4169
latency average: 76.853 ms
latency stddev: 55.239 ms
tps = 207.607456 (including connections establishing)
tps = 207.748916 (excluding connections establishing)

```

data目录在内存中
---------------------
```
$ ./pgbench --client=16 --jobs=16  --time=20 --progress=1  postgres
starting vacuum...end.
progress: 1.0 s, 1710.8 tps, lat 9.217 ms stddev 5.359
progress: 2.0 s, 2055.9 tps, lat 7.771 ms stddev 4.212
progress: 3.0 s, 1837.2 tps, lat 8.704 ms stddev 5.085
progress: 4.0 s, 1989.9 tps, lat 7.992 ms stddev 4.249
progress: 5.0 s, 1856.2 tps, lat 8.680 ms stddev 5.048
progress: 6.0 s, 1971.6 tps, lat 8.122 ms stddev 4.327
progress: 7.0 s, 1804.0 tps, lat 8.834 ms stddev 5.235
progress: 8.0 s, 1973.6 tps, lat 8.124 ms stddev 4.456
progress: 9.0 s, 2152.9 tps, lat 7.434 ms stddev 4.087
progress: 10.0 s, 1843.9 tps, lat 8.665 ms stddev 5.084
progress: 11.0 s, 2077.6 tps, lat 7.698 ms stddev 4.675
progress: 12.0 s, 1662.5 tps, lat 9.637 ms stddev 5.916
progress: 13.0 s, 1853.4 tps, lat 8.615 ms stddev 4.853
progress: 14.0 s, 1812.1 tps, lat 8.840 ms stddev 4.836
progress: 15.0 s, 2052.6 tps, lat 7.798 ms stddev 4.230
progress: 16.0 s, 1892.0 tps, lat 8.444 ms stddev 4.816
progress: 17.0 s, 1712.4 tps, lat 9.321 ms stddev 5.256
progress: 18.0 s, 1856.6 tps, lat 8.637 ms stddev 4.602
progress: 19.0 s, 1753.9 tps, lat 9.130 ms stddev 5.089
progress: 20.0 s, 2071.4 tps, lat 7.719 ms stddev 4.375
transaction type: TPC-B (sort of)
scaling factor: 1
query mode: simple
number of clients: 16
number of threads: 16
duration: 20 s
number of transactions actually processed: 37956
latency average: 8.430 ms
latency stddev: 4.822 ms
tps = 1896.061501 (including connections establishing)
tps = 1897.040816 (excluding connections establishing)

```


总结
----------
在磁盘上性能为207tps
在内存上性能为1897tps
提升了9倍。




如果还想看到更多此类文章，请移步到[小宇的博客](http://shenyu.wiki)。
