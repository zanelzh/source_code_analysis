#select源码剖析

----------



首先，我们先了解几个数据结构。

	//fd_set_bits 结构体
	typedef struct {
		unsigned long *in, *out, *ex;
		unsigned long *res_in, *res_out, *res_ex;
	} fd_set_bits;

	//fd_set
	typedef __kernel_fd_set		fd_set;
	//这个 __kernel_fd_set 又是什么呢？
	typedef struct {
		unsigned long fds_bits [__FDSET_LONGS];
	} __kernel_fd_set;
	//原来是个数组，长度为__FDSET_LONGS，接着往下看
	#undef __NFDBITS
	#define __NFDBITS	(8 * sizeof(unsigned long))//一个long类型对应的位的个数，即字节数*8
	
	//1024个文件描述符，说明需要1024个位
	#undef __FD_SETSIZE
	#define __FD_SETSIZE	1024
	
	#undef __FDSET_LONGS
	#define __FDSET_LONGS	(__FD_SETSIZE/__NFDBITS)//1024个位需要的long类型个数

接下来，我们先看入口函数sys_select

	sys_select源码
	
下面来介绍do_select

首先了解一下需要的数据结构

poll_table

__NFDBITS


	do_select源码