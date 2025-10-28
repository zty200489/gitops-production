# Benchmark JuiceFS

Currently, our JuiceFS setup uses NFS as the backend object storage service and Postgresql as the metadata database. The NFS share runs atop a ZFS Dataset with `recordsize=4M`, `sync=always`, and `compression=zstd`. This allows for guaranteed data integrity and excellent compression rates. We conduct the perfromance benchmarks in the [official JuiceFS documentations](https://juicefs.com/docs/zh/community/benchmark/) to understand the performance baselines for future developemnt.

Before conducting tests, we turned off trash to avoid clogging up the system and to see how the system behaves under pressure with the command:

```
juicefs config <META-URL> --trash-days 0
```

## Benchmark results with `juicefs bench`

We perform basic benchmarking of the filesystem using the `juicefs bench` subcommand.

```cmd
 ~ ❯ juicefs bench /share -p 4
Cleaning kernel cache, may ask for root privilege...
  Write big blocks: 4096/4096 [==============================================================]  478.9/s  used: 8.552713993s
   Read big blocks: 4096/4096 [==============================================================]  1797.6/s used: 2.278648207s
Write small blocks: 400/400 [==============================================================]  240.9/s  used: 1.660763265s
 Read small blocks: 400/400 [==============================================================]  2314.8/s used: 172.844061ms
  Stat small files: 400/400 [==============================================================]  5597.5/s used: 71.532095ms 
Benchmark finished!
BlockSize: 1.0 MiB, BigFileSize: 1.0 GiB, SmallFileSize: 128 KiB, SmallFileCount: 100, NumThreads: 4
Time used: 15.4 s, CPU: 129.2%, Memory: 662.5 MiB
+------------------+------------------+---------------+
|       ITEM       |       VALUE      |      COST     |
+------------------+------------------+---------------+
|   Write big file |     478.94 MiB/s |   8.55 s/file |
|    Read big file |    1798.10 MiB/s |   2.28 s/file |
| Write small file |    240.9 files/s | 16.60 ms/file |
|  Read small file |   2322.3 files/s |  1.72 ms/file |
|        Stat file |   5637.9 files/s |  0.71 ms/file |
|   FUSE operation | 71308 operations |    0.68 ms/op |
|      Update meta |  6526 operations |    1.00 ms/op |
|       Put object |  1424 operations |  121.26 ms/op |
|       Get object |  1082 operations |   65.11 ms/op |
|    Delete object |  1424 operations |    3.59 ms/op |
| Write into cache |  1093 operations |    2.45 ms/op |
|  Read from cache |   342 operations |    0.07 ms/op |
+------------------+------------------+---------------+
```

## Benchmark results with `juicefs objbench`

Next we benchmark on the object storage service, which in our case is the NFS backend, directly using the `juicefs objbench` subcommand.

```cmd
 ~ ❯ sudo juicefs objbench --storage=nfs <OBJECT-URL>
Start Functional Testing ...
2025/10/28 15:13:03.093063 juicefs[1793897] <WARNING>: lookup uid 3001: user: unknown userid 3001 [UserName@utils.go:186]
2025/10/28 15:13:03.093247 juicefs[1793897] <WARNING>: lookup gid 3001: group: unknown groupid 3001 [GroupName@utils.go:202]
+----------+---------------------+-------------+
| CATEGORY |         TEST        |    RESULT   |
+----------+---------------------+-------------+
|    basic |     create a bucket |        pass |
|    basic |       put an object |        pass |
|    basic |       get an object |        pass |
|    basic |       get non-exist |        pass |
|    basic |  get partial object |        pass |
|    basic |      head an object |        pass |
|    basic |    delete an object |        pass |
|    basic |    delete non-exist |        pass |
|    basic |        list objects |        pass |
|     sync |         special key |        pass |
|     sync |    put a big object |        pass |
|     sync | put an empty object |        pass |
|     sync |    multipart upload | not support |
|     sync |  change owner/group |        pass |
|     sync |   change permission |        pass |
|     sync |        change mtime |        pass |
+----------+---------------------+-------------+

Start Performance Testing ...
 put small objects: 100/100 [==============================================================]  481.7/s    used: 207.61235ms 
 get small objects: 100/100 [==============================================================]  3018.2/s   used: 33.306787ms 
    upload objects: 256/256 [==============================================================]  94.2/s     used: 2.719109462s
  download objects: 256/256 [==============================================================]  393.9/s    used: 649.890497ms
      list objects: 1424/1424 [==============================================================]  143633.2/s used: 9.979224ms  
      head objects: 356/356 [==============================================================]  6695.9/s   used: 53.253846ms 
      update mtime: 356/356 [==============================================================]  3217.6/s   used: 110.724136ms
change permissions: 356/356 [==============================================================]  3235.1/s   used: 110.141436ms
change owner/group: 356/356 [==============================================================]  3307.3/s   used: 107.708736ms
    delete objects: 356/356 [==============================================================]  2124.5/s   used: 167.674046ms
Benchmark finished! block-size: 4.0 MiB, big-object-size: 1.0 GiB, small-object-size: 128 KiB, small-objects: 100, NumThreads: 4
+--------------------+---------------------+----------------------+
|        ITEM        |        VALUE        |         COST         |
+--------------------+---------------------+----------------------+
|     upload objects |        376.71 MiB/s |      42.47 ms/object |
|   download objects |       1577.58 MiB/s |      10.14 ms/object |
|  put small objects |    482.64 objects/s |       8.29 ms/object |
|  get small objects |   3158.35 objects/s |       1.27 ms/object |
|       list objects | 159889.32 objects/s | 8.91 ms/ 356 objects |
|       head objects |   6860.43 objects/s |       0.58 ms/object |
|     delete objects |   2142.19 objects/s |       1.87 ms/object |
| change permissions |   3252.51 objects/s |       1.23 ms/object |
| change owner/group |   3334.74 objects/s |       1.20 ms/object |
|       update mtime |   3229.79 objects/s |       1.24 ms/object |
+--------------------+---------------------+----------------------+
```

## Benchmark results with `fio`

We perform more complex file read and write test on the filesystem using `fio`. In accordance with the documentation, we perform 4 consecutive fio jobs covering sequential write, sequential read, random write, and random reads workloads.

### Sequential write

```cmd
 ~ ❯ fio --name=jfs-test --directory=/share/fio --ioengine=libaio --rw=write --bs=1m --size=1g --numjobs=4 --direct=1 --group_reporting
jfs-test: (g=0): rw=write, bs=(R) 1024KiB-1024KiB, (W) 1024KiB-1024KiB, (T) 1024KiB-1024KiB, ioengine=libaio, iodepth=1
...
fio-3.39
Starting 4 processes
jfs-test: Laying out IO file (1 file / 1024MiB)
jfs-test: Laying out IO file (1 file / 1024MiB)
jfs-test: Laying out IO file (1 file / 1024MiB)
jfs-test: Laying out IO file (1 file / 1024MiB)
Jobs: 4 (f=4): [f(4)][100.0%][eta 00m:00s]                        
jfs-test: (groupid=0, jobs=4): err= 0: pid=1797425: Tue Oct 28 15:19:43 2025
  write: IOPS=278, BW=278MiB/s (292MB/s)(4096MiB/14710msec); 0 zone resets
    slat (usec): min=532, max=86896, avg=14346.44, stdev=24356.80
    clat (nsec): min=1503, max=58693, avg=5553.12, stdev=2785.93
     lat (usec): min=537, max=86904, avg=14351.99, stdev=24357.83
    clat percentiles (nsec):
     |  1.00th=[ 2512],  5.00th=[ 3184], 10.00th=[ 3504], 20.00th=[ 3920],
     | 30.00th=[ 4256], 40.00th=[ 4640], 50.00th=[ 5024], 60.00th=[ 5408],
     | 70.00th=[ 5984], 80.00th=[ 6752], 90.00th=[ 7840], 95.00th=[ 8896],
     | 99.00th=[17280], 99.50th=[22400], 99.90th=[36096], 99.95th=[41728],
     | 99.99th=[58624]
   bw (  KiB/s): min=45056, max=1030144, per=98.07%, avg=279620.27, stdev=62974.30, samples=120
   iops        : min=   44, max= 1006, avg=273.07, stdev=61.50, samples=120
  lat (usec)   : 2=0.20%, 4=22.73%, 10=74.27%, 20=2.12%, 50=0.66%
  lat (usec)   : 100=0.02%
  cpu          : usr=0.39%, sys=1.07%, ctx=32889, majf=0, minf=33
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=0,4096,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
  WRITE: bw=278MiB/s (292MB/s), 278MiB/s-278MiB/s (292MB/s-292MB/s), io=4096MiB (4295MB), run=14710-14710msec
```

### Sequential read

```cmd
 ~ ❯ fio --name=jfs-test --directory=/share/fio --ioengine=libaio --rw=read --bs=1m --size=1g --numjobs=4 --direct=1 --group_reporting
jfs-test: (g=0): rw=read, bs=(R) 1024KiB-1024KiB, (W) 1024KiB-1024KiB, (T) 1024KiB-1024KiB, ioengine=libaio, iodepth=1
...
fio-3.39
Starting 4 processes
Jobs: 4 (f=4)
jfs-test: (groupid=0, jobs=4): err= 0: pid=1798182: Tue Oct 28 15:20:50 2025
  read: IOPS=1831, BW=1831MiB/s (1920MB/s)(4096MiB/2237msec)
    slat (usec): min=424, max=56318, avg=2144.68, stdev=4659.08
    clat (nsec): min=1191, max=474982, avg=5632.86, stdev=12819.36
     lat (usec): min=427, max=56325, avg=2150.31, stdev=4659.35
    clat percentiles (nsec):
     |  1.00th=[  1592],  5.00th=[  2384], 10.00th=[  3152], 20.00th=[  3792],
     | 30.00th=[  4192], 40.00th=[  4448], 50.00th=[  4768], 60.00th=[  5088],
     | 70.00th=[  5408], 80.00th=[  5856], 90.00th=[  6752], 95.00th=[  7648],
     | 99.00th=[ 16768], 99.50th=[ 28032], 99.90th=[162816], 99.95th=[346112],
     | 99.99th=[473088]
   bw (  MiB/s): min= 1638, max= 1980, per=98.52%, avg=1804.00, stdev=33.35, samples=16
   iops        : min= 1638, max= 1980, avg=1804.00, stdev=33.35, samples=16
  lat (usec)   : 2=3.08%, 4=21.58%, 10=73.27%, 20=1.34%, 50=0.32%
  lat (usec)   : 100=0.15%, 250=0.20%, 500=0.07%
  cpu          : usr=0.40%, sys=6.29%, ctx=32953, majf=0, minf=1063
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=4096,0,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
   READ: bw=1831MiB/s (1920MB/s), 1831MiB/s-1831MiB/s (1920MB/s-1920MB/s), io=4096MiB (4295MB), run=2237-2237msec
```

### Random write

```cmd
 ~ ❯ fio --name=jfs-test --directory=/share/fio --ioengine=libaio --rw=randwrite --bs=1m --size=1g --numjobs=4 --direct=1 --group_reporting
jfs-test: (g=0): rw=randwrite, bs=(R) 1024KiB-1024KiB, (W) 1024KiB-1024KiB, (T) 1024KiB-1024KiB, ioengine=libaio, iodepth=1
...
fio-3.39
Starting 4 processes
Jobs: 4 (f=3): [f(4)][100.0%][w=311MiB/s][w=311 IOPS][eta 00m:00s]
jfs-test: (groupid=0, jobs=4): err= 0: pid=1798603: Tue Oct 28 15:21:58 2025
  write: IOPS=197, BW=197MiB/s (207MB/s)(4096MiB/20746msec); 0 zone resets
    slat (usec): min=504, max=87697, avg=20196.19, stdev=27426.39
    clat (nsec): min=1738, max=231504, avg=6290.66, stdev=5069.39
     lat (usec): min=508, max=87705, avg=20202.48, stdev=27427.70
    clat percentiles (usec):
     |  1.00th=[    3],  5.00th=[    4], 10.00th=[    4], 20.00th=[    5],
     | 30.00th=[    5], 40.00th=[    6], 50.00th=[    6], 60.00th=[    7],
     | 70.00th=[    7], 80.00th=[    8], 90.00th=[    9], 95.00th=[   10],
     | 99.00th=[   22], 99.50th=[   28], 99.90th=[   50], 99.95th=[   52],
     | 99.99th=[  233]
   bw (  KiB/s): min=47104, max=745849, per=99.06%, avg=200273.15, stdev=37712.01, samples=167
   iops        : min=   46, max=  728, avg=195.56, stdev=36.82, samples=167
  lat (usec)   : 2=0.02%, 4=12.77%, 10=83.13%, 20=2.91%, 50=1.05%
  lat (usec)   : 100=0.10%, 250=0.02%
  cpu          : usr=0.33%, sys=0.78%, ctx=32961, majf=0, minf=33
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=0,4096,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
  WRITE: bw=197MiB/s (207MB/s), 197MiB/s-197MiB/s (207MB/s-207MB/s), io=4096MiB (4295MB), run=20746-20746msec
```

### Random read

```cmd
 ~ ❯ fio --name=jfs-test --directory=/share/fio --ioengine=libaio --rw=randread --bs=1m --size=1g --numjobs=4 --direct=1 --group_reporting
jfs-test: (g=0): rw=randread, bs=(R) 1024KiB-1024KiB, (W) 1024KiB-1024KiB, (T) 1024KiB-1024KiB, ioengine=libaio, iodepth=1
...
fio-3.39
Starting 4 processes
Jobs: 4 (f=4): [r(4)][60.0%][r=1333MiB/s][r=1333 IOPS][eta 00m:02s]
jfs-test: (groupid=0, jobs=4): err= 0: pid=1799208: Tue Oct 28 15:22:38 2025
  read: IOPS=1094, BW=1095MiB/s (1148MB/s)(4096MiB/3742msec)
    slat (usec): min=479, max=139290, avg=3283.45, stdev=7590.85
    clat (nsec): min=1443, max=151651, avg=6446.45, stdev=3542.97
     lat (usec): min=483, max=139304, avg=3289.90, stdev=7591.12
    clat percentiles (usec):
     |  1.00th=[    3],  5.00th=[    4], 10.00th=[    5], 20.00th=[    6],
     | 30.00th=[    6], 40.00th=[    6], 50.00th=[    7], 60.00th=[    7],
     | 70.00th=[    7], 80.00th=[    8], 90.00th=[    8], 95.00th=[    9],
     | 99.00th=[   19], 99.50th=[   23], 99.90th=[   37], 99.95th=[   44],
     | 99.99th=[  153]
   bw (  MiB/s): min=  540, max= 1868, per=100.00%, avg=1119.86, stdev=115.85, samples=25
   iops        : min=  540, max= 1868, avg=1119.31, stdev=115.91, samples=25
  lat (usec)   : 2=0.73%, 4=5.18%, 10=91.16%, 20=2.12%, 50=0.76%
  lat (usec)   : 100=0.02%, 250=0.02%
  cpu          : usr=0.60%, sys=4.15%, ctx=33003, majf=0, minf=1058
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=4096,0,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
   READ: bw=1095MiB/s (1148MB/s), 1095MiB/s-1095MiB/s (1148MB/s-1148MB/s), io=4096MiB (4295MB), run=3742-3742msec
```
