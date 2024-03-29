Table of Contents
=================

   * [0. 前言](#0-前言)
   * [1.给内核打Lustre patch](#1给内核打lustre-patch)
      * [1.1 前期准备](#11-前期准备)
         * [1.1.1 安装依赖包](#111-安装依赖包)
         * [1.1.2 创建编译用户](#112-创建编译用户)
         * [1.1.3 准备源码](#113-准备源码)
      * [1.2 编译内核(带Lustre patch)](#12-编译内核带lustre-patch)
         * [1.2.1 生成目录及.rpmmacros](#121-生成目录及rpmmacros)
         * [1.2.2 安装内核源码](#122-安装内核源码)
         * [1.2.3 使用rpmbuild准备内核源码](#123-使用rpmbuild准备内核源码)
         * [1.2.4 将Lustre补丁合并为一个文件](#124-将lustre补丁合并为一个文件)
         * [1.2.5 将patch应用到spec文件中](#125-将patch应用到spec文件中)
         * [1.2.6 设置参数](#126-设置参数)
         * [1.2.7 设置x86_64](#127-设置x86_64)
         * [1.2.8 编译kernel rpm包](#128-编译kernel-rpm包)
         * [1.2.9 保存编译产生的rpm包](#129-保存编译产生的rpm包)
      * [1.3 小结](#13-小结)
   * [2.编译Lustre](#2编译lustre)
      * [2.1 编译Lustre Server](#21-编译lustre-server)
      * [2.2 编译Lustre Client(CentOS环境)](#22-编译lustre-clientcentos环境)
      * [2.3 编译Lustre Client(Ubuntu环境)](#23-编译lustre-clientubuntu环境)
      * [2.4 小结](#24-小结)
   * [Reference](#reference)

# 0. 前言
Lustre代码运行环境是内核态，在使用Lustre之前需要对Kernel打patch，打patch并不会改变Kernel的主要代码流程，而是Lustre本身的功能所需要，比如quota功能、ldiskfs功能等。在大多数时候，我们直接使用社区的rpm包即可，使用社区编译好的Linux内核 [https://downloads.whamcloud.com/public/lustre](https://downloads.whamcloud.com/public/lustre)
![](./_image/2019-07-09/2019-07-15-17-05-34.png)

以及社区编译好的Lustre rpm包:
![](./_image/2019-07-09/2019-07-15-17-09-51.png)
![](./_image/2019-07-09/2019-07-15-17-09-38.png)

使用社区编译好的Linux内核、Lustre rpm包需要对Linux内核的小版本匹配上，如果不匹配，在安装的时候会报错。  

本文提供一种自己编译Linux内核、编译Lustre安装包的方式，这样就可以不严重依赖社区对应的Linux Kernel小版本，如果是Lustre新手，建议直接使用社区rpm包，参考另一篇文章《Lustre环境搭建——基于CentOS》

说明：Lustre Server节点需要打过patch的Kernel，Lustre Client不需要

# 1.给内核打Lustre patch
> 可以通过mock等方式编译，既可以跨平台，又不影响现有内核环境。本文为了简化流程，直接使用本机环境进行编译。

## 1.1 前期准备
### 1.1.1 安装依赖包
- 依赖包如下所示：
```bash
yum -y install automake xmlto asciidoc elfutils-libelf-devel zlib-devel binutils-devel newt-devel python-devel libyaml-devel hmaccalc perl-ExtUtils-Embed rpm-build make gcc redhat-rpm-config patchutils git libtool net-tools elfutils-devel bison audit-libs-deve pesign numactl-devel pciutils-devel ncurses-devel libselinux-devel
```

### 1.1.2 创建编译用户
> 后面如果没有特殊说明，均在build用户操作
```bash
sudo useradd -m build
```
### 1.1.3 准备源码
- 准备Lustre源码
```bash
git clone git://git.whamcloud.com/fs/lustre-release.git
```
- 切换到制定版本
```bash
git checkout 2.12.2
```
> 在本文的例子中，使用Lustre版本是2.12.2，对应的操作系统是CentOS7.6，内核是3.10.0-957.el7.x86_64，在[http://wiki.lustre.org/Lustre_2.12.2_Changelog](http://wiki.lustre.org/Lustre_2.12.2_Changelog)中有Lustre版本需要的操作系统以及内核  

## 1.2 编译内核(带Lustre patch)
### 1.2.1 生成目录及.rpmmacros
执行下面的操作:
```bash
mkdir -p ~/kernel/rpmbuild/{BUILD,RPMS,SOURCES,SPECS,SRPMS}
cd ~/kernel
echo '%_topdir %(echo $HOME)/kernel/rpmbuild' > ~/.rpmmacros
```

### 1.2.2 安装内核源码
执行下面的命令：
```bash
wget http://ftp.riken.jp/Linux/cern/centos/7/os/Sources/SPackages/kernel-3.10.0-957.el7.src.rpm
rpm -ivh kernel-3.10.0-957.el7.src.rpm
```

### 1.2.3 使用rpmbuild准备内核源码
执行下面的命令：
```bash
cd ~/kernel/rpmbuild
rpmbuild -bp --target=`uname -m` ./SPECS/kernel.spec
```
下面是正常情况下的输出:
```bash
Building target platforms: x86_64
Building for target x86_64
Executing(%prep): /bin/sh -e /var/tmp/rpm-tmp.XUFBjh
+ umask 022
+ cd /home/build/kernel/rpmbuild/BUILD
+ patch_command='patch -p1 -F1 -s'
+ cd /home/build/kernel/rpmbuild/BUILD
+ rm -rf kernel-3.10.0-957.el7
+ /usr/bin/mkdir -p kernel-3.10.0-957.el7
+ cd kernel-3.10.0-957.el7
+ /usr/bin/tar -xf -
+ /usr/bin/xz -dc /home/build/kernel/rpmbuild/SOURCES/linux-3.10.0-957.el7.tar.xz
+ STATUS=0
+ '[' 0 -ne 0 ']'
+ /usr/bin/chmod -Rf a+rX,u+w,g-w,o-w .
+ mv linux-3.10.0-957.el7 linux-3.10.0-957.el7.x86_64
+ cd linux-3.10.0-957.el7.x86_64
+ cp /home/build/kernel/rpmbuild/SOURCES/kernel-3.10.0-ppc64-debug.config /home/build/kernel/rpmbuild/SOURCES/kernel-3.10.0-ppc64.config /home/build/kernel/rpmbuild/SOURCES/kernel-3.10.0-ppc64le-debug.config /home/build/kernel/rpmbuild/SOURCES/kernel-3.10.0-ppc64le.config /home/build/kernel/rpmbuild/SOURCES/kernel-3.10.0-s390x-debug.config /home/build/kernel/rpmbuild/SOURCES/kernel-3.10.0-s390x-kdump.config /home/build/kernel/rpmbuild/SOURCES/kernel-3.10.0-s390x.config /home/build/kernel/rpmbuild/SOURCES/kernel-3.10.0-x86_64-debug.config /home/build/kernel/rpmbuild/SOURCES/kernel-3.10.0-x86_64.config .
+ ApplyOptionalPatch linux-kernel-test.patch
+ local patch=linux-kernel-test.patch
+ shift
+ '[' '!' -f /home/build/kernel/rpmbuild/SOURCES/linux-kernel-test.patch ']'
++ wc -l /home/build/kernel/rpmbuild/SOURCES/linux-kernel-test.patch
++ awk '{print $1}'
+ local C=0
+ '[' 0 -gt 9 ']'
+ ApplyOptionalPatch debrand-single-cpu.patch
+ local patch=debrand-single-cpu.patch
+ shift
+ '[' '!' -f /home/build/kernel/rpmbuild/SOURCES/debrand-single-cpu.patch ']'
++ wc -l /home/build/kernel/rpmbuild/SOURCES/debrand-single-cpu.patch
++ awk '{print $1}'
+ local C=25
+ '[' 25 -gt 9 ']'
+ ApplyPatch debrand-single-cpu.patch
+ local patch=debrand-single-cpu.patch
+ shift
+ '[' '!' -f /home/build/kernel/rpmbuild/SOURCES/debrand-single-cpu.patch ']'
Patch1000: debrand-single-cpu.patch
+ case "$patch" in
+ patch -p1 -F1 -s
+ ApplyOptionalPatch debrand-rh_taint.patch
+ local patch=debrand-rh_taint.patch
+ shift
+ '[' '!' -f /home/build/kernel/rpmbuild/SOURCES/debrand-rh_taint.patch ']'
++ wc -l /home/build/kernel/rpmbuild/SOURCES/debrand-rh_taint.patch
++ awk '{print $1}'
+ local C=25
+ '[' 25 -gt 9 ']'
+ ApplyPatch debrand-rh_taint.patch
+ local patch=debrand-rh_taint.patch
+ shift
+ '[' '!' -f /home/build/kernel/rpmbuild/SOURCES/debrand-rh_taint.patch ']'
Patch1001: debrand-rh_taint.patch
+ case "$patch" in
+ patch -p1 -F1 -s
+ ApplyOptionalPatch debrand-rh-i686-cpu.patch
+ local patch=debrand-rh-i686-cpu.patch
+ shift
+ '[' '!' -f /home/build/kernel/rpmbuild/SOURCES/debrand-rh-i686-cpu.patch ']'
++ wc -l /home/build/kernel/rpmbuild/SOURCES/debrand-rh-i686-cpu.patch
++ awk '{print $1}'
+ local C=11
+ '[' 11 -gt 9 ']'
+ ApplyPatch debrand-rh-i686-cpu.patch
+ local patch=debrand-rh-i686-cpu.patch
+ shift
+ '[' '!' -f /home/build/kernel/rpmbuild/SOURCES/debrand-rh-i686-cpu.patch ']'
Patch1002: debrand-rh-i686-cpu.patch
+ case "$patch" in
+ patch -p1 -F1 -s
+ chmod +x scripts/checkpatch.pl
+ touch .scmversion
+ '[' -L configs ']'
+ rm -f configs
+ mkdir configs
+ for cfg in 'kernel-3.10.0-*.config'
++ echo kernel-3.10.0-x86_64-debug.config kernel-3.10.0-x86_64.config
++ grep -c kernel-3.10.0-ppc64-debug.config
+ '[' 0 -eq 0 ']'
+ rm -f kernel-3.10.0-ppc64-debug.config
+ for cfg in 'kernel-3.10.0-*.config'
++ echo kernel-3.10.0-x86_64-debug.config kernel-3.10.0-x86_64.config
++ grep -c kernel-3.10.0-ppc64.config
+ '[' 0 -eq 0 ']'
+ rm -f kernel-3.10.0-ppc64.config
+ for cfg in 'kernel-3.10.0-*.config'
++ echo kernel-3.10.0-x86_64-debug.config kernel-3.10.0-x86_64.config
++ grep -c kernel-3.10.0-ppc64le-debug.config
+ '[' 0 -eq 0 ']'
+ rm -f kernel-3.10.0-ppc64le-debug.config
+ for cfg in 'kernel-3.10.0-*.config'
++ echo kernel-3.10.0-x86_64-debug.config kernel-3.10.0-x86_64.config
++ grep -c kernel-3.10.0-ppc64le.config
+ '[' 0 -eq 0 ']'
+ rm -f kernel-3.10.0-ppc64le.config
+ for cfg in 'kernel-3.10.0-*.config'
++ echo kernel-3.10.0-x86_64-debug.config kernel-3.10.0-x86_64.config
++ grep -c kernel-3.10.0-s390x-debug.config
+ '[' 0 -eq 0 ']'
+ rm -f kernel-3.10.0-s390x-debug.config
+ for cfg in 'kernel-3.10.0-*.config'
++ grep -c kernel-3.10.0-s390x-kdump.config
++ echo kernel-3.10.0-x86_64-debug.config kernel-3.10.0-x86_64.config
+ '[' 0 -eq 0 ']'
+ rm -f kernel-3.10.0-s390x-kdump.config
+ for cfg in 'kernel-3.10.0-*.config'
++ grep -c kernel-3.10.0-s390x.config
++ echo kernel-3.10.0-x86_64-debug.config kernel-3.10.0-x86_64.config
+ '[' 0 -eq 0 ']'
+ rm -f kernel-3.10.0-s390x.config
+ for cfg in 'kernel-3.10.0-*.config'
++ grep -c kernel-3.10.0-x86_64-debug.config
++ echo kernel-3.10.0-x86_64-debug.config kernel-3.10.0-x86_64.config
+ '[' 1 -eq 0 ']'
+ for cfg in 'kernel-3.10.0-*.config'
++ echo kernel-3.10.0-x86_64-debug.config kernel-3.10.0-x86_64.config
++ grep -c kernel-3.10.0-x86_64.config
+ '[' 1 -eq 0 ']'
+ for i in '*.config'
+ mv kernel-3.10.0-x86_64-debug.config .config
++ cut -b 3-
++ head -1 .config
+ Arch=x86_64
+ make ARCH=x86_64 listnewconfig
+ grep -E '^CONFIG_'
+ true
+ '[' -s .newoptions ']'
+ rm -f .newoptions
+ make ARCH=x86_64 oldnoconfig
scripts/kconfig/conf --olddefconfig Kconfig
#
# configuration written to .config
#
+ echo '# x86_64'
+ cat .config
+ for i in '*.config'
+ mv kernel-3.10.0-x86_64.config .config
++ head -1 .config
++ cut -b 3-
+ Arch=x86_64
+ make ARCH=x86_64 listnewconfig
+ grep -E '^CONFIG_'
+ true
+ '[' -s .newoptions ']'
+ rm -f .newoptions
+ make ARCH=x86_64 oldnoconfig
scripts/kconfig/conf --olddefconfig Kconfig
#
# configuration written to .config
#
+ echo '# x86_64'
+ cat .config
+ find . '(' -name '*.orig' -o -name '*~' ')' -exec rm -f '{}' ';'
+ find . -name .gitignore -exec rm -f '{}' ';'
+ cd ..
+ exit 0
```
> 现在获取了CentOS的源代码，路径在～/kernel/rpmbuild/BUILD/kernel-3.10.0-957.el7/linux-3.10.0-957.el7.x86_64  

### 1.2.4 将Lustre补丁合并为一个文件
执行下面的命令:
```bash
_TOPDIR=`rpm --eval %{_topdir}`
for i in `cat $HOME/lustre-release/lustre/kernel_patches/series/3.10-rhel7.6.series`; do
cat $HOME/lustre-release/lustre/kernel_patches/patches/$i
done > $_TOPDIR/SOURCES/patch-lustre.patch
```
>  这里有个坑，注意lustre-release/lustre/kernel_patches/series/3.10-rhel7.6.series文件，不同的lustre版本的series文件也不同

现在所有的lustre补丁都在patch-lustre.patch这个文件中
![](./_image/2019-07-09/2019-07-15-20-14-54.png)

### 1.2.5 将patch应用到spec文件中
执行下面的命令：
```bash
_TOPDIR=`rpm --eval %{_topdir}`
sed -i.inst -e '/find $RPM_BUILD_ROOT\/lib\/modules\/$KernelVer/a\
    cp -a fs/ext3/* $RPM_BUILD_ROOT/lib/modules/$KernelVer/build/fs/ext3 \
    cp -a fs/ext4/* $RPM_BUILD_ROOT/lib/modules/$KernelVer/build/fs/ext4' \
-e '/^# empty final patch to facilitate testing of kernel patches/i\
Patch99995: patch-lustre.patch' \
-e '/^ApplyOptionalPatch linux-kernel-test.patch/i\
ApplyOptionalPatch patch-lustre.patch' \
-e '/^%define listnewconfig_fail 1/s/1/0/' \
$_TOPDIR/SPECS/kernel.spec
```
执行结束以后，所有的patch都应用到kernel.spec这个文件中

### 1.2.6 设置参数
参数值为：（如果使用ldiskfs，建议修改为对应的值）
```bash
CONFIG_FUSION_MAX_SGE=256
CONFIG_SCSI_MAX_SG_SEGMENTS=128
```
执行下面的命令：
```bash
_TOPDIR=`rpm --eval %{_topdir}`
sed -i.inst -e 's/\(CONFIG_FUSION_MAX_SGE=\).*/\1256/' \
-e 's/\(CONFIG_SCSI_MAX_SG_SEGMENTS\)/\1128/' \
$_TOPDIR/SOURCES/kernel-3.10.0-x86_64.config
! `grep -q CONFIG_SCSI_MAX_SG_SEGMENTS $_TOPDIR/SOURCES/kernel-3.10.0-x86_64.config.inst` && \
echo "CONFIG_SCSI_MAX_SG_SEGMENTS=128" >> $_TOPDIR/SOURCES/kernel-3.10.0-x86_64.config
```

### 1.2.7 设置x86_64
需要确保第一行是x86_64，执行下面的命令：
```bash
_TOPDIR=`rpm --eval %{_topdir}`
echo '# x86_64' > $_TOPDIR/SOURCES/kernel-3.10.0-x86_64.config
cat $HOME/lustre-release/lustre/kernel_patches/kernel_configs/kernel-3.10.0-3.10-rhel7.6-x86_64.config >> $_TOPDIR/SOURCES/kernel-3.10.0-x86_64.config
```

### 1.2.8 编译kernel rpm包
执行下面的命令：
```bash
_TOPDIR=`rpm --eval %{_topdir}`
rpmbuild -ba --with firmware --with baseonly \
--without debuginfo \
--without kabichk \
--define "buildid _lustre" \
--target x86_64 \
$_TOPDIR/SPECS/kernel.spec
```
> --without debuginfo这个参数是不生成debuginfo的包
> --without kabichk这个参数是关闭KABI验证

### 1.2.9 保存编译产生的rpm包
执行下面的命令：
```bash
_TOPDIR=`rpm --eval %{_topdir}`
mkdir -p $HOME/releases/lustre-kernel
mv $_TOPDIR/RPMS/*/{kernel-*,python-perf-*,perf-*} $HOME/releases/lustre-kernel
```
如果一切正常，会产生下面的rpm包：
```bash
[build@host lustre-kernel]$ ll
total 119520
-rw-rw-r-- 1 build build 50488648 Jul 15 20:33 kernel-3.10.0-957.el7_lustre.x86_64.rpm
-rw-rw-r-- 1 build build 24576492 Jul 15 20:33 kernel-devel-3.10.0-957.el7_lustre.x86_64.rpm
-rw-rw-r-- 1 build build  8354704 Jul 15 20:33 kernel-headers-3.10.0-957.el7_lustre.x86_64.rpm
-rw-rw-r-- 1 build build  7444040 Jul 15 20:33 kernel-tools-3.10.0-957.el7_lustre.x86_64.rpm
-rw-rw-r-- 1 build build  7346772 Jul 15 20:33 kernel-tools-libs-3.10.0-957.el7_lustre.x86_64.rpm
-rw-rw-r-- 1 build build  7333640 Jul 15 20:33 kernel-tools-libs-devel-3.10.0-957.el7_lustre.x86_64.rpm
-rw-rw-r-- 1 build build  9066448 Jul 15 20:33 perf-3.10.0-957.el7_lustre.x86_64.rpm
-rw-rw-r-- 1 build build  7760224 Jul 15 20:33 python-perf-3.10.0-957.el7_lustre.x86_64.rpm
```
这里产生的rpm包和社区一样，是打了patch的内核

## 1.3 小结
这一章主要完成给Kernel打patch的工作，最终产生的kernel包带有xx_lustre后缀。

# 2.编译Lustre
## 2.1 编译Lustre Server
- 安装依赖
```bash
yum -y install net-tools vim epel-release xmlto asciidoc elfutils-libelf-devel zlib-devel binutils-devel newt-devel python-devel hmaccalc perl-ExtUtils-Embed bison elfutils-devel audit-libs-devel python-docutils sg3_utils expect attr lsof quilt libselinux-devel libtool linux-firmware xfsprogs kmod libyaml libyaml-devel
```
>  如果kernel没有问题，IB驱动也没问题，编译是不会出现问题，一般都是缺少相关依赖，缺少什么直接安装就行

- 执行编译
```bash
$ sh autogen.sh
$ ./configure --enable-server  --with-linux=/usr/src/kernels/3.10.0-957.el7_lustre.x86_64 --with-o2ib=/usr/src/ofa_kernel/default
$ make rpms -j $(nproc)
```
>  这里--enable-server参数也就是只编译Lustre Server，如果不设置这个参数，默认客户端和Server端都编译，因为Client不依赖内核版本，所以这里开启编译Server的选项
>  这里的网络使用Mellanox OFED，所以使用参数--with-o2ib

- 编译后的结果
```bash
[root@host lustre-rpm]# ll
总用量 63532
-rw-r--r--. 1 root root  4063284 7月  11 20:38 kmod-lustre-2.12.2-1.el7.x86_64.rpm
-rw-r--r--. 1 root root   487928 7月  11 20:38 kmod-lustre-osd-ldiskfs-2.12.2-1.el7.x86_64.rpm
-rw-r--r--. 1 root root    45392 7月  11 20:38 kmod-lustre-tests-2.12.2-1.el7.x86_64.rpm
-rw-r--r--. 1 root root   718360 7月  11 20:38 lustre-2.12.2-1.el7.x86_64.rpm
-rw-r--r--. 1 root root 14571134 7月  11 20:38 lustre-2.12.2-1.src.rpm
-rw-r--r--. 1 root root 36895420 7月  11 20:38 lustre-debuginfo-2.12.2-1.el7.x86_64.rpm
-rw-r--r--. 1 root root    41652 7月  11 20:38 lustre-iokit-2.12.2-1.el7.x86_64.rpm
-rw-r--r--. 1 root root    13896 7月  11 20:38 lustre-osd-ldiskfs-mount-2.12.2-1.el7.x86_64.rpm
-rw-r--r--. 1 root root     7572 7月  11 20:38 lustre-resource-agents-2.12.2-1.el7.x86_64.rpm
-rw-r--r--. 1 root root  8186508 7月  11 20:38 lustre-tests-2.12.2-1.el7.x86_64.rpm
```
>  这里编译后的包和社区提供的rpm包相同，可以按照《Lustre环境搭建》中

## 2.2 编译Lustre Client(CentOS环境)
Lustre Client对内核没有特殊的要求，因此不需要替换Linux内核，直接执行下面的编译命令：
```bash
$ sh autogen.sh
$ ./configure --disable-server --enable-client --with-linux=/usr/src/kernels/3.10.0-693.el7.x86_64 --with-o2ib=/usr/src/ofa_kernel/default
$ make rpms -j $(nproc)
```
>  需要提前安装依赖，在前面的章节中有所体现，编译遇到的常见问题大都是缺少相关的依赖

## 2.3 编译Lustre Client(Ubuntu环境)
本小结针对Ubuntu 16.04编译Lustre Client

- 安装依赖
```bash
$ apt-get install -y make dpkg-dev debhelper dh-virtualenv libsnmp-dev quilt dpatch libreadline-dev module-assistant libyaml-dev libtool m4 automake autoconf make 
```

- 针对IB环境的编译命令
```bash
sh autogen.sh
./configure --disable-server --with-linux=/usr/src/linux-headers-4.4.0-31-generic --with-o2ib=/usr/src/ofa_kernel/default
make debs -j $(nproc)
```
>  --with-linux 需要改成自己OS对应的路径

- 针对TCP环境的编译命令
```bash
sh autogen.sh
./configure --disable-server --with-linux=/usr/src/linux-headers-4.4.0-31-generic --with-o2ib=no                                       
make debs -j $(nproc)
```

- 编译后的deb包如下所示：
```bash
查看debs目录下的deb包:
root@vm:~/makedebs/lustre-release-tcp/debs# ll
total 167612
drwxr-xr-x  2 root root      4096 May  6 14:03 ./
drwxr-xr-x 16 root root      4096 May  6 14:03 ../
-rw-r--r--  1 root root    117820 May  6 13:58 linux-patch-lustre_2.10.3-1_all.deb
-rw-r--r--  1 root root      2662 May  6 13:59 lustre_2.10.3-1_amd64.changes
-rw-r--r--  1 root root      1236 May  6 13:57 lustre_2.10.3-1.dsc
-rw-r--r--  1 root root  14296440 May  6 13:57 lustre_2.10.3-1.tar.gz
-rw-r--r--  1 root root  15635292 May  6 14:03 lustre-client-modules-4.4.0-31-generic_2.10.3-1_amd64.deb
-rw-r--r--  1 root root    482284 May  6 13:59 lustre-dev_2.10.3-1_amd64.deb
-rw-r--r--  1 root root 132961380 May  6 13:59 lustre-source_2.10.3-1_all.deb
-rw-r--r--  1 root root   7626114 May  6 13:59 lustre-tests_2.10.3-1_amd64.deb
-rw-r--r--  1 root root    485136 May  6 13:59 lustre-utils_2.10.3-1_amd64.deb
```

## 2.4 小结
在编译Lustre过程中遇到的最多问题是缺少相关的依赖，或者依赖的版本不对，根据错误提示去操作即可。如果遇到乱七八糟的问题，找一个干净的环境，最好是刚装完系统的环境，按照这个教程一步一步操作，就可以完成Lustre的所有编译操作。我本人在刚开始接触Lustre时，编译出现各种幺蛾子，导致编译出现各种问题，为此还专门请教过社区的大佬，现在回想起来，自己还是有很大的成长空间。

# Reference
[Lustre并行文件系统源码安装配置](http://hmli.ustc.edu.cn/doc/linux/lustre/)
[http://wiki.lustre.org/Compiling_Lustre](http://wiki.lustre.org/Compiling_Lustre)