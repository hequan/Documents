date: 2019-07-16 17:15
app: markdown
layout: 'post'
title: 'BeeGFS架构介绍'
tags: BeeGFS

# 0. 引言
在HPC领域，Lustre和GPFS一直都是这个领域的主角。Lustre虽然开源，但是太过庞大和复杂，需要专业的运维成本；GPFS是商业软件，普通用户无法获取其源代码。在HPC领域又出现了一个新的文件系统——BeeGFS，在2015年开源，详见：[https://www.beegfs.io/wiki/TableOfContents](https://www.beegfs.io/wiki/TableOfContents)。至于BeeGFS的发展，看一下IO500的排名：（在前40名有8套系统是基于BeeGFS）
![](./_image/2019-07-16/2019-07-16-19-20-48.png)
![](./_image/2019-07-16/2019-07-16-19-23-15.png)

>  从上图可以看出，在HPC的IO领域，仍然是Lustre和GPFS的天下

本文简单介绍一下BeeGFS

# 1. 概览
- 开源协议
    - Client——GPLv2
    - Server——[BeeGFS EULA](https://www.beegfs.io/docs/BeeGFS_EULA.txt)
        - EULA不是标准协议，是私有协议，有些特殊功能使用有问题
- 兼容平台
    - RHEL/Fedora、SLES/OpenSuse 、Debian/Ubuntu
- Linux内核版本
    - Linux kernels 2.6.18+
- 硬件平台
    - Intel/AMD x86_64、ARM、OpenPOWER、Windows client
- 支持的网络类型
    - Ethernet、InfiniBand、Omni-Path、RoCE
- 支持POSIX接口

# 2. 通用架构
BeeGFS主要由四个模块组成：
- 管理服务
- 数据存储服务
- 元数据服务
- 客户端服务

另外，BeeGFS提供了一个监控服务：ADMON
![](./_image/2019-07-16/2019-07-16-20-16-20.png)

## 2.1 管理服务
- light-weight
- The management service maintains a list of all other BeeGFS services and their state
## 2.2 数据存储服务
![](./_image/2019-07-16/2019-07-24-17-06-53.png)

- OST支持线性扩展
- 建议OST使用 RAID-6(8+2/10+2)的方式
- OST上的文件系统使用xfs
- 在OST上对小文件做了压缩处理
- 支持Striping
- 可以给OST设置不同的标签，选择算法会根据标签选择OST
- OST有阈值设置，目的是避免写爆磁盘

## 2.3 元数据服务
- 专用元数据服务器
- 元数据存到MDS的MDT上
- 底层磁盘的本地文件系统使用ext4
- 支持多MDS服务，元数据性能随着MDS数量的增加线性扩展(一定范围内)
## 2.4 客户端服务
- /etc/fstab方式挂载——非首选
- beegfs-mount.conf——首选的方式
- 可以共享挂载点——NFSv4/Samba/HDFS/Windows(近期支持)

# 3. 功能特性
## 3.1 Admon
正常情况下，BeeGFS提供beegfs-ctl命令行工具，这个工具十分强大。
除了命令行工具，BeeGFS还提供一个图形化界面：Admon， Administration and Monitoring System

## 3.2 Buddy Mirroring
这个是BeeGFS对比其它并行文件系统的一个优势，也就是支持镜像，可以认为是一个两副本，或者RAID-1的方式。
镜像是以target为单位的，比如文件A，在Server_a上存一份，在Server_b上仍然存储一份，镜像的形式有以下几种方式：

第一种是相互独立的，两两之间没有交互
![](./_image/2019-07-16/2019-07-24-17-24-50.png)



第二种是交互式的，任意选择两个Server存储数据

![](./_image/2019-07-16/2019-07-24-17-25-19.png)

其实这两种方式都没有区别，只是故障域不同，可以选择机器为故障域，也可以选择机器上的OST盘为故障域。

看过代码以后发现，这里的两副本遵循强一致性。
故障切换需要时间，一般是300s，看业务负载情况
切换以后如果OST没有恢复，以单副本运行
切换以后如果OST恢复了，会做数据同步操作(这里有坑)

## 3.3 Storage Pools
![](./_image/2019-07-16/2019-07-24-17-37-45.png)- 不是cache，是数据的流动
- 数据只会存在一份
- 数据如何流动？如何迁移数据？
- 可以针对pool设置quota
- 数据的mirror只能在一个pool中，不能跨pool存在
- pool的数量可以有很多，单个pool可以存在不同的存储介质

## 3.4 BeeOND: BeeGFS On Demand
这是一种Burst Buffer的方法，本质上搭建一套单副本的BeeGFS，后端BeeGFS作为持久性存储
![](./_image/2019-07-16/2019-07-24-17-39-08.png)

- BeeOND支持多种存储后端
- BeeOND有自己独立的挂载点，就像使用另一套存储
- 支持多种数据迁移工具：
    - cp
    - rsync
    - parallel copy tool

# Reference
[BeeGFS架构介绍](https://www.beegfs.io/docs/whitepapers/Introduction_to_BeeGFS_by_ThinkParQ.pdf)
[IO500详情](https://www.vi4io.org/std/io500/start)
[EULA协议](https://www.beegfs.io/docs/BeeGFS_EULA.txt)