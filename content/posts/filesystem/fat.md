---
title: "Fat32文件系统详解"
date: 2020-05-14T20:06:37+08:00
description: ""
draft: false
tags: [文件系统]
categories: [文件系统]
---

本文主要介绍了fat32文件系统，包括硬盘分区的格式、文件分配表和目录项等重要内容。

下面简要的描述fat32文件系统比较关键的两个函数：

1. `f_mount`函数：在指定的硬盘上挂载文件系统（主要是读取引导区和`Volume ID`）；

2. `f_open`函数：

   a. 首先去根目录区（0x02H簇号），找到该文件的根目录（主要是匹配目录项，一个目录项占32B，所以一个簇大概$2^n*512/32=2^n*16$个目录项）；

   b. 查看当前目录项属性，如果是子目录，则查找下一个目录项（类似于步骤a）。如果是当前要查找的文件，则查找完毕；

### fat32磁盘空间的划分

首先第一个扇区（512字节）作为引导扇区，其中引导扇区记录了当前硬盘的分区情况，以及每个分区所包含的信息：文件系统类型、分区开始的扇区和占用扇区的个数；

然后根据对应的分区可以找到该分区的第一个扇区，该扇区称为`volume ID`。`volume ID`包含了一些关于当前文件系统的重要信息，以fat32为例：

* 每个扇区占多少字节（总是512字节）；
* 每个簇有多少个扇区（2的n次方）
* 保留扇区的个数；
* 文件分配表的个数（总是2，有一个是备份）；
* 文件分配表占用的扇区数（根据分区大小计算即可）；
* 根目录所在的簇（一般是2）；

接着是保留扇区空间；

再接着是文件分配表区（显示链接法）：

* fat32说明其文件分配表的表项是32位的；
* 每一个表项的位置代表着一个簇号，该表项的值表示下一个簇号；

最后是数据区，不仅包含了文件数据还包含了文件系统数据。

### 引导区

硬盘的第一个扇区称为引导区，也叫主引导扇区（MBR）。Boot Code也就是我们常说的bootloader代码。partition n表示分区，可以看到最多支持四个分区。

![](https://gitee.com/chengshuyi/scripts/raw/master/img/20200514222207.png)

这里我们主要看下分区描述符所占的16字节包含哪些信息？格式如下图所示：

![](https://gitee.com/chengshuyi/scripts/raw/master/img/20200602135143.png)

重要的信息有三个：

1. Type code：表明当前文件系统的类型；
2. LBA Begin：当前分区起始扇区号；
3. Number of Sectors：当前分区占有多少扇区；

### volume ID

分区的第一个扇区称为`volume ID`。`volume ID`包含了一些关于当前文件系统的重要信息：

* 每个扇区占多少字节（总是512字节）；
* 每个簇有多少个扇区（2的n次方）
* 保留扇区的个数；
* 文件分配表的个数（总是2，有一个是备份）；
* 文件分配表占用的扇区数（根据分区大小计算即可）；
* 根目录所在的簇（一般是2）；

![](https://gitee.com/chengshuyi/scripts/raw/master/img/20200602135838.png)

### 保留扇区

![](https://gitee.com/chengshuyi/scripts/raw/master/img/20200602161814.png)

### 文件分配表区

文件分配表区是FAT文件系统管理磁盘空间和文件的最重要区域，它保存逻辑盘数据区各簇使用情况信息，采用**位示图法**来表示，文件所占用的存储空间及空闲空间的管理都是通过FAT实现的。FAT区共保存了两个相同的文件分配表, 便于第一个损坏时, 还有第二个可用。

一个扇区的大小是512B，那么一个扇区可以存放128个簇号。因此，32bit的值中高7-31位表示扇区号（当前FAT表该簇号所在的扇区），低7位表示偏移（便宜单位是32bit）。

**未被分配使用和已回收**的簇相应位置写零，**坏簇**相应位置填入特定值 0FFFFFF7H 标识，**已分配**的簇相应位置填入非零值，具体为：如果该簇是文件的**最后一簇**, 填入的值为 0FFFFFFFH, 如果该簇**不是文件的最后一簇**，填入的值为该文件占用的下一个簇的簇号，这样，正好将文件占用的各簇构成一个簇链，保存在FAT32 表中。

### 数据区

数据区是存放文件数据的地方，这里需要特别考虑根目录区或者目录区。FAT32 文件系统中，根目录作为数据区的一部分，采用与子目录相似的管理方式，这 一点 与FAT12、FAT16明显不同，如FAT16文件系统的根目录区( ROOT区) 是固定区域、固定大小的，占用从FAT区之后紧接着的32个扇区，最多保存 512 个目录项( 其根目录保存的文件数受限的原因在此) ，作为系统区的一部分。FAT32的根目录一般情况下从02H簇开始使用，大小视需要增加，因此根目录下的文件数目不再受最多 512 的限制。

下图是目录项的格式：

![](https://gitee.com/chengshuyi/scripts/raw/master/img/20200602162221.png)

其中比较重要的有：

* 短文件名：8+3格式；
* 文件属性：只读、隐藏、是否为子目录等；
* 簇号（HIgh）：存储文件起始簇的簇号的高16位；
* 簇号（Low）：存储文件起始簇的簇号的低16位；
* 大小：文件的长度；

<!-- ### 专业名词缩写

LBA:logical block addressing

MBR:master Boot Record

CHS:Cylinder Head Sector -->

### 参考文献

[1]. Understanding FAT32 Filesystems. https://www.pjrc.com/tech/8051/ide/fat32.html.

[2]. 张明亮, 张宗杰. 浅析FAT32文件系统[J]. 计算机与数字工程, 2005(01):56-59.