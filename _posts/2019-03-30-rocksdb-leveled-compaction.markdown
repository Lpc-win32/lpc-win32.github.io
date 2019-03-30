---
layout:     post
title:      "rocksdb and badger leveled compaction"
subtitle:   " \"Rocksdb and badger leveled compaction\""
date:       2019-03-30 17:12
header-img: "img/post-bg-unix-linux.jpg"
author:     "pepperliu"
catalog:      true
tags:
    - 存储引擎
    - leveldb
    - rocksdb
    - badger
---

### 1. 文件的结构

文件在磁盘上的存储采用多层级（multiple levels）的方式。level0 ~ leveln。其中level0是特殊的，其数据是通过memtable flush而来。除了level0的数据无序外，level1 ~ leveln的数据均遵循字典序排序

![level_Compaction](http://blog.lpc-win32.com/img/2019-03-30/01.png)

除了level0的数据无序外，level1 ~ leveln的数据均遵循字典序排序，数据range partitioned在多个SST文件中

![level_Compaction](http://blog.lpc-win32.com/img/2019-03-30/02.png)

除level0外每个level在有序的执行，因为keys在每个SST文件中也是有序的。当查询一个key的位置时，我们首先使用binary search的方式查询各个文件的start与end1以确保key是否在这个文件中。如果在其范围内，则继续采用binary search的方式在这个文件中找到key的实际位置。没找到，则继续向更高层级查找

除level0外，每个level有目标大小，确保当前level小于下面level的大小，通常呈指数倍增

![level_Compaction](http://blog.lpc-win32.com/img/2019-03-30/03.png)

### 2. Compactions（压缩操作）

compaction操作在level0文件达到 level0\_file\_num\_compaction\_trigger 被触发，level0的文件将被合并入level1。通常必须将所有level0的文件合并到level1中，因为level0文件的key是有重叠的(overlapping)

![level_Compaction](http://blog.lpc-win32.com/img/2019-03-30/04.png)

level0 compaction操作之后，level1的大小可能会超过目标值

![level_Compaction](http://blog.lpc-win32.com/img/2019-03-30/05.png)

在这种场景下，我们会从level1至少选择一个文件，合并到level2中key有交叠的文件中。

![level_Compaction](http://blog.lpc-win32.com/img/2019-03-30/06.png)

如果结果导致push到下一个level的大小也超过了限制，我们会按照之前做的一样，至少选择一个文件，并且把它merge到下一个level

![level_Compaction](http://blog.lpc-win32.com/img/2019-03-30/07.png)

And Then

![level_Compaction](http://blog.lpc-win32.com/img/2019-03-30/08.png)

And Then

![level_Compaction](http://blog.lpc-win32.com/img/2019-03-30/09.png)

如果我们需要，多个compaction操作可以并行执行

![level_Compaction](http://blog.lpc-win32.com/img/2019-03-30/10.png)

最大compactions并行数由 max\_background\_compactions参数控制

然而，level0 to level1的compaction操作不能并行。这可能成为整体compaction速度的瓶颈，可以通过设置max\_subcompactions来提醒level0 to level1 compaction的性能，如下图所示：

![level_Compaction](http://blog.lpc-win32.com/img/2019-03-30/11.png)

### 3. Compaction Picking

当多个level同时达到了compaction操作的条件，RocksDB需要选择最先执行compaction的level。通过分数机制：

- 非0 levels而言，score \= level文件的总长度 \/ Target值。正在做compaction的文件不计入总行度中
- 对于level0而言，score \= max\{文件数量 \/ level0\_file\_num\_compaction\_trigger， level0文件总长度 \/ max\_bytes\_for\_level\_base} 并且level0文件数量 \> level0\_file\_num\_compaction\_trigger

我们通过比较每个level的score大小，选择得分最高的优先做compaction

### 4. Levels Target Size

- 当level\_compaction\_dynamic\_level\_bytes为false

```
L1 触发阈值：max\_bytes\_for\_level\_base  
Target_Size(Ln+1) = Target_Size(Ln) * max_bytes_for_level_multiplier * max_bytes_for_level_multiplier_additional[n].max_bytes_for_level_multiplier_additional

例如：
max_bytes_for_level_base = 16384
max_bytes_for_level_multiplier = 10
max_bytes_for_level_multiplier_additional = 1
那么每个level的触发阈值为 L1, L2, L3 and L4 分别为 16384, 163840, 1638400, 1638400
```

- 当level\_compaction\_dynamic\_level\_bytes为true

```
最后一个level的文件长度总是固定的。
上面level触发阈值通过公式计算：Target_Size(Ln-1) = Target_Size(Ln) / max_bytes_for_level_multiplier
如果计算得到的值小于 max_bytes_for_level_base / max_bytes_for_level_multiplier， 那么该level将维持为空，L0做compaction时将直接merge到第一个有合法阈值的level上。
例如：
max_bytes_for_level_base = 1G
num_levels = 6
level 6 size = 276G
那么从L1到L6的触发阈值分别为：0， 0， 0.276G， 2.76G， 27.6G，276G
```

![level_Compaction](http://blog.lpc-win32.com/img/2019-03-30/12.png)

这样分配，保证了稳定的LSM\-tree结构。并且有90%的数据存储在最后一层，9\%的数据保存在倒数第二层

### 5. TTL

简单来说就是为数据设置超时时间，规定时间内没有被使用则丢弃
