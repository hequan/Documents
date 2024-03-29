Table of Contents
=================

   * [1. 概述](#1-概述)
   * [2. VFS与ext4](#2-vfs与ext4)
      * [2.1 sys_read()](#21-sys_read)
      * [2.2 vfs_read()](#22-vfs_read)
         * [2.2.1 rw_verify_area()](#221-rw_verify_area)
      * [2.3 do_sync_read()](#23-do_sync_read)
         * [2.3.1 异步I/O](#231-异步io)
         * [2.3.2 do_sync_read()函数](#232-do_sync_read函数)
      * [2.4 generic_file_aio_read()](#24-generic_file_aio_read)
      * [2.5 do_generic_file_read()](#25-do_generic_file_read)
         * [2.5.1 address_space -&gt; readpage()方法](#251-address_space---readpage方法)
         * [2.5.2 file_read_actor()](#252-file_read_actor)
         * [2.5.3 mpage_readpage()](#253-mpage_readpage)
      * [2.8 mpage_bio_submit()](#28-mpage_bio_submit)
      * [2.9 小结](#29-小结)
   * [3. 读数据返回](#3-读数据返回)
      * [3.1 读进程的阻塞和继续执行过程](#31-读进程的阻塞和继续执行过程)
      * [3.2 读数据返回过程](#32-读数据返回过程)


# 1. 概述
我们对系统调用read()非常熟悉，也常听说“零拷贝”。在看Linux内核源码时，有很多人会有一些困惑，比如：
- 读文件的整个流程是怎样的?
- 内核是如何Cache已经读取的文件数据?
-  驱动从磁盘上读取的数据是否会直接写到用户的缓冲区中?
- 内核是在哪个地方分配空间来存储将要读取的数据?
- 是在哪个地方将当前进程阻塞，直至读取数据结束?
- “零拷贝”是如何实现 的?
本文以CentOS 7.6 内核版本3.10.0-957.12.1.el7.x86_64为例，分析从用户进程通过read()读取文件，直至数据返回给用户的整个流程。  

在我们分析源码之前，仍要回顾一下内核中块设备操作的流程，如下图所示：
![](./_image/2019-09-19/2019-09-30-20-14-09.png)
对于一个进程使用系统调用read()读取磁盘上的文件。下面步骤是内核响应进程读请求的步骤：
- (1)  系统调用read()会触发相应的VFS(Virtual Filesystem Switch)函数，传递的参数有文件描述符和文件偏移量
- (2)  VFS确定请求的数据是否已经在内存缓冲区中; 若数据不在内存中，确定如何执行读操作
- (3)  假设内核必须从块设备上读取数据，这样内核就必须确定数据在物理设备上的位置, 这由映射层(Mapping Layer)来完成
- (4)  此时内核通过通用块设备层(Generic Block Layer)在块设备上执行读操作，启动I/O操作， 传输请求的数据
- (5)  在通用块设备层之下是I/O调度层(I/O Scheduler Layer)，根据内核的调度策略，对等待 的I/O等待队列排序
- (6)  最后，块设备驱动(Block Device Driver)通过向磁盘控制器发送相应的命令，执行真正 的数据传输
  
本文分析read()系统调用的整个过程，包括VFS、通用块设备层、I/O调度层、块设备驱 动等内容。  

# 2. VFS与ext4
在分析内核源码之前，回顾一下read()函数原型:
`ssize_t read(unsigned int fd, char * buf, size_t count)`
read()函数是从打开的文件中读取数据。如read()成功，则返回读到的字节数。如已到 达文件的尾端，则返回0。如果失败，则返回-1。
有多种情况可使实际读到的字节数少于要求读字节数:
- 读普通文件时，在读到要求字节数之前已到达了文件尾端。例如，若在到达文件尾端之 前还有50个字节，而要求读200个字节，则read返回50，下一次再调用read时，它将返回0 (文 件尾端)
- 当从终端设备读时，通常一次最多读一行
- 当从网络读时，网络中的缓冲机构可能造成返回值小于所要求读的字节数
- 某些面向记录的设备，例如磁带，一次最多返回一个记录

总结来说：
读操作从文件的当前位移量处开始，在成功返回之前，该位移量增加实际读得的字节数

## 2.1 sys_read()
sys_read()是系统调用read()的实现，源码在文件fs/read_write.c中
```c
SYSCALL_DEFINE3(read, unsigned int, fd, char __user *, buf, size_t, count)
{
	struct fd f = fdget_pos(fd);
	ssize_t ret = -EBADF;

	if (f.file) {
		loff_t pos = file_pos_read(f.file);
		ret = vfs_read(f.file, buf, count, &pos);
		if (ret >= 0)
			file_pos_write(f.file, pos);
		fdput_pos(f);
	}
	return ret;
}
```
在sys_read()函数中，实现主体是vfs_read()，也就是read()函数的实现主体。
 file_pos_read()和file_pos_write()的功能单一，分别是读取/更新当前文件的读写位置;
 file_pos_write()是根据读文件结果，更新文件读写位置。
这两个函数源码均在fs/read_write.c 中。
fdget_pos()返回打开文件的file结构。
若files_struct为多进程/线程共享(fput_neede=1)，fdput_pos()就调用fput()减少共享 计数。

## 2.2 vfs_read()
sys_read()是系统调用read()的实现，实现主体是vfs_read()，也就是read()函数 的实现主体。源码在文件fs/read_write.c中。
该函数有4个参数
- file是已打开文件的file结构，该结构是由fd而来;
- buf是用户空间的缓冲区，就是内核将读取到的数据拷贝到这里;
- count是将要读取的字节数;
- pos是文件当前读写位置。
```c
ssize_t vfs_read(struct file *file, char __user *buf, size_t count, loff_t *pos)
{
	ssize_t ret;

	if (!(file->f_mode & FMODE_READ))
		return -EBADF;
	if (!file->f_op || (!file->f_op->read && !file->f_op->aio_read))
		return -EINVAL;
	if (unlikely(!access_ok(VERIFY_WRITE, buf, count)))
		return -EFAULT;

	ret = rw_verify_area(READ, file, pos, count);
	if (ret >= 0) {
		count = ret;
		if (file->f_op->read)
			ret = file->f_op->read(file, buf, count, pos);
		else
			ret = do_sync_read(file, buf, count, pos);
		if (ret > 0) {
			fsnotify_access(file);
			add_rchar(current, ret);
		}
		inc_syscr(current);
	}

	return ret;
}
```
在后面我们会看到vfs_read()函数调用的是do_sync_read()，可以理解为vfs_read() 就是对do_sync_read()函数的封装，在调用do_sync_read()函数之前进行一些合法性检查。
第一次检查是检查文件的访问模式，若不允许读，则返回-EBADF；
第二次检查是在执行真正的读操作之前，文件操作表file->f_op不能为空，并且文件操作表中read或aio_read方法必须要有一个有实现; 否则就返回-EINVAL；
第三次的access_ok()函数检查用户态的buf是否有效，VERIFY_WRITE表示用户buf是否可写。VERIFY_WRITE的优先级高于VERIFY_READ，因为用户的buf可写的话，必然可读
rw_verify_area()检查要访问的文件部分是否有冲突的强制锁，通过inode结构lock当前要操作的区域来判断。文件系统是否允许使用强制锁，可以在mount时指定，如果mount的时候指定了 MS_MANDLOCK，则允许使用强制锁。
### 2.2.1 rw_verify_area()
rw_verify_area()的具体实现为：
```c
/*
 * rw_verify_area doesn't like huge counts. We limit
 * them to something that fits in "int" so that others
 * won't have to do range checks all the time.
 */
int rw_verify_area(int read_write, struct file *file, loff_t *ppos, size_t count)
{
	struct inode *inode;
	loff_t pos;
	int retval = -EINVAL;

	inode = file_inode(file);
	if (unlikely((ssize_t) count < 0))
		return retval;
	pos = *ppos;
	if (unlikely(pos < 0)) {
		if (!unsigned_offsets(file))
			return retval;
		if (count >= -pos) /* both values are in 0..LLONG_MAX */
			return -EOVERFLOW;
	} else if (unlikely((loff_t) (pos + count) < 0)) {
		if (!unsigned_offsets(file))
			return retval;
	}

	if (unlikely(inode->i_flock && mandatory_lock(inode))) {
		retval = locks_mandatory_area(inode, file, pos, pos + count - 1,
				read_write == READ ? F_RDLCK : F_WRLCK);
		if (retval < 0)
			return retval;
	}
	retval = security_file_permission(file,
				read_write == READ ? MAY_READ : MAY_WRITE);
	if (retval)
		return retval;
	return count > MAX_RW_
```
通过上面合法性检查后就是读操作本身了。可想而知，不同的文件系统有不同的读操作， 具体的文件系统通过file_operations结构提供用于读操作的函数指针。如果读操作的返回值大于 0，说明读数据成功，且ret值为读取的字节数。则调用fsnotify_access()通知文件被读取， add_rchar()增加当前进程读取的字节数。
inc_syscr()函数的作用是增加当前进程read系统调用的次数。

本文以ext4的实现为例，来看一下ext4文件系统的file_operations结构，其定义在 fs/ext4/file.c文件中。
可以看到ext4文件系统中file->f_op->read的实现方法为do_sync_read()
```c
const struct file_operations_extend  ext4_file_operations = {
	.kabi_fops = {
		.llseek		= ext4_llseek,
		.read		= do_sync_read,
		.write		= do_sync_write,
		.aio_read	= ext4_file_read,
		.aio_write	= ext4_file_write,
		.unlocked_ioctl = ext4_ioctl,
#ifdef CONFIG_COMPAT
		.compat_ioctl	= ext4_compat_ioctl,
#endif
		.mmap		= ext4_file_mmap,
		.open		= ext4_file_open,
		.release	= ext4_release_file,
		.fsync		= ext4_sync_file,
		.get_unmapped_area = thp_get_unmapped_area,
		.splice_read	= generic_file_splice_read,
		.splice_write	= generic_file_splice_write,
		.fallocate	= ext4_fallocate,
	},
	.mmap_supported_flags = MAP_SYNC,
};
```
## 2.3 do_sync_read()
在具体描述do_sync_read()操作之前，先来看一下同步I/O和异步I/O
Linux中最常用的输入/输出(I/O)模型是同步 I/O。在这个模型中，当请求发出之后，应用程序就会阻塞，直到请求满足为止。同步I/O场景下，调用应用程序在等待 I/O 请求完成时不需要使用任何CPU执行。
但是在某些情况中，I/O 请求可能需要与其他进程产生交叠。可移植 操作系统接口(POSIX)异步 I/O应用程序接口就提供了这种功能。
### 2.3.1 异步I/O
异步I/O背后的基本思想是允许进程发起很多 I/O 操作，而不用阻塞或等待任何操作完成。稍后或在接收到 I/O 操作完成的通知时，进程就可以检索 I/O 操作的结果。异步 I/O 允 许用户空间来初始化操作而不必等待它们的完成; 因此， 一个应用程序可以在它的 I/O 在进 行中时做其他的处理。一个复杂的、高性能的应用程序还可使用异步 I/O 来使多个操作在同一个时间进行。
实现异步I/O操作的file_operations方法，都使用kiocb控制块(I/O Control Block)。其定义 在include/linux/aio.h文件中。
```c

struct kiocb {
	atomic_t		ki_users;

	struct file		*ki_filp;
	struct kioctx		*ki_ctx;	/* NULL for sync ops */
	kiocb_cancel_fn		*ki_cancel;
	void			(*ki_dtor)(struct kiocb *);

	union {
		void __user		*user;
		struct task_struct	*tsk;
	} ki_obj;

	__u64			ki_user_data;	/* user's data for completion */
	loff_t			ki_pos;

	void			*private;
	/* State that we remember to be able to restart/retry  */
	unsigned short		ki_opcode;
	size_t			ki_nbytes; 	/* copy of iocb->aio_nbytes */
	char 			__user *ki_buf;	/* remaining iocb->aio_buf */
	size_t			ki_left; 	/* remaining bytes */
	struct iovec		ki_inline_vec;	/* inline vector */
 	struct iovec		*ki_iovec;
 	unsigned long		ki_nr_segs;
 	unsigned long		ki_cur_seg;

	struct list_head	ki_list;	/* the aio core uses this
						 * for cancellation */

	/*
	 * If the aio_resfd field of the userspace iocb is not zero,
	 * this is the underlying eventfd context to deliver events to.
	 */
	struct eventfd_ctx	*ki_eventfd;
};
```
- ki_user_data:  返回给用户进程的值;
- ki_pos:  当前文件读写位置; 
- ki_opcode: 操作类型(read、write或者sync); 
- ki_nbytes:将要传输的字节数;
-  ki_buf:用户空间缓存区当前位置; 
- ki_left:还未完成传输的字节数; 
- private:文件系统层使用

kiocb结构的意义在于使能异步操作。若驱动能够初始化这个操作(或者简单地，将它排队 到它能够被执行时)，它必须做两件事情:
- (1)记录它需要知道的关于这个操作的所有东西， 并且返回 -EIOCBQUEUED 给调 用者。
- (2)记录操作信息包括安排对用户空间缓冲的存取;当在调用进程的上下文运行时， 一旦返回，将不能再来存取缓冲。

### 2.3.2 do_sync_read()函数
do_sync_read()函数的实现主体在文件fs/read_write.c中。
```c
ssize_t do_sync_read(struct file *filp, char __user *buf, size_t len, loff_t *ppos)
{
	struct iovec iov = { .iov_base = buf, .iov_len = len };
	struct kiocb kiocb;
	ssize_t ret;

	init_sync_kiocb(&kiocb, filp);
	kiocb.ki_pos = *ppos;
	kiocb.ki_left = len;
	kiocb.ki_nbytes = len;

	ret = filp->f_op->aio_read(&kiocb, &iov, 1, kiocb.ki_pos);
	if (-EIOCBQUEUED == ret)
		ret = wait_on_sync_kiocb(&kiocb);
	*ppos = kiocb.ki_pos;
	return ret;
}
```
函数中有个临时变量kiocb，它是用来跟踪记录即将进行I/O操作的完成状态。通过宏 init_sync_kiocb()来初始化kiocb，设置对象为同步操作。init_sync_kiocb()函数的实现就不展开了。
对于ext4文件系统，file->f_op->read的实现方法为ext4_file_read()(见 ext4_file_operations定义)
ext4_file_read()函数的实现为：
```c
static ssize_t
ext4_file_read(
	struct kiocb		*iocb,
	const struct iovec	*iovp,
	unsigned long		nr_segs,
	loff_t 			pos)
{
#ifdef CONFIG_FS_DAX
	if (IS_DAX(file_inode(iocb->ki_filp)))
		return ext4_file_dax_read(iocb, iovp, nr_segs, pos);
#endif
	return generic_file_aio_read(iocb, iovp, nr_segs, pos);
}
```
ext4_file_read是对generic_file_aio_read的包装，如果有CONFIG_FS_DAX参数，会调用ext4_file_dax_read，否则还是走generic_file_aio_read。
CONFIG_FS_DAX的意思是： Direct Access (DAX) support，具体细节可以去搜索引擎查找

## 2.4 generic_file_aio_read()
generic_file_aio_read()函数实现在文件mm/filemap.c中。该函数是read()系统调用执 行主体，且所有文件系统共用。该函数实现同步和异步的读操作。
函数有4个参数，iocb和iov在前面一节已作介绍，nr_segs的值事实上为1(do_sync_read ()参数传递)，ppos是用来保存文件当前读写位置。
```c
/**
 * generic_file_aio_read - generic filesystem read routine
 * @iocb:	kernel I/O control block
 * @iov:	io vector request
 * @nr_segs:	number of segments in the iovec
 * @pos:	current file position
 *
 * This is the "read()" routine for all filesystems
 * that can use the page cache directly.
 */
ssize_t
generic_file_aio_read(struct kiocb *iocb, const struct iovec *iov,
		unsigned long nr_segs, loff_t pos)
{
	struct file *filp = iocb->ki_filp;
	ssize_t retval;
	unsigned long seg = 0;
	size_t count;
	loff_t *ppos = &iocb->ki_pos;

	count = 0;
	retval = generic_segment_checks(iov, &nr_segs, &count, VERIFY_WRITE);
	if (retval)
		return retval;
```
generic_segment_checks()用来验证iovec描述的用户空间缓冲区是 有效的。因为读文件的起始地址和大小都是由sys_read()传递过来，在使用它们之前，必须 要先进行合法性检查。若参数无效，则返回-EFAULT。  
```c
if (io_is_direct(filp)) {
		loff_t size;
		struct address_space *mapping;
		struct inode *inode;

		mapping = filp->f_mapping;
		inode = mapping->host;
		if (!count)
			goto out; /* skip atime */
		size = i_size_read(inode);
		if (pos < size) {
			retval = filemap_write_and_wait_range(mapping, pos,
					pos + iov_length(iov, nr_segs) - 1);
			if (!retval) {
				retval = mapping->a_ops->direct_IO(READ, iocb,
							iov, pos, nr_segs);
			}
			if (retval > 0) {
				*ppos = pos + retval;
				count -= retval;
			}

			/*
			 * Btrfs can have a short DIO read if we encounter
			 * compressed extents, so if there was an error, or if
			 * we've already read everything we wanted to, or if
			 * there was a short read because we hit EOF, go ahead
			 * and return.  Otherwise fallthrough to buffered io for
			 * the rest of the read.
			 */
			if (retval < 0 || !count || *ppos >= size) {
				file_accessed(filp);
				goto out;
			}
		}
       /*
		 *  Buffered reads will not work for DAX files, so don't
		 *  bother trying.
		 */
		if (IS_DAX(inode)) {
			file_accessed(filp);
			goto out;
		}
	}
```
当执行direct IO读操作时，则进入直读模式，也就是跳过页缓存。
这里我们暂时不关注direct IO，而是关注最常用的情形:异步读取基于页面缓存的文件。也就是会执行下面的代码。
```c
count = retval;
	for (seg = 0; seg < nr_segs; seg++) {
		read_descriptor_t desc;
		loff_t offset = 0;

		/*
		 * If we did a short DIO read we need to skip the section of the
		 * iov that we've already read data into.
		 */
		if (count) {
			if (count > iov[seg].iov_len) {
				count -= iov[seg].iov_len;
				continue;
			}
			offset = count;
			count = 0;
		}

		desc.written = 0;
		desc.arg.buf = iov[seg].iov_base + offset;
		desc.count = iov[seg].iov_len - offset;
		if (desc.count == 0)
			continue;
		desc.error = 0;
		do_generic_file_read(filp, ppos, &desc, file_read_actor);
		retval += desc.written;
		if (desc.error) {
			retval = retval ?: desc.error;
			break;
		}
		if (desc.count > 0)
			break;
	}
```
定义一个read_descriptor_t类型的临时变量desc，该临时变量保 存将要进行读操作的状态，该状态和用户空间缓冲区有关。这里再提出iov[seg].iov_base的 值，就是sys_read(unsigned int fd, char __user * buf, size_t count)中的第2个参数的值; iov[seg].iov_len值是要读取的字节数count。
初始化临时变量desc后，就调用函数do_generic_file_read()，传递的参数包括文件对象指针filp，文件偏移量指针ppos，desc地址和函数file_read_actor()地址。
generic_file_aio_read()函数返回值是拷贝到用户缓冲区的字节数，这个值就是 read_descriptor_t结构体中written的值。
>  read_descriptor_t数据结构
read_descriptor_t数据结构定义在文件include/linux/fs.h中。
```c
/*
 * "descriptor" for what we're up to with a read.
 * This allows us to use the same read code yet
 * have multiple different users of the data that
 * we read from a file.
 *
 * The simplest case just copies the data to user
 * mode.
 */
typedef struct {
	size_t written;
	size_t count;
	union {
		char __user *buf;
		void *data;
	} arg;
	int error;
} read_descriptor_t;
```
各成员变量含义如下: 
- written:已拷贝到用户缓冲区的字节数; 
- count:待传输的字节数; 
- arg.buf:用户缓冲区的当前位置; 
- error:读操作的错误码(0表示没有错误)

## 2.5 do_generic_file_read()
do_generic_file_read()函数的实现在mm/filemap.c文件中，主要功能是从磁盘上读取请 求的页面，然后将数据拷贝到用户空间缓冲区中。注释也作了说明，该函数是通用文件读例 程，真正执行读操作，是通过mapping->a_ops->readpage()来完成。
这个函数非常长，我们分段来阅读。
```c
/**
 * do_generic_file_read - generic file read routine
 * @filp:	the file to read
 * @ppos:	current file position
 * @desc:	read_descriptor
 * @actor:	read method
 *
 * This is a generic file read routine, and uses the
 * mapping->a_ops->readpage() function for the actual low-level stuff.
 *
 * This is really ugly. But the goto's actually try to clarify some
 * of the logic when it comes to error handling etc.
 */
static void do_generic_file_read(struct file *filp, loff_t *ppos,
		read_descriptor_t *desc, read_actor_t actor)
{
	struct address_space *mapping = filp->f_mapping;
	struct inode *inode = mapping->host;
	struct file_ra_state *ra = &filp->f_ra;
	pgoff_t index;
	pgoff_t last_index;
	pgoff_t prev_index;
	unsigned long offset;      /* offset into pagecache page */
	unsigned int prev_offset;
	int error;

	index = *ppos >> PAGE_CACHE_SHIFT;
	prev_index = ra->prev_pos >> PAGE_CACHE_SHIFT;
	prev_offset = ra->prev_pos & (PAGE_CACHE_SIZE-1);
	last_index = (*ppos + desc->count + PAGE_CACHE_SIZE-1) >> PAGE_CACHE_SHIFT;
	offset = *ppos & ~PAGE_CACHE_MASK;
```
获取文件address_space对象:mapping = filep->f_mapping；
接下来获取address_space数据结构的拥有者，也就是inode对象，它的地址保存在address_space对象中的host成员变量中;inode对象拥有用来存储读取数据的页面；
获取预读状态ra:ra = &filp->f_ra，记录预读状态；
内核从磁盘上读取文件数据，实际上是以页面大小(4096字节)为单位的;即使用户进程只读1字节，内核每次仍然会至少读取4096字节(若文件本身小于4K怎么办?) 宏PAGE_CACHE_SHIFT的值为12，PAGE_CACHE_SIZE的大小就是一个页面大小(4096字 节)，PAGE_CACHE_MASK的值为~(PAGE_SIZE-1)；
将文件理解成分为多个页面;由前面的解释，我们就不难理解，收下按计算将要读取的数据起始位置index和结束位置last_index所在文件中的页偏移。要读取数据第1字节不 一定是和页面对齐的，所以用临时变量offset记录读取的第一个字节在文件页面内的偏移量。
接下来开始一个for( ; ;)循环处理这些页。
```c
for (;;) {
		struct page *page;
		pgoff_t end_index;
		loff_t isize;
		unsigned long nr, ret;

		cond_resched();
```
cond_resched()检查当前进程TIF_NEED_RESCHED标志，确定是否需要调度;若设置了该标志，则调度schedule()进行进程调度。  

```c
find_page:
		if (fatal_signal_pending(current)) {
			error = -EINTR;
			goto out;
		}

		page = find_get_page(mapping, index);
		if (!page) {
			page_cache_sync_readahead(mapping,
					ra, filp,
					index, last_index - index);
			page = find_get_page(mapping, index);
			if (unlikely(page == NULL))
				goto no_cached_page;
		}
```
用户读取文件数据，使用之后，内核不会立即将数据丢弃;而是将数据缓存在内核里。进程再次请求该数据时，就会直接返回给用户，不必再从磁盘上读取;这样可以大大提高效率。
find_get_page()就是查找请求的数据是否已经在内核缓冲区中。如果返回值为NULL，则 说明此页不在高速缓存中，那么它将执行以下步骤:
（1）page_cache_sync_readahead()，这个函数会在cache未命中时被调用，它将提交 预读请求，修改ra参数，当然也会相应的去预读一些页；
（2）再次去高速缓存中查找(上一步的预取可能已经将其读到cache中)，若果没有找到， 那么说明所请求的数据确实不在cache中，就跳到no_cached_page;  

```c
if (PageReadahead(page)) {
			page_cache_async_readahead(mapping,
					ra, filp, page,
					index, last_index - index);
		}
		if (!PageUptodate(page)) {
			if (inode->i_blkbits == PAGE_CACHE_SHIFT ||
					!mapping->a_ops->is_partially_uptodate)
				goto page_not_up_to_date;
			if (!trylock_page(page))
				goto page_not_up_to_date;
			/* Did it get truncated before we got the lock? */
			if (!page->mapping)
				goto page_not_up_to_date_locked;
			if (!mapping->a_ops->is_partially_uptodate(page,
								desc, offset))
				goto page_not_up_to_date_locked;
			unlock_page(page);
		}
```
继续执行，说明所请求数据已在高速缓存中，那么此时判断它是否为readahead页 ，如果是，则发起异步预读请求page_cache_async_readahead()，这个函数在 page有PG_readahead字段时才被调用，意味着已经用光了足够的readahead window中的 page，我们需要添加更多的page到预读窗口。
接着判断此页的PG_uptodate位，如果被置位，则说明页中所存数据是最新的，因此无需从磁盘读数据，跳到最后的page_ok处。
mapping->a_ops->is_partially_uptodate()检查页面中的buffer是否都处于update状态。 当文件或者设备映射到内存中时，它们的inode结构就会和address_space相关联。当页面属于 一个文件时，page->mapping就会指向这个地址空间。如果这个页面是匿名的且映射开启，则 address_space就是swapper_space，swapper_space是管理交换地址空间的。
若页面不是最新的，说明页中的数据无效:
 (1)若mapping->a_ops->is_partially_uptodate是NULL，或者trylock_page(page)是NULL，那么跳到page_not_up_to_date；
 (2)若文件页面还没建立映射或者页面中的buffer不都是update状态，则跳转到page_not_up_to_date_locked；
 (3)前面没有跳转的话，说明页面是PG_uptodate状态，通过unlock_page()解锁页面;    

```c
page_ok:
		/*
		 * i_size must be checked after we know the page is Uptodate.
		 *
		 * Checking i_size after the check allows us to calculate
		 * the correct value for "nr", which means the zero-filled
		 * part of the page is not copied back to userspace (unless
		 * another truncate extends the file - this is desired though).
		 */

		isize = i_size_read(inode);
		end_index = (isize - 1) >> PAGE_CACHE_SHIFT;
		if (unlikely(!isize || index > end_index)) {
			page_cache_release(page);
			goto out;
		}

		/* nr is the maximum number of bytes to copy from this page */
		nr = PAGE_CACHE_SIZE;
		if (index == end_index) {
			nr = ((isize - 1) & ~PAGE_CACHE_MASK) + 1;
			if (nr <= offset) {
				page_cache_release(page);
				goto out;
			}
		}
		nr = nr - offset;

		/* If users can be writing to this page using arbitrary
		 * virtual addresses, take care about potential aliasing
		 * before reading the page on the kernel side.
		 */
		if (mapping_writably_mapped(mapping))
			flush_dcache_page(page);

		/*
		 * When a sequential read accesses a page several times,
		 * only mark it as accessed the first time.
		 */
		if (prev_index != index || offset != prev_offset)
			mark_page_accessed(page);
		prev_index = index;

		/*
		 * Ok, we have the page, and it's up-to-date, so
		 * now we can copy it to user space...
		 *
		 * The actor routine returns how many bytes were actually used..
		 * NOTE! This may not be the same as how much of a user buffer
		 * we filled up (we may be padding etc), so we can only update
		 * "pos" here (the actor routine has to update the user buffer
		 * pointers and the remaining count).
		 */
		ret = actor(desc, page, offset, nr);
		offset += ret;
		index += offset >> PAGE_CACHE_SHIFT;
		offset &= ~PAGE_CACHE_MASK;
		prev_offset = offset;

		page_cache_release(page);
		if (ret == nr && desc->count)
			continue;
		goto out;
```
代码总是会被执行，即使页面不在Cache中，从磁盘读取数据，之后也会跳转page_ok处。

在page_ok中，首先检查文件大小，也就是index * 4096是否超过i_size，如果超过，则减少page的计数 器，并且跳转到out处。设置nr大小，nr是从此页可读取的最大字节数，就是向用户缓冲区拷贝的字节数。之后就跳转到page_ok标号处。通常的页是4KB，但是对于文件的最后一页，可能不足4KB，所以要进行调整。
如果用户使用不正常的虚拟地址写这个页，要flush_dcache_page()，将dcache相应的 page里的数据写到memory里去，以保证dcache内的数据与memory内的数据的一致性。但在 x86架构中，flush_dcache_page()的实现为空，不做任何操作。
紧接着调用make_page_accessed()设置页面的PG_active和PG_referenced标志，表示页面正在被使用且不能被调换(swap)出去。若同样的页面被do_generic_file_read() 多次读，这一步仅在第一次读时执行(即offset为0)。
下面一行就是将已读取到的数据拷贝到用户空间的缓冲区中，调用的是file_read_actor()函 数，稍后我们再仔细分析这个函数。数据拷贝到用户空间的缓冲区后，更新局部变量offset和 index的值，递减页面计数(page_cache_release)。若read_descriptor_t描述符中的count 值不为0，说明还有其他文件数据需要读，for循环不退出，重新从开始处执行，进行下一 轮读。  

```c
page_not_up_to_date:
		/* Get exclusive access to the page ... */
		error = lock_page_killable(page);
		if (unlikely(error))
			goto readpage_error;

page_not_up_to_date_locked:
		/* Did it get truncated before we got the lock? */
		if (!page->mapping) {
			unlock_page(page);
			page_cache_release(page);
			continue;
		}

		/* Did somebody else fill it already? */
		if (PageUptodate(page)) {
			unlock_page(page);
			goto page_ok;
		}
```
- page_not_up_to_date:
    lock_page_killable()，给page加锁，这个过程可以被中断的，如果加锁失败，则 跳到readpage_error，否则跳转到page_not_up_to_date_locked。
- page_not_up_to_date_locked:
    (1) 如果在获取锁之前就此页已经被truncated，释放锁，减少页的计数，结束本次循环，进入下一次循环；
    (2) 否则，它没有被truncated，如果它已经被填充(已从磁盘上读取)，那么解锁，并且跳转到page_ok    

```c
readpage:
		/*
		 * A previous I/O error may have been due to temporary
		 * failures, eg. multipath errors.
		 * PG_error will be set again if readpage fails.
		 */
		ClearPageError(page);
		/* Start the actual read. The read will unlock the page. */
		error = mapping->a_ops->readpage(filp, page);

		if (unlikely(error)) {
			if (error == AOP_TRUNCATED_PAGE) {
				page_cache_release(page);
				goto find_page;
			}
			goto readpage_error;
		}

		if (!PageUptodate(page)) {
			error = lock_page_killable(page);
			if (unlikely(error))
				goto readpage_error;
			if (!PageUptodate(page)) {
				if (page->mapping == NULL) {
					/*
					 * invalidate_mapping_pages got it
					 */
					unlock_page(page);
					page_cache_release(page);
					goto find_page;
				}
				unlock_page(page);
				shrink_readahead_size_eio(filp, ra);
				error = -EIO;
				goto readpage_error;
			}
			unlock_page(page);
		}

		goto page_ok;

readpage_error:
		/* UHHUH! A synchronous read error occurred. Report it */
		desc->error = error;
		page_cache_release(page);
		goto out;

no_cached_page:
		/*
		 * Ok, it wasn't cached, so we need to create a new
		 * page..
		 */
		page = page_cache_alloc_cold(mapping);
		if (!page) {
			desc->error = -ENOMEM;
			goto out;
		}
		error = add_to_page_cache_lru(page, mapping,
						index, GFP_KERNEL);
		if (error) {
			page_cache_release(page);
			if (error == -EEXIST)
				goto find_page;
			desc->error = error;
			goto out;
		}
		goto readpage;
	}

out:
	ra->prev_pos = prev_index;
	ra->prev_pos <<= PAGE_CACHE_SHIFT;
	ra->prev_pos |= prev_offset;

	*ppos = ((loff_t)index << PAGE_CACHE_SHIFT) + offset;
	file_accessed(filp);
}
```
整个read_page流程就是真正执行从磁盘读取数据的过程，具体来说是：error = mapping->a_ops->readpage(filp, page) 操作执行了真正从磁盘中读取数据，调用文件的address_space对象readpage()方法，对于ext4文件系统来说， readpage方法实现都是函数ext4_readpage()，这个函数仅是对mpage_readpage()简单封装。从函数名就可以看出，该函数一次只读一个页面，显然对于内核来说这种情形是很少有的，大部分是一次读取多个页面，也就是预读(一次读多个页面)。对于do_generic_file_read ()来说，执行最多的路径是page_cache_sync_readahead()和 page_cache_async_readahead()。
mapping->a_ops->readpage()，如果读取失败，error返回值不为0，再进一步判断error 是AOP_TRUNCATED_PAGE时，则减少计数，跳到find_page，如果不是此error，那么跳到 readpage_error。
若页面没有被设置PG_uptodate标志位，即页中的数据不是最新的，那么给页加锁，如果 加锁失败，则跳到readpage_error。加锁成功则再次判断他的PG_uptodate位，如果仍然没有 被置位，再判断page的mapping是不是为空，若为空，则解锁并且减少page计数，然后跳到 find_page。如果mapping不为空，则调整ra，并跳到readpage_error。
经过以上判断检查后，说明页中的数据是最新的，那么跳到page_ok。
下面是几个标号分支处理说明:
- readpage_error:
    -  到此步骤，说明同步read出错，报告这个错误，并且跳出循环，即跳到out
- no_cached_page:
    - 分配一个新页:page_cache_alloc_cold()，然后添加到cache的lru上: add_to_page_cache_lru()，跳到readpage  

这里，我们就可以回答本文概述中的一个问题:“内核是在哪个地方分配空间来存储将要 读取的数据?”。答案是，在do_generic_file_read()函数中的page_cache_alloc_cold(mapping)中，此时内存缓存中没有要请求的数据。这里提醒一下，read()的一个参数buf，其缓冲区是在用户空间的，大小与页面大小无关。而page_cache_alloc_cold(mapping)分配的空间是在内核中，且大小是以页面大小为单位。  

out:
所请求或者可提供的字节已经被读取，更新filp->f_ra提前读数据结构。 然后更新*ppos的值，也就是更新当前文件读写位置; 编程中经常使用read()、write ()的话，不难理解ppos的值。最后调用file_accessed()来更新文件inode节点中i_atime的时 间为当前时间，且将inode标记为脏。至此，do_generic_file_read()函数执行完毕。那么有个问题，现在用户请求的数据是否已经拷贝到用户缓冲区buf中?  

### 2.5.1 address_space -> readpage()方法

调用方法在：
```c
/* Start the actual read. The read will unlock the page. */
		error = mapping->a_ops->readpage(filp, page);
```
  
源码均在fs/ext4/inode.c
```c
static const struct address_space_operations ext4_aops = {
	.readpage		= ext4_readpage,
	.readpages		= ext4_readpages,
	.writepage		= ext4_writepage,
	.writepages		= ext4_writepages,
	.write_begin		= ext4_write_begin,
	.write_end		= ext4_write_end,
	.bmap			= ext4_bmap,
	.invalidatepage_range	= ext4_invalidatepage,
	.releasepage		= ext4_releasepage,
	.direct_IO		= ext4_direct_IO,
	.migratepage		= buffer_migrate_page,
	.is_partially_uptodate  = block_is_partially_uptodate,
	.error_remove_page	= generic_error_remove_page,
};
``` 
这里我们列出系统默认操作方式ext4_aops，readpage实现为ext4_readpage ()，仅是对mpage_readpage()函数作进一步封装。后面我们将详细分析mpage_readpage ()的实现。
ext4_get_block()函数的主要功能:基于给定的inode，查找文件的逻辑块iblock，并将其 与bh进行映射。如果没有查找到，且参数create为1，则执行块分配操作为文件申请新的 block，然后再与bh建立映射。
```c
static int ext4_readpage(struct file *file, struct page *page)
{
	int ret = -EAGAIN;
	struct inode *inode = page->mapping->host;

	trace_ext4_readpage(page);

	if (ext4_has_inline_data(inode))
		ret = ext4_readpage_inline(inode, page);

	if (ret == -EAGAIN)
		return mpage_readpage(page, ext4_get_block);

	return ret;
}
```

### 2.5.2 file_read_actor()
函数file_read_actor()源码出mm/filemap.c。
该函数在do_generic_file_read()的page_ok模块中被调用，是用来将内核缓冲区中的数据拷贝到用 户缓冲区中。
```c
int file_read_actor(read_descriptor_t *desc, struct page *page,
			unsigned long offset, unsigned long size)
{
	char *kaddr;
	unsigned long left, count = desc->count;

	if (size > count)
		size = count;

	/*
	 * Faults on the destination of a read are common, so do it before
	 * taking the kmap.
	 */
	if (IS_ENABLED(CONFIG_HIGHMEM) && !fault_in_pages_writeable(desc->arg.buf, size)) {
		kaddr = kmap_atomic(page);
		left = __copy_to_user_inatomic(desc->arg.buf,
						kaddr + offset, size);
		kunmap_atomic(kaddr);
		if (left == 0)
			goto success;
	}

	/* Do it the slow way */
	kaddr = kmap(page);
	left = __copy_to_user(desc->arg.buf, kaddr + offset, size);
	kunmap(page);

	if (left) {
		size -= left;
		desc->error = -EFAULT;
	}
success:
	desc->count = count - size;
	desc->written += size;
	desc->arg.buf += size;
	return size;
}
```
函数执行步骤如下：
(1) 若空间处于高端内存，则调用kmap_atomic()建立内核持久映射；
(2) 调用__copy_to_user_inatomic()将数据拷贝到用户空间;注意此时进程可能会被阻塞，因为访问用户空间时发生页面错误;
(3) 调用kunmap()解除地址映射;
(4) 更新count、written和arg.buf的值。count表示待读取的字节数，written是已拷贝到用 户控件的字节数，arg.buf是用户空间的缓冲区地址。

### 2.5.3 mpage_readpage()
前面提到Linux内核读文件通常采取预读机制，不是每次只读一个页面。在 do_generic_file_read()函数中，执行最多的是page_cache_sync_readahead()和 page_cache_async_readahead()，很少直接调用mapping->a_ops->readpage()。我们会在另外的章节中单独介绍Linux内核预读和Cache机制，在预读机制中，会调用 mapping->a_ops->readpages()一次读取多个页面。
这里我们先分析执行较少时的情形，调用a_ops->readpage()一次读取一个页面。一次读取多个页面，在Linux内核预读机制章节分析。
```c
/*
 * This isn't called much at all
 */
int mpage_readpage(struct page *page, get_block_t get_block)
{
	struct bio *bio = NULL;
	sector_t last_block_in_bio = 0;
	struct buffer_head map_bh;
	unsigned long first_logical_block = 0;
	gfp_t gfp = mapping_gfp_constraint(page->mapping, GFP_KERNEL);

	map_bh.b_state = 0;
	map_bh.b_size = 0;
	bio = do_mpage_readpage(bio, page, 1, &last_block_in_bio,
			&map_bh, &first_logical_block, get_block, gfp);
	if (bio)
		mpage_bio_submit(READ, bio);
	return 0;
}
EXPORT_SYMBOL(mpage_readpage);
```
主要的执行函数是do_mpage_readpage()，下面详细分析do_mpage_readpage()函数的执行流程。
do_mpage_readpage()也在文件fs/mpage.c中，试图读取文件中的一个page大小的数 据。理想的情况是这个page大小的数据在磁盘上都是连续的，然后只需要提交一个bio请求就 可以获取所有的数据。
do_mpage_readpage()大部分工作在检查page上所有的物理块是否连续，检查的方法就是调用文件系统提供的get_block()函数;如果不连续，就调用block_read_full_page ()以 buffer 缓冲区的形式来逐个块获取数据。
do_mpage_readpage分析page对应的物理块映射关系进行不同处理: 
(1)调用get_block()函数检查page中所有的物理块是否连续; 
(2)若磁盘上物理块不连续，就调用mpage_bio_submit()读取；
 (3)若磁盘上物理块连续。如果没有bio，则新分配一个bio。
 (4)物理块连续，但是和上一个page的物理block不连续。把上一次的bio给提交了，然后重新分配bio。
 (5)所请求的页面有hole，也有数据，就block_read_full_page()逐块读取。
 (6)所请求的页面全是hole，将page清零，不用读了，直接返回。
```c
/*
 * This is the worker routine which does all the work of mapping the disk
 * blocks and constructs largest possible bios, submits them for IO if the
 * blocks are not contiguous on the disk.
 *
 * We pass a buffer_head back and forth and use its buffer_mapped() flag to
 * represent the validity of its disk mapping and to decide when to do the next
 * get_block() call.
 */
static struct bio *
do_mpage_readpage(struct bio *bio, struct page *page, unsigned nr_pages,
		sector_t *last_block_in_bio, struct buffer_head *map_bh,
		unsigned long *first_logical_block, get_block_t get_block,
		gfp_t gfp)
{
	struct inode *inode = page->mapping->host;
	const unsigned blkbits = inode->i_blkbits;
	const unsigned blocks_per_page = PAGE_CACHE_SIZE >> blkbits;
	const unsigned blocksize = 1 << blkbits;
	sector_t block_in_file;
	sector_t last_block;
	sector_t last_block_in_file;
	sector_t blocks[MAX_BUF_PER_PAGE];
	unsigned page_block;
	unsigned first_hole = blocks_per_page;
	struct block_device *bdev = NULL;
	int length;
	int fully_mapped = 1;
	unsigned nblocks;
	unsigned relative_block;

	if (page_has_buffers(page))
		goto confused;

	block_in_file = (sector_t)page->index << (PAGE_CACHE_SHIFT - blkbits);
	last_block = block_in_file + nr_pages * blocks_per_page;
	last_block_in_file = (i_size_read(inode) + blocksize - 1) >> blkbits;
	if (last_block > last_block_in_file)
		last_block = last_block_in_file;
	page_block = 0;
```
首先检查页面PG_private标志;若该标志已设置，则表明:
(1)它是缓冲页面 (即该页面与一个buffer head链表关联)，且已经从磁盘上读取到内存中;
(2)页面中的块在 磁盘上不是相邻的;这样的话，就跳转到confused模块，一次读取页面的一个块。
获取块大小blocksize(保存在page->mapping->host->inode->i_blkbits)，并且计算访问这 个页面所需的两个值: 页面中的块数和页面的第一个块(即页内的第一个块相对于文 件起始位置的索引)。
block_in_file: page中的第一个block number；
last_block: page中最后一个block 大小；
last_block_in_file: 文件最后一个block大小；
last_block最终值为本次page读操作的最后一个block大小。  

```c
/*
	 * Map blocks using the result from the previous get_blocks call first.
	 */
	nblocks = map_bh->b_size >> blkbits;
	if (buffer_mapped(map_bh) && block_in_file > *first_logical_block &&
			block_in_file < (*first_logical_block + nblocks)) {
		unsigned map_offset = block_in_file - *first_logical_block;
		unsigned last = nblocks - map_offset;

		for (relative_block = 0; ; relative_block++) {
			if (relative_block == last) {
				clear_buffer_mapped(map_bh);
				break;
			}
			if (page_block == blocks_per_page)
				break;
			blocks[page_block] = map_bh->b_blocknr + map_offset +
						relative_block;
			page_block++;
			block_in_file++;
		}
		bdev = map_bh->b_bdev;
	}
```
通常情况下，map_bh仅是临时变量，不会执行上述代码，所以这里忽略。

```c
/*
	 * Then do more get_blocks calls until we are done with this page.
	 */
	map_bh->b_page = page;
	while (page_block < blocks_per_page) {
		map_bh->b_state = 0;
		map_bh->b_size = 0;

		if (block_in_file < last_block) {
			map_bh->b_size = (last_block-block_in_file) << blkbits;
			if (get_block(inode, block_in_file, map_bh, 0))
				goto confused;
			*first_logical_block = block_in_file;
		}

		if (!buffer_mapped(map_bh)) {
			fully_mapped = 0;
			if (first_hole == blocks_per_page)
				first_hole = page_block;
			page_block++;
			block_in_file++;
			continue;
		}

		/* some filesystems will copy data into the page during
		 * the get_block call, in which case we don't want to
		 * read it again.  map_buffer_to_page copies the data
		 * we just collected from get_block into the page's buffers
		 * so readpage doesn't have to repeat the get_block call
		 */
		if (buffer_uptodate(map_bh)) {
			map_buffer_to_page(page, map_bh, page_block);
			goto confused;
		}
	
		if (first_hole != blocks_per_page)
			goto confused;		/* hole -> non-hole */

		/* Contiguous blocks? */
		if (page_block && blocks[page_block-1] != map_bh->b_blocknr-1)
			goto confused;
		nblocks = map_bh->b_size >> blkbits;
		for (relative_block = 0; ; relative_block++) {
			if (relative_block == nblocks) {
				clear_buffer_mapped(map_bh);
				break;
			} else if (page_block == blocks_per_page)
				break;
			blocks[page_block] = map_bh->b_blocknr+relative_block;
			page_block++;
			block_in_file++;
		}
		bdev = map_bh->b_bdev;
	}
```
(1) 对于页内的每一个块，调用文件系统相关的get_block函数计算逻辑块数，就是相对于磁盘或分区起始位置的索引;
(2) 页面里所有块的逻辑块数存放在一个局部数组blocks[]中;
(3) 检查前面几步中可能出现的异常条件。例如，有些块在磁盘上不相邻，或有些块落入文件空洞中，或某个块缓冲区已经由get_block函数填充。然跳转到confused 模块处。  

```c
if (first_hole != blocks_per_page) {
		zero_user_segment(page, first_hole << blkbits, PAGE_CACHE_SIZE);
		if (first_hole == 0) {
			SetPageUptodate(page);
			unlock_page(page);
			goto out;
		}
	} else if (fully_mapped) {
		SetPageMappedToDisk(page);
	}
```
如果发现文件中有空洞，就将整个page清零，因为文件洞的区域物理层不会真的去磁盘上读取，必须在这里主动清零，否则文件洞区域内容可能随机。该页面也可能是文件的最后一部分数据，于是页面中的部分块在磁盘上没有对应的数据。这样的话，将这些块中都清零。否则 设置页面描述符的PG_mappeddisk标志。  

```c
	/*
	 * This page will go to BIO.  Do we need to send this BIO off first?
	 */
	if (bio && (*last_block_in_bio != blocks[0] - 1))
		bio = mpage_bio_submit(READ, bio);

alloc_new:
	if (bio == NULL) {
		if (first_hole == blocks_per_page) {
			if (!bdev_read_page(bdev, blocks[0] << (blkbits - 9),
								page))
				goto out;
		}
		bio = mpage_alloc(bdev, blocks[0] << (blkbits - 9),
			  	min_t(int, nr_pages, bio_get_nr_vecs(bdev)),
				gfp);
		if (bio == NULL)
			goto confused;
	}
```
调用mpage_alloc()分配一个新bio描述符，仅包含一个段(segment); 并将成员变量bi_bdev的值初始化为块设备描述符地址，将成员变量bi_sector初始化为页面中第一 个块的逻辑块号。这两个值在前面已经计算。   

```c
length = first_hole << blkbits;
	if (bio_add_page(bio, page, length, 0) < length) {
		bio = mpage_bio_submit(READ, bio);
		goto alloc_new;
	}

	relative_block = block_in_file - *first_logical_block;
	nblocks = map_bh->b_size >> blkbits;
	if ((buffer_boundary(map_bh) && relative_block == nblocks) ||
	    (first_hole != blocks_per_page))
		bio = mpage_bio_submit(READ, bio);
	else
		*last_block_in_bio = blocks[blocks_per_page - 1];
out:
	return bio;

confused:
	if (bio)
		bio = mpage_bio_submit(READ, bio);
	if (!PageUptodate(page))
	        block_read_full_page(page, get_block);
	else
		unlock_page(page);
	goto out;
}
```
若物理块上连续，则尝试页面合并成新的bio。
如果当前page和上一个page的物理 block不连续，把上一次的bio给提交，然后重新分配bio。
若当前page和上一个page，物理上连续，合并成新的bio后，返回更新后的bio，等待下一 次操作(316行)。
> confused模块:
若函数跳转到这里，页面中包含的块在磁盘上是不相邻的。若页面是最新的 (设置了PG_update标志)，调用unlock_page()来对页面解锁，否则就调用block_read_full_page()来一次读取页面上的一个块。  

> mpage_bio_submit()函数
首先设置bio->bi_end_io的值为mpage_end_io()， 该函数是bio结束方法。当I/O数据传输结束时，它就立即被执行。假设没有I/O错误，该函数主 要设置页面描述符中的PG_uptodate标志，然后调用unlock_page()来解锁页面，并唤醒在事 件上等待的所有进程;最后调用bio_put()销毁bio描述符。在调用submit_bio()，设置数据传输方向到bi_rw标志，更新per-CPU类型变量page_states以记录读扇区数;然后调用 generic_make_request()。
## 2.8 mpage_bio_submit()
```c
static struct bio *mpage_bio_submit(int rw, struct bio *bio)
{
	bio->bi_end_io = mpage_end_io;
	guard_bio_eod(rw, bio);
	submit_bio(rw, bio);
	return NULL;
}
```
mpage_bio_submit()的源码非常简单，就是设置bio的bi_end_io方法，然后调用 submit_bio()。源码实现在fs/mpage.c中。

## 2.9 小结
至此，我们分析了read()在虚拟文件系统和文件系统中的流程，分析到了 mpage_bio_submit()，该函数最后调用submit_bio()，此时进入了通用块设备层(Generic Block Layer)，接下来会进入I/O调度层、驱动层，最后数据返回。

# 3. 读数据返回
## 3.1 读进程的阻塞和继续执行过程
当内核将读请求发到磁盘控制器驱动的请求队列中，显然将请求放入队列中后，用户进程 请求的数据并没有立即读到内核空间和用户缓冲区中。将读请求放入队列中后，sys_read() 执行结束，并返回到用户空间?用户进程读数据时，是否会有阻塞过程?
将请求放入队列后，read()系统调用并没有立即从内核空间返回到用户空间。因为请求的数据还没有读到内存中，就不能返回到用户空间，一旦返回到用户空间，用户进程就可以随时访问，可此时用户缓冲区中还没有请求的数据呢，又如何能访问?
将请求放到块设备(磁盘)的请求队列上后，用户进程就会阻塞，直到所请求的数据就绪为 止。本小节就主要分析在sys_read()执行过程中，在哪里阻塞，又是何时被唤醒继续执行的过程。
回顾前面分析的函数do_generic_file_read()函数：
```c
static void do_generic_file_read(struct file *filp, loff_t *ppos,
		read_descriptor_t *desc, read_actor_t actor)
{
	struct address_space *mapping = filp->f_mapping;
	struct inode *inode = mapping->host;
	struct file_ra_state *ra = &filp->f_ra;
	...................................
		cond_resched();
find_page:
		if (fatal_signal_pending(current)) {
			error = -EINTR;
			goto out;
		}

		page = find_get_page(mapping, index);
		if (!page) {
			page_cache_sync_readahead(mapping,
					ra, filp,
					index, last_index - index);
			page = find_get_page(mapping, index);
			if (unlikely(page == NULL))
				goto no_cached_page;
		}
		if (PageReadahead(page)) {
			page_cache_async_readahead(mapping,
					ra, filp, page,
					index, last_index - index);
		}
		if (!PageUptodate(page)) {
			if (inode->i_blkbits == PAGE_CACHE_SHIFT ||
					!mapping->a_ops->is_partially_uptodate)
				goto page_not_up_to_date;
			if (!trylock_page(page))
				goto page_not_up_to_date;
			/* Did it get truncated before we got the lock? */
			if (!page->mapping)
				goto page_not_up_to_date_locked;
			if (!mapping->a_ops->is_partially_uptodate(page,
								desc, offset))
				goto page_not_up_to_date_locked;
			unlock_page(page);
		}
```
读文件数据大多数情况下都是通过预读的，也就是执行page_cache_sync_readahead() 或page_cache_async_readahead()。该函数中完成了将数据请求放到块设备队列上去，然后该函数就执行结束了，但请求的数据并不会那么快已读到内存中。
执行预读后，通过find_get_page()再次查找所请求的页面是否在缓冲区中，经过预读后，页面已经在缓冲区中。进而执行if (!PageUptodate(page))，因为请求的数据还未就绪，那么页面得 PG_update标志就没有被设置，就会跳转到page_not_up_to_date标号处。
```c
page_not_up_to_date:
		/* Get exclusive access to the page ... */
		error = lock_page_killable(page);
		if (unlikely(error))
			goto readpage_error;

page_not_up_to_date_locked:
		/* Did it get truncated before we got the lock? */
		if (!page->mapping) {
			unlock_page(page);
			page_cache_release(page);
			continue;
		}

		/* Did somebody else fill it already? */
		if (PageUptodate(page)) {
			unlock_page(page);
			goto page_ok;
		}
```
进程的阻塞正是在lock_page_killable()inline函数中，当请求的数据读到内存中后， lock_page_killable()才会执行完毕，该函数定义在include/linux/pagemap.h中。
```c
/*
 * lock_page_killable is like lock_page but can be interrupted by fatal
 * signals.  It returns 0 if it locked the page and -EINTR if it was
 * killed while waiting.
 */
static inline int lock_page_killable(struct page *page)
{
	might_sleep();
	if (!trylock_page(page))
		return __lock_page_killable(page);
	return 0;
}
```
__lock_page_killable()的实现在mm/filemap.c中
```c
int __lock_page_killable(struct page *page)
{
	DEFINE_WAIT_BIT(wait, &page->flags, PG_locked);

	return __wait_on_bit_lock(page_waitqueue(page), &wait,
					bit_wait_io, TASK_KILLABLE);
}
EXPORT_SYMBOL_GPL(__lock_page_killable);
```
当数据还未就绪时，调用__wait_on_bit_lock()等待解除PG_locked和设置PG_update。
后面我们会分析内核在何时、何处解除page锁标志和设置PG_update标志。

## 3.2 读数据返回过程
这里我们就不详细分析磁盘控制器驱动是如何从请求队列中摘掉请求，然后读取数据的。 现在从数据完全都到内存缓冲区中开始分析。当数据读到内存中后，磁盘控制器就会向CPU发一个中断，然后就会执行中断处理程序。
```c
/*
 * I/O completion handler for multipage BIOs.
 *
 * The mpage code never puts partial pages into a BIO (except for end-of-file).
 * If a page does not map to a contiguous run of blocks then it simply falls
 * back to block_read_full_page().
 *
 * Why is this?  If a page's completion depends on a number of different BIOs
 * which can complete in any order (or at the same time) then determining the
 * status of that page is hard.  See end_buffer_async_read() for the details.
 * There is no point in duplicating all that complexity.
 */
static void mpage_end_io(struct bio *bio, int err)
{
	struct bio_vec *bv;
	int i;

	bio_for_each_segment_all(bv, bio, i) {
		struct page *page = bv->bv_page;
		page_endio(page, bio_data_dir(bio), err);
	}

	bio_put(bio);
}
```
具体的实现是page_endio操作，这个函数将读和写封装到一个函数中了
```c
/*
 * After completing I/O on a page, call this routine to update the page
 * flags appropriately
 */
void page_endio(struct page *page, int rw, int err)
{
	if (rw == READ) {
		if (!err) {
			SetPageUptodate(page);
		} else {
			ClearPageUptodate(page);
			SetPageError(page);
		}
		unlock_page(page);
	} else { /* rw == WRITE */
		if (err) {
			SetPageError(page);
			if (page->mapping)
				mapping_set_error(page->mapping, err);
		}
		end_page_writeback(page);
	}
}
EXPORT_SYMBOL_GPL(page_endio);
```
根据rw判断是读还是写，我们这里暂时只讨论读的情况。
若数据完全就绪，此时就设置page的PG_update标志，然后解锁页面(unlock_page函数)。 当读进程再次被调度执行时，就不会再被阻塞在lock_page_killable()函数中了。
对于读取的数据如何拷贝到用户缓冲区中，do_generic_file_read()函数中已作了分析。



