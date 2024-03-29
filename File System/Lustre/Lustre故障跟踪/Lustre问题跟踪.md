date: 2019-07-24 17:15
app: markdown
layout: 'post'
title: 'Lustre故障跟踪之nrs_tbf_rule_match'
tags: Lustre

最近半年总是遇到一个bug，导致Lustre MDS节点发生crash，这篇文章主要是分析这个bug产生的原因及如何定位

先上日志：
```bash
[1139547.817517] LustreError: 20086:0:(nrs_tbf.c:235:nrs_tbf_rule_match()) ASSERTION( (tmp_rule->tr_flags & 0x0000001) == 0 ) failed:
[1139547.817540] LustreError: 20086:0:(nrs_tbf.c:235:nrs_tbf_rule_match()) LBUG
[1139547.817544] Pid: 20086, comm: mdt00_068
[1139547.817547]
Call Trace:
[1139547.817586]  [<ffffffffc09d57ae>] libcfs_call_trace+0x4e/0x60 [libcfs]
[1139547.817602]  [<ffffffffc09d583c>] lbug_with_loc+0x4c/0xb0 [libcfs]
[1139547.817700]  [<ffffffffc0f804c5>] nrs_tbf_rule_match+0xc5/0xd0 [ptlrpc]
[1139547.817780]  [<ffffffffc0f834ad>] nrs_tbf_res_get+0xad/0x4c0 [ptlrpc]
[1139547.817852]  [<ffffffffc0f7621c>] nrs_resource_get+0x7c/0x100 [ptlrpc]
[1139547.817922]  [<ffffffffc0f76790>] nrs_resource_get_safe+0x80/0xf0 [ptlrpc]
[1139547.817993]  [<ffffffffc0f7a263>] ptlrpc_nrs_req_initialize+0x83/0x100 [ptlrpc]
[1139547.818059]  [<ffffffffc0f48f31>] ptlrpc_main+0x1771/0x1e40 [ptlrpc]
[1139547.818125]  [<ffffffffc0f477c0>] ? ptlrpc_main+0x0/0x1e40 [ptlrpc]
[1139547.818134]  [<ffffffff810b252f>] kthread+0xcf/0xe0
[1139547.818141]  [<ffffffff810b2460>] ? kthread+0x0/0xe0
[1139547.818149]  [<ffffffff816b8798>] ret_from_fork+0x58/0x90
[1139547.818155]  [<ffffffff810b2460>] ? kthread+0x0/0xe0
[1139547.818159]
[1139547.818162] Kernel panic - not syncing: LBUG
[1139547.818212] CPU: 16 PID: 20086 Comm: mdt00_068 Tainted: G           OEL ------------   3.10.0-693.11.6.el7_lustre.x86_64 #1
[1139547.818355] Call Trace:
[1139547.818385]  [<ffffffff816a5e7d>] dump_stack+0x19/0x1b
[1139547.818433]  [<ffffffff8169fd64>] panic+0xe8/0x20d
[1139547.818492]  [<ffffffffc09d5854>] lbug_with_loc+0x64/0xb0 [libcfs]
[1139547.818611]  [<ffffffffc0f804c5>] nrs_tbf_rule_match+0xc5/0xd0 [ptlrpc]
[1139547.818732]  [<ffffffffc0f834ad>] nrs_tbf_res_get+0xad/0x4c0 [ptlrpc]
[1139547.818848]  [<ffffffffc0f7621c>] nrs_resource_get+0x7c/0x100 [ptlrpc]
[1139547.818965]  [<ffffffffc0f76790>] nrs_resource_get_safe+0x80/0xf0 [ptlrpc]
[1139547.819084]  [<ffffffffc0f7a263>] ptlrpc_nrs_req_initialize+0x83/0x100 [ptlrpc]
[1139547.819203]  [<ffffffffc0f48f31>] ptlrpc_main+0x1771/0x1e40 [ptlrpc]
[1139547.819316]  [<ffffffffc0f477c0>] ? ptlrpc_register_service+0xe30/0xe30 [ptlrpc]
[1139547.819376]  [<ffffffff810b252f>] kthread+0xcf/0xe0
[1139547.819419]  [<ffffffff810b2460>] ? insert_kthread_work+0x40/0x40
[1139547.819470]  [<ffffffff816b8798>] ret_from_fork+0x58/0x90
[1139547.819516]  [<ffffffff810b2460>] ? insert_kthread_work+0x40/0x40
```
Lustre version: 2.10.3
Kernel version 3.10.693.11.6_lustre
OFED: MLNX_OFED_LINUX-4.2-1.2.0.0

机器在发生crash时产生了vmcore文件，在/var/crash目录中，因此后面的分析主要是针对/var/crash目录中的kdump文件进行

# 1、配置vmcore文件解析环境
如何分析vmcore这个文件？
- 需要vmlinux
- 需要crash工具

具体来说，需要以下三个步骤：
1、安装kernel debuginfo，对于我的机器来说，对应debuginfo包是:
```bash
kernel-debuginfo-3.10.0-693.11.6.el7_lustre.x86_64.rpm
kernel-debuginfo-common-x86_64-3.10.0-693.11.6.el7_lustre.x86_64.rpm
```

2、安装crash工具以及debuginfo包
```bash
crash-7.1.9-2.el7.x86_64.rpm
crash-debuginfo-7.1.9-2.el7.x86_64.rpm
```

3、查询vmlinux路径
```bash
[root@mds01 ]# find / -name vmlinux
/usr/lib/debug/usr/lib/modules/3.10.0-693.11.6.el7_lustre.x86_64/vmlinux
```
接下来就可以分析vmcore文件了，如下所示：
执行分析命令
```bash
crash /usr/lib/debug/usr/lib/modules/3.10.0-693.11.6.el7_lustre.x86_64/vmlinux vmcore
```
一般会得到下面的结果:
```bash
crash 7.1.9-2.el7
Copyright (C) 2002-2016  Red Hat, Inc.
Copyright (C) 2004, 2005, 2006, 2010  IBM Corporation
Copyright (C) 1999-2006  Hewlett-Packard Co
Copyright (C) 2005, 2006, 2011, 2012  Fujitsu Limited
Copyright (C) 2006, 2007  VA Linux Systems Japan K.K.
Copyright (C) 2005, 2011  NEC Corporation
Copyright (C) 1999, 2002, 2007  Silicon Graphics, Inc.
Copyright (C) 1999, 2000, 2001, 2002  Mission Critical Linux, Inc.
This program is free software, covered by the GNU General Public License,
and you are welcome to change it and/or distribute copies of it under
certain conditions.  Enter "help copying" to see the conditions.
This program has absolutely no warranty.  Enter "help warranty" for details.

GNU gdb (GDB) 7.6
Copyright (C) 2013 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-unknown-linux-gnu"...

      KERNEL: /usr/lib/debug/usr/lib/modules/3.10.0-693.11.6.el7_lustre.x86_64/vmlinux
    DUMPFILE: vmcore  [PARTIAL DUMP]
        CPUS: 80
        DATE: Fri Apr 12 18:29:17 2019
      UPTIME: 13 days, 04:32:25
LOAD AVERAGE: 4.58, 19.01, 16.58
       TASKS: 1697
    NODENAME: mds01
     RELEASE: 3.10.0-693.11.6.el7_lustre.x86_64
     VERSION: #1 SMP Wed Jan 31 20:03:36 UTC 2018
     MACHINE: x86_64  (2400 Mhz)
      MEMORY: 766.7 GB
       PANIC: "Kernel panic - not syncing: LBUG"
         PID: 20086
     COMMAND: "mdt00_068"
        TASK: ffff885972b03f40  [THREAD_INFO: ffff88534e754000]
         CPU: 16
       STATE: TASK_RUNNING (PANIC)
```

# 2、利用crash工具分析vmcore
从上面的到的信息可以看出，当前crash进程的pid是20086
发生crash对应的代码是：
```c
static struct nrs_tbf_rule *
nrs_tbf_rule_match(struct nrs_tbf_head *head,
		   struct nrs_tbf_client *cli)
{
	struct nrs_tbf_rule *rule = NULL;
	struct nrs_tbf_rule *tmp_rule;

	spin_lock(&head->th_rule_lock);
	/* Match the newest rule in the list */
	list_for_each_entry(tmp_rule, &head->th_list, tr_linkage) {
		LASSERT((tmp_rule->tr_flags & NTRS_STOPPING) == 0);
		if (head->th_ops->o_rule_match(tmp_rule, cli)) {
			rule = tmp_rule;
			break;
		}
	}

	if (rule == NULL)
		rule = head->th_rule;

	nrs_tbf_rule_get(rule);
	spin_unlock(&head->th_rule_lock);
	return rule;
}
```

这里面涉及到两个数据结构：nrs_tbf_head和nrs_tbf_rule

下面是nrs_tbf_rule的定义:
```c
#define MAX_TBF_NAME (16)

#define NTRS_STOPPING	0x0000001
#define NTRS_DEFAULT	0x0000002

struct nrs_tbf_rule {
	/** Name of the rule. */
	char				 tr_name[MAX_TBF_NAME];
	/** Head belongs to. */
	struct nrs_tbf_head		*tr_head;
	/** Likage to head. */
	struct list_head		 tr_linkage;
	/** Nid list of the rule. */
	struct list_head		 tr_nids;
	/** Nid list string of the rule.*/
	char				*tr_nids_str;
	/** Jobid list of the rule. */
	struct list_head		 tr_jobids;
	/** Jobid list string of the rule.*/
	char				*tr_jobids_str;
	/** Opcode bitmap of the rule. */
	struct cfs_bitmap		*tr_opcodes;
	/** Opcode list string of the rule.*/
	char				*tr_opcodes_str;
	/** Condition list of the rule.*/
	struct list_head		tr_conds;
	/** Generic condition string of the rule. */
	char				*tr_conds_str;
	/** RPC/s limit. */
	__u64				 tr_rpc_rate;
	/** Time to wait for next token. */
	__u64				 tr_nsecs;
	/** Token bucket depth. */
	__u64				 tr_depth;
	/** Lock to protect the list of clients. */
	spinlock_t			 tr_rule_lock;
	/** List of client. */
	struct list_head		 tr_cli_list;
	/** Flags of the rule. */
	__u32				 tr_flags;
	/** Usage Reference count taken on the rule. */
	atomic_t			 tr_ref;
	/** Generation of the rule. */
	__u64				 tr_generation;
};
```

下面是nrs_tbf_head的定义:
```c
/**
 * Private data structure for the TBF policy
 */
struct nrs_tbf_head {
	/**
	 * Resource object for policy instance.
	 */
	struct ptlrpc_nrs_resource	 th_res;
	/**
	 * List of rules.
	 */
	struct list_head		 th_list;
	/**
	 * Lock to protect the list of rules.
	 */
	spinlock_t			 th_rule_lock;
	/**
	 * Generation of rules.
	 */
	atomic_t			 th_rule_sequence;
	/**
	 * Default rule.
	 */
	struct nrs_tbf_rule		*th_rule;
	/**
	 * Timer for next token.
	 */
	struct hrtimer			 th_timer;
	/**
	 * Deadline of the timer.
	 */
	__u64				 th_deadline;
	/**
	 * Sequence of requests.
	 */
	__u64				 th_sequence;
	/**
	 * Heap of queues.
	 */
	struct cfs_binheap		*th_binheap;
	/**
	 * Hash of clients.
	 */
	struct cfs_hash			*th_cli_hash;
	/**
	 * Type of TBF policy.
	 */
	char				 th_type[NRS_TBF_TYPE_MAX_LEN + 1];
	/**
	 * Rule operations.
	 */
	struct nrs_tbf_ops		*th_ops;
	/**
	 * Flag of type.
	 */
	__u32				 th_type_flag;
	/**
	 * Index of bucket on hash table while purging.
	 */
	int				 th_purge_start;
};
```

所以需要分析在发生crash时tmp_rule->tr_flags的值是多少，因为正常情况下只可能是1和2，不可能有其它值，但是如果发生了野指针的指针漂移，只要最后一位是1，就会触发crash。

>  下面的工作主要是从当前的栈里取出tmp_rule->tr_flags的值

查看当前PID的栈帧
```bash
#4  [ffff88534e757cd0] nrs_tbf_rule_match at ffffffffc0f804c5 [ptlrpc]
    ffff88534e757cd8: ffff8852e0748b00 ffff88533b693800
    ffff88534e757ce8: ffff885eb9f08100 ffff88534e757d60
    ffff88534e757cf8: ffff886b0c0a8000 ffff88534e757d48
    ffff88534e757d08: ffffffffc0f834ad
```
根据经验，栈顶的数据应该是nrs_tbf_head中的内容(其实是轮询出来的)，网上有一张图给了我灵感：
![](./_image/2019-07-24/2019-07-24-22-10-06.png)


接着打印nrs_tbf_head中的内容即可

```bash
crash> struct nrs_tbf_head ffff8852e0748b00
struct: invalid data structure reference: nrs_tbf_head
```

出现这个错误的原因是没有加载符号表，安装Lustre rpm包时并没有安装debuginfo包
解决方案：
- 安装lustre-debuginfo-2.10.3-1.el7.x86_64.rpm
- 动态加载符号表

动态加载符号表的方法如下：
mod -S /lib/modules/3.10.0-693.11.6.el7_lustre.x86_64/extra/lustre/
```bash
 MODULE       NAME                       SIZE  OBJECT FILE
ffffffffc0861020  ksocklnd                 179389  /lib/modules/3.10.0-693.11.6.el7_lustre.x86_64/extra/lustre/net/ksocklnd.ko
mod: cannot find or load object file for intel_powerclamp module
ffffffffc08b03c0  ko2iblnd                 233842  /lib/modules/3.10.0-693.11.6.el7_lustre.x86_64/extra/lustre/net/ko2iblnd.ko
mod: cannot find or load object file for jbd2 module
mod: cannot find or load object file for edac_core module
ffffffffc0901d40  fld                       89953  /lib/modules/3.10.0-693.11.6.el7_lustre.x86_64/extra/lustre/fs/fld.ko
mod: cannot find or load object file for ipmi_msghandler module
mod: cannot find or load object file for mlx5_core module
ffffffffc0a03120  libcfs                   415815  /lib/modules/3.10.0-693.11.6.el7_lustre.x86_64/extra/lustre/net/libcfs.ko
ffffffffc0a843a0  lnet                     484580  /lib/modules/3.10.0-693.11.6.el7_lustre.x86_64/extra/lustre/net/lnet.ko
mod: cannot find or load object file for nf_conntrack module
ffffffffc0ae4880  fid                       90740  /lib/modules/3.10.0-693.11.6.el7_lustre.x86_64/extra/lustre/fs/fid.ko
ffffffffc0b2cf40  lquota                   359017  /lib/modules/3.10.0-693.11.6.el7_lustre.x86_64/extra/lustre/fs/lquota.ko
mod: cannot find or load object file for ldiskfs module
mod: cannot find or load object file for coretemp module
ffffffffc0c2a560  mgc                       94146  /lib/modules/3.10.0-693.11.6.el7_lustre.x86_64/extra/lustre/fs/mgc.ko
mod: cannot find or load object file for osd_ldiskfs module
mod: cannot find or load object file for ghash_clmulni_intel module
ffffffffc0dbeb60  obdclass                1901030  /lib/modules/3.10.0-693.11.6.el7_lustre.x86_64/extra/lustre/fs/obdclass.ko
ffffffffc10598c0  ptlrpc                  2236624  /lib/modules/3.10.0-693.11.6.el7_lustre.x86_64/extra/lustre/fs/ptlrpc.ko
ffffffffc1143da0  mgs                      345419  /lib/modules/3.10.0-693.11.6.el7_lustre.x86_64/extra/lustre/fs/mgs.ko
ffffffffc11e6b80  lfsck                    726122  /lib/modules/3.10.0-693.11.6.el7_lustre.x86_64/extra/lustre/fs/lfsck.ko
ffffffffc1290020  mdt                      726117  /lib/modules/3.10.0-693.11.6.el7_lustre.x86_64/extra/lustre/fs/mdt.ko
ffffffffc1315020  lod                      464779  /lib/modules/3.10.0-693.11.6.el7_lustre.x86_64/extra/lustre/fs/lod.ko
ffffffffc1379b60  mdd                      378429  /lib/modules/3.10.0-693.11.6.el7_lustre.x86_64/extra/lustre/fs/mdd.ko
ffffffffc13cf640  osp                      336921  /lib/modules/3.10.0-693.11.6.el7_lustre.x86_64/extra/lustre/fs/osp.ko
mod: cannot find or load object file for kvm_intel module
```
从结果看，ptlrpc.ko已经加载进去

接下来可以打印结构体中的内容了
struct nrs_tbf_head ffff8852e0748b00
```bash
crash> struct nrs_tbf_head ffff8852e0748b00
struct nrs_tbf_head {
  th_res = {
    res_parent = 0x0,
    res_policy = 0xffff885eb9f08100
  },
  th_list = {
    next = 0xffff885219687bd8,
    prev = 0xffff884a160c58d8
  },
  th_rule_lock = {
    {
      rlock = {
        raw_lock = {
          val = {
            counter = 1
          }
        }
      }
    }
  },
  th_rule_sequence = {
    counter = 607
  },
  th_rule = 0xffff884a160c58c0,
  th_timer = {
    node = {
      node = {
        __rb_parent_color = 18446612488267270960,
        rb_right = 0x0,
        rb_left = 0x0
      },
      expires = {
        tv64 = 1139545063897123
      }
    },
    _softexpires = {
      tv64 = 1139545063897123
    },
    function = 0xffffffffc0f80830 <nrs_tbf_timer_cb>,
    base = 0xffff885ebf993960,
    state = 0,
    start_pid = 18992,
    start_site = 0xffffffff810b65a2 <hrtimer_start+18>,
    start_comm = "mdt00_045\000\000\000\000\000\000"
  },
  th_deadline = 1139545063897123,
  th_sequence = 20564788676,
  th_binheap = 0xffff884105ed4060,
  th_cli_hash = 0xffff88b105ee2000,
  th_type = "nid\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000",
  th_ops = 0xffffffffc10485e0 <nrs_tbf_nid_ops>,
  th_type_flag = 2,
  th_purge_start = 0
}
```
th_rule = 0xffff884a160c58c0, 这个就是我们要找的nrs_tbf_rule，可以直接打印这个栈中的内容

```bash
crash> struct nrs_tbf_rule 0xffff884a160c58c0
struct nrs_tbf_rule {
  tr_name = "default\000\000\000\000\000\000\000\000",
  tr_head = 0xffff8852e0748b00,
  tr_linkage = {
    next = 0xffff8852e0748b10,
    prev = 0xffff885219687bd8
  },
  tr_nids = {
    next = 0xffff884a160c58e8,
    prev = 0xffff884a160c58e8
  },
  tr_nids_str = 0xffff8854bffade60 "*",
  tr_jobids = {
    next = 0x0,
    prev = 0x0
  },
  tr_jobids_str = 0x0,
  tr_opcodes = 0x0,
  tr_opcodes_str = 0x0,
  tr_conds = {
    next = 0x0,
    prev = 0x0
  },
  tr_conds_str = 0x0,
  tr_rpc_rate = 10000,
  tr_nsecs = 100000,
  tr_depth = 3,
  tr_rule_lock = {
    {
      rlock = {
        raw_lock = {
          val = {
            counter = 0
          }
        }
      }
    }
  },
  tr_cli_list = {
    next = 0xffff88533b6938a0,
    prev = 0xffff88533b6938a0
  },
  tr_flags = 2,
  tr_ref = {
    counter = 2
  },
  tr_generation = 0
}
```

接着找下一个rule，tr_linkage指向的就是下一个rule
```bash
找下一个             0xffff885219687bd8 - 0x18
struct nrs_tbf_rule 0xffff885219687bc0
```
所以下一个就直接找0xffff885219687bc0，因为struct nrs_tbf_rule前面有两个数据结构，所以需要减去24个字节，分别是
```c
/** Name of the rule. */
	char				 tr_name[MAX_TBF_NAME]; // 16个字节
	/** Head belongs to. */
	struct nrs_tbf_head		*tr_head;  // 8个字节
```
```bash
crash> struct nrs_tbf_rule 0xffff885219687bc0
struct nrs_tbf_rule {
  tr_name = "rule_clients\000\000\000",
  tr_head = 0xffff8852e0748b00,
  tr_linkage = {
    next = 0xffff884a160c58d8,
    prev = 0xffff8852e0748b10
  },
  tr_nids = {
    next = 0xffff885e1a280380,
    prev = 0xffff885e1a280380
  },
  tr_nids_str = 0xffff885e0f762a40 "xxx.[22-90]@o2ib",
  tr_jobids = {
    next = 0x0,
    prev = 0x0
  },
  tr_jobids_str = 0x0,
  tr_opcodes = 0x0,
  tr_opcodes_str = 0x0,
  tr_conds = {
    next = 0x0,
    prev = 0x0
  },
  tr_conds_str = 0x0,
  tr_rpc_rate = 3000,
  tr_nsecs = 333333,
  tr_depth = 3,
  tr_rule_lock = {
    {
      rlock = {
        raw_lock = {
          val = {
            counter = 0
          }
        }
      }
    }
  },
  tr_cli_list = {
    next = 0xffff885219687c60,
    prev = 0xffff885219687c60
  },
  tr_flags = 0,
  tr_ref = {
    counter = 1
  },
  tr_generation = 0
}
```

接着再往下找是struct nrs_tbf_rule 0xffff8852e0748af8

```bash
crash>  struct nrs_tbf_rule 0xffff8852e0748af8
struct nrs_tbf_rule {
  tr_name = "\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000",
  tr_head = 0xffff885eb9f08100,
  tr_linkage = {
    next = 0xffff885219687bd8,
    prev = 0xffff884a160c58d8
  },
  tr_nids = {
    next = 0x25f00000001,
    prev = 0xffff884a160c58c0
  },
  tr_nids_str = 0xffff8852e0748b30 "0\213t\340R\210\377\377",
  tr_jobids = {
    next = 0x0,
    prev = 0x0
  },
  tr_jobids_str = 0x40c6902bd3823 <Address 0x40c6902bd3823 out of bounds>,
  tr_opcodes = 0x40c6902bd3823,
  tr_opcodes_str = 0xffffffffc0f80830 <nrs_tbf_timer_cb> "\017\037D",
  tr_conds = {
    next = 0xffff885ebf993960,
    prev = 0x0
  },
  tr_conds_str = 0x4a30 <Address 0x4a30 out of bounds>,
  tr_rpc_rate = 18446744071579592098,
  tr_nsecs = 3760610349430367341,
  tr_depth = 53,
  tr_rule_lock = {
    {
      rlock = {
        raw_lock = {
          val = {
            counter = 45955107
          }
        }
      }
    }
  },
  tr_cli_list = {
    next = 0x4c9c1c5c4,
    prev = 0xffff884105ed4060
  },
  tr_flags = 99491840,
  tr_ref = {
    counter = -30543
  },
  tr_generation = 6580590
}
```
这个地址是有问题的，再往下找就是7bd8了，形成了一个双向链表

但是这三个rule的tr_flags都不会触发那个assert操作。。

已经将这个问题提交到社区，看看是否有人感兴趣
[https://jira.whamcloud.com/browse/LU-12582](https://jira.whamcloud.com/browse/LU-12582)





