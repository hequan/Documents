Table of Contents
=================

   * [1.前言](#1前言)
   * [2.准备软件环境](#2准备软件环境)
      * [2.1 准备操作系统](#21-准备操作系统)
         * [2.1.1 关闭SELINUX](#211-关闭selinux)
         * [2.1.2 关闭防火墙](#212-关闭防火墙)
         * [2.1.3 同步时钟](#213-同步时钟)
      * [2.2  升级Kernel](#22--升级kernel)
         * [2.2.1 安装依赖](#221-安装依赖)
         * [2.2.2 替换kernel rpm包](#222-替换kernel-rpm包)
      * [2.3 安装IB驱动](#23-安装ib驱动)
         * [2.3.1 安装依赖](#231-安装依赖)
         * [2.3.2 编译IB驱动](#232-编译ib驱动)
      * [2.4 安装Lustre相关rpm包](#24-安装lustre相关rpm包)
         * [2.4.1 安装e2fsprogs相关包](#241-安装e2fsprogs相关包)
         * [2.4.2 安装Lustre Server](#242-安装lustre-server)
      * [2.5 启动Lustre模块](#25-启动lustre模块)
         * [2.5.1 写lustre.conf文件](#251-写lustreconf文件)
         * [2.5.2 启动Lustre模块](#252-启动lustre模块)
         * [2.5.3 检查Lustre模块是否启动完毕](#253-检查lustre模块是否启动完毕)
   * [3.格式化文件系统](#3格式化文件系统)
      * [3.1 说明](#31-说明)
      * [3.2 格式化命令](#32-格式化命令)
   * [4.挂载使用Lustre](#4挂载使用lustre)
      * [4.1 安装客户端](#41-安装客户端)
      * [4.2 挂载Lustre](#42-挂载lustre)
   * [Reference](#reference)

# 1.前言
Lustre文件系统的介绍暂时掠过，后面我会专门用一个主题来介绍Lustre文件系统。  
本文默认读者已经了解Lustre文件系统的架构，因此这里主要讲述如何从零搭建一套可用的Lustre环境。  

硬件配置: 
- 2个MDS节点(每个MDS节点配置2个MDT)
- 4个OSS节点(每个OSS节点配置4个OST)

Lustre环境搭建主要分为两个部分：
- 1、准备软件环境 
- 2、格式化文件系统

# 2.准备软件环境
- Lustre版本 2.10.3-基于IB
- CentOS 7.4.1708  
- OFED驱动版本 4.2-1.2.0.0

## 2.1 准备操作系统
### 2.1.1 关闭SELINUX
```bash
vim /etc/sysconfig/selinux 
修改相应字段
SELINUX=disabled
```
### 2.1.2 关闭防火墙
```bash
systemctl stop firewalld.service
systemctl disable firewalld.service
```
### 2.1.3 同步时钟
```bash
timedatectl set-timezone Asia/Shanghai
```

## 2.2  升级Kernel
### 2.2.1 安装依赖
```bash
yum -y install net-tools vim epel-release xmlto asciidoc elfutils-libelf-devel zlib-devel binutils-devel newt-devel python-devel hmaccalc perl-ExtUtils-Embed bison elfutils-devel audit-libs-devel python-docutils sg3_utils expect attr lsof quilt libselinux-devel libtool linux-firmware xfsprogs kmod libyaml libyaml-devel
```

### 2.2.2 替换kernel rpm包
```bash
rpm -Uvh kernel-tools-libs-3.10.0-693.11.6.el7_lustre.x86_64.rpm kernel-tools-3.10.0-693.11.6.el7_lustre.x86_64.rpm kernel-tools-libs-devel-3.10.0-693.11.6.el7_lustre.x86_64.rpm
rpm -Uvh kernel-headers-3.10.0-693.11.6.el7_lustre.x86_64.rpm kernel-devel-3.10.0-693.11.6.el7_lustre.x86_64.rpm kernel-3.10.0-693.11.6.el7_lustre.x86_64.rpm
```
> rpm包下载地址: https://downloads.whamcloud.com/public/lustre/lustre-2.10.3-ib/MOFED-4.2-1.2.0.0/el7.4.1708/server/RPMS/x86_64/
> 这个包是社区编译好的，可以直接替换

>  安装完后执行下面命令，确定内核是否已经更新: grub2-editenv list
```bash
[root@~]# grub2-editenv list
saved_entry=CentOS Linux (3.10.0-693.11.6.el7_lustre.x86_64) 7 (Core)
说明内核已经更新完成
```
>  更换完毕，重启服务器使内核生效  

## 2.3 安装IB驱动
### 2.3.1 安装依赖
```bash
yum -y install pciutils gtk2 atk cairo libxml2-python tcsh libnl lsof tcl tk createrepo gcc-gfortran gcc createrepo libyaml libyaml-devel
```
>  这里列出了基本的依赖，如果有缺少，按照指示安装即可

### 2.3.2 编译IB驱动
从MLNX官方下载IB驱动，本文使用的是MLNX_OFED_LINUX-4.2-1.2.0.0-rhel7.4-x86_64
- 解压IB驱动包
- 进入目录
- 编译安装驱动
```bash
./mlnxofedinstall --without-fw-update --add-kernel-support
```
>  如果缺少依赖，这里会报错，根据提示安装即可。

- 检查IB驱动版本
```bash
[root@ ~]# hca_self_test.ofed
Host Driver Version .................... MLNX_OFED_LINUX-4.5-1.0.1.0 (OFED-4.5-1.0.1): 3.10.0-693.11.6.el7_lustre.x86_64

[root@ ~]# ofed_info -s
MLNX_OFED_LINUX-4.5-1.0.1.0:
```
> 上面两种方法均可检查版本

- 重启IB服务，使其生效
```bash
/etc/init.d/openibd restart
```
- 配置ipoib地址
```bash
修改: /etc/sysconfig/network-scripts/ifcfg-ib0
```

## 2.4 安装Lustre相关rpm包
### 2.4.1 安装e2fsprogs相关包
rpm包下载地址: https://downloads.whamcloud.com/public/e2fsprogs/latest/el7/RPMS/x86_64/

执行下面的安装命令:
```bash
rpm -Uvh e2fsprogs-*.rpm libcom_err-*.rpm libss-*.rpm
```

### 2.4.2 安装Lustre Server
rpm包下载地址: https://downloads.whamcloud.com/public/lustre/lustre-2.10.3-ib/MOFED-4.2-1.2.0.0/el7.4.1708/server/RPMS/x86_64/

执行下面的安装命令:
```bash
rpm -Uvh lustre-osd-ldiskfs-mount-2.10.3-1.el7.x86_64.rpm kmod-lustre-2.10.3-1.el7.x86_64.rpm kmod-lustre-osd-ldiskfs-2.10.3-1.el7.x86_64.rpm lustre-2.10.3-1.el7.x86_64.rpm
```

## 2.5 启动Lustre模块
### 2.5.1 写lustre.conf文件
在lustre.conf文件中指定lnet使用的网络类型
- 使用IB的方式
```bash
vim /etc/modprobe.d/lustre.conf
options lnet networks=o2ib(ib0)
```
- 使用TCP的方式
```bash
vim /etc/modprobe.d/lustre.conf
options lnet networks=tcp(eth0)
```
- 二者同时使用
```bash
vim /etc/modprobe.d/lustre.conf
options lnet networks=o2ib(ib0),tcp(eth0)
```
>  这种方式优先使用o2ib的方式

### 2.5.2 启动Lustre模块
```bash
modprobe lnet
lctl network configure
```
### 2.5.3 检查Lustre模块是否启动完毕
```bash
[root@ ~]# lctl list_nids
192.1.168.10@o2ib
```

# 3.格式化文件系统
## 3.1 说明
- 两个MDS节点，分别是mds00和mds01
    - mds00有两块数据盘, sdb和sdc
    - mds01有两块数据盘，sdb和sdc
- 四个OSS节点，分别是oss00、oss01、oss02、oss03
    - oss0x有四块数据盘，sdb、sdc、sdd、sde
- mgt和mdt00公用一个设备，不指定专门的mdt设备
- mdt的inode大小和ost的inode大小需要根据实际情况进行调整，后面会专门写一篇文章来说明

## 3.2 格式化命令
在mds00上执行如下的命令:
```bash
mkfs.lustre --fsname=testfs --mgs --mdt --index=0 --mkfsoptions="-i 2048" --servicenode=10.1.10.6@o2ib --reformat /dev/sdb
mkfs.lustre --fsname=testfs --mdt --index=1 --servicenode=10.1.10.6@o2ib --mgsnode=10.1.10.6@o2ib --mkfsoptions="-i 2048" --reformat /dev/sdc

mkdir /mnt/mgtmdt00
mkdir /mnt/mdt01

mount.lustre /dev/sdb /mnt/mgtmdt00
mount.lustre /dev/sdc /mnt/mdt01
```
>  mgt所在的节点就是mgsnode，这里将mgt和mdt00放在一个设备上
  
在mds01上执行如下的格式化命令：
```bash
mkfs.lustre --fsname=testfs --mdt --index=2 --servicenode=10.1.10.7@o2ib --mgsnode=10.1.10.6@o2ib --mkfsoptions="-i 2048" --reformat /dev/sdb
mkfs.lustre --fsname=testfs --mdt --index=3 --servicenode=10.1.10.7@o2ib --mgsnode=10.1.10.6@o2ib --mkfsoptions="-i 2048" --reformat /dev/sdc

mkdir /mnt/mdt02
mkdir /mnt/mdt03

mount.lustre /dev/sdb /mnt/mdt02
mount.lustre /dev/sdc /mnt/mdt03
```
  
在oss01上执行如下的格式化命令:
```bash
kfs.lustre --fsname=testfs --ost --index=0 --servicenode=10.1.10.8@o2ib --mgsnode=10.1.10.6@o2ib --mkfsoptions="-i 16384" --reformat /dev/sdb

mkfs.lustre --fsname=testfs --ost --index=1 --servicenode=10.1.10.8@o2ib --mgsnode=10.1.10.6@o2ib --mkfsoptions="-i 16384" --reformat /dev/sdc

mkfs.lustre --fsname=testfs --ost --index=2 --servicenode=10.1.10.8@o2ib --mgsnode=10.1.10.6@o2ib --mkfsoptions="-i 16384" --reformat /dev/sdd

mkfs.lustre --fsname=testfs --ost --index=3 --servicenode=10.1.10.8@o2ib --mgsnode=10.1.10.6@o2ib --mkfsoptions="-i 16384" --reformat /dev/sde

mkdir /mnt/ost00
mkdir /mnt/ost01
mkdir /mnt/ost02
mkdir /mnt/ost03

mount.lustre /dev/sdb /mnt/ost00
mount.lustre /dev/sdc /mnt/ost01
mount.lustre /dev/sdd /mnt/ost02
mount.lustre /dev/sde /mnt/ost03
```
  
在oss02上执行如下的格式化命令：
```bash
mkfs.lustre --fsname=testfs --ost --index=4 --servicenode=10.1.10.9@o2ib --mgsnode=10.1.10.6@o2ib --mkfsoptions="-i 16384" --reformat /dev/sdb

mkfs.lustre --fsname=testfs --ost --index=5 --servicenode=10.1.10.9@o2ib --mgsnode=10.1.10.6@o2ib --mkfsoptions="-i 16384" --reformat /dev/sdc

mkfs.lustre --fsname=testfs --ost --index=6 --servicenode=10.1.10.9@o2ib --mgsnode=10.1.10.6@o2ib --mkfsoptions="-i 16384" --reformat /dev/sdd

mkfs.lustre --fsname=testfs --ost --index=7 --servicenode=10.1.10.9@o2ib --mgsnode=10.1.10.6@o2ib --mkfsoptions="-i 16384" --reformat /dev/sde

mkdir /mnt/ost04
mkdir /mnt/ost05
mkdir /mnt/ost06
mkdir /mnt/ost07

mount.lustre /dev/sdb /mnt/ost04
mount.lustre /dev/sdc /mnt/ost05
mount.lustre /dev/sdd /mnt/ost06
mount.lustre /dev/sde /mnt/ost07
```

在oss03上执行如下的格式化命令:
```bash
mkfs.lustre --fsname=testfs --ost --index=8 --servicenode=10.1.10.10@o2ib --mgsnode=10.1.10.6@o2ib --mkfsoptions="-i 16384" --reformat /dev/sdb

mkfs.lustre --fsname=testfs --ost --index=9 --servicenode=10.1.10.10@o2ib --mgsnode=10.1.10.6@o2ib --mkfsoptions="-i 16384" --reformat /dev/sdc

mkfs.lustre --fsname=testfs --ost --index=10 --servicenode=10.1.10.10@o2ib --mgsnode=10.1.10.6@o2ib --mkfsoptions="-i 16384" --reformat /dev/sdd

mkfs.lustre --fsname=testfs --ost --index=11 --servicenode=10.1.10.10@o2ib --mgsnode=10.1.10.6@o2ib --mkfsoptions="-i 16384" --reformat /dev/sde

mkdir /mnt/ost08
mkdir /mnt/ost09
mkdir /mnt/ost10
mkdir /mnt/ost11

mount.lustre /dev/sdb /mnt/ost08
mount.lustre /dev/sdc /mnt/ost09
mount.lustre /dev/sdd /mnt/ost10
mount.lustre /dev/sde /mnt/ost11
```

在oss04上执行如下的格式化命令:
```bash
mkfs.lustre --fsname=testfs --ost --index=12 --servicenode=10.1.10.11@o2ib --mgsnode=10.1.10.6@o2ib --mkfsoptions="-i 16384" --reformat /dev/sdb

mkfs.lustre --fsname=testfs --ost --index=13 --servicenode=10.1.10.11@o2ib --mgsnode=10.1.10.6@o2ib --mkfsoptions="-i 16384" --reformat /dev/sdc

mkfs.lustre --fsname=testfs --ost --index=14 --servicenode=10.1.10.11@o2ib --mgsnode=10.1.10.6@o2ib --mkfsoptions="-i 16384" --reformat /dev/sdd

mkfs.lustre --fsname=testfs --ost --index=15 --servicenode=10.1.10.11@o2ib --mgsnode=10.1.10.6@o2ib --mkfsoptions="-i 16384" --reformat /dev/sde

mkdir /mnt/ost12
mkdir /mnt/ost13
mkdir /mnt/ost14
mkdir /mnt/ost15

mount.lustre /dev/sdb /mnt/ost12
mount.lustre /dev/sdc /mnt/ost13
mount.lustre /dev/sdd /mnt/ost14
mount.lustre /dev/sde /mnt/ost15
```
在6个节点上执行完毕，Lustre文件系统格式化结束，可以在客户端中使用
# 4.挂载使用Lustre
## 4.1 安装客户端
客户端的安装和服务端类似，只不过rpm包不同
在安装客户端的rpm包之前，也需要安装依赖，和服务端的类似，这里不再重复
具体步骤如下：
1、从官网下载rpm包
```
https://downloads.whamcloud.com/public/lustre/lustre-2.10.3-ib/MOFED-4.2-1.2.0.0/el7.4.1708/client/RPMS/x86_64/
```
2、执行下面的安装命令
```bash
rpm -Uvh kmod-lustre-client-2.10.3-1.el7.x86_64.rpm lustre-client-2.10.3-1.el7.x86_64.rpm
```
  
3、启动Lustre模块
参考2.5小结

## 4.2 挂载Lustre
执行下面的挂载命令即可挂载fsname为testfs的Lustre文件系统
```bash
mkdir /mnt/lustre
mount.lustre 10.1.10.6@o2ib:/testfs /mnt/lustre
```
>  可以通过df -h或者lfs df -h查看整个文件系统的容量

# Reference
[http://wiki.lustre.org](http://wiki.lustre.org)
[Lustre Manual](http://doc.lustre.org/lustre_manual.pdf)
