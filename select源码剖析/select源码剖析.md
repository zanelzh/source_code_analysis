# select源码剖析

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

	asmlinkage long
	sys_select(int n, fd_set __user *inp, fd_set __user *outp, fd_set __user *exp, struct timeval __user *tvp)
	{
		fd_set_bits fds;
		char *bits;
		long timeout;
		int ret, size, max_fdset;
	
		timeout = MAX_SCHEDULE_TIMEOUT;
		if (tvp) {
			time_t sec, usec;
	
			if ((ret = verify_area(VERIFY_READ, tvp, sizeof(*tvp)))
				|| (ret = __get_user(sec, &tvp->tv_sec))
				|| (ret = __get_user(usec, &tvp->tv_usec)))
				goto out_nofds;
	
			ret = -EINVAL;
			if (sec < 0 || usec < 0)
				goto out_nofds;
	
			if ((unsigned long) sec < MAX_SELECT_SECONDS) {
				timeout = ROUND_UP(usec, 1000000/HZ);
				timeout += sec * (unsigned long) HZ;
			}
		}
	
		ret = -EINVAL;
		/* 判断文件描述符数量 */
		if (n < 0)
			goto out_nofds;
	
		/* max_fdset can increase, so grab it once to avoid race */
		/* 获取打开文件的最大文件描述符数量 */
		max_fdset = current->files->max_fdset;
		/* 如果超过最大值，n改为最大值 */
		if (n > max_fdset)
			n = max_fdset;
		
		/*
		 * We need 6 bitmaps (in/out/ex for both incoming and outgoing),
		 * since we used fdset we need to allocate memory in units of
		 * long-words. 
		 */
		ret = -ENOMEM;
		/* 获取需要字节大小 */
		size = FDS_BYTES(n);
		/* 调用kmalloc分配6*size大小的空间 */
		bits = select_bits_alloc(size);
		/* 检查分配结果，失败的话退出 */
		if (!bits)
			goto out_nofds;
		/* 用bits填充fds中的指针，将bits对应空间分为6份，后边直接拿fds来进行操作 */
		fds.in      = (unsigned long *)  bits;
		fds.out     = (unsigned long *) (bits +   size);
		fds.ex      = (unsigned long *) (bits + 2*size);
		fds.res_in  = (unsigned long *) (bits + 3*size);
		fds.res_out = (unsigned long *) (bits + 4*size);
		fds.res_ex  = (unsigned long *) (bits + 5*size);
		/* 从用户空间拷贝数据，即获得fd信息 */
		if ((ret = get_fd_set(n, inp, fds.in)) ||
			(ret = get_fd_set(n, outp, fds.out)) ||
			(ret = get_fd_set(n, exp, fds.ex)))
			goto out;
		/* 对返回的所有标志位清零，接下来将要设置其标志位 */
		zero_fd_set(n, fds.res_in);
		zero_fd_set(n, fds.res_out);
		zero_fd_set(n, fds.res_ex);
	
		/* 执行do_select ,接下来将介绍do_select */
		ret = do_select(n, &fds, &timeout);
	
		if (tvp && !(current->personality & STICKY_TIMEOUTS)) {
			time_t sec = 0, usec = 0;
			if (timeout) {
				sec = timeout / HZ;
				usec = timeout % HZ;
				usec *= (1000000/HZ);
			}
			put_user(sec, &tvp->tv_sec);
			put_user(usec, &tvp->tv_usec);
		}
	
		if (ret < 0)
			goto out;
		if (!ret) {
			ret = -ERESTARTNOHAND;
			if (signal_pending(current))
				goto out;
			ret = 0;
		}
	
		/* 结果在fds中，将结果返回给用户空间 */
		if (set_fd_set(n, inp, fds.res_in) ||
			set_fd_set(n, outp, fds.res_out) ||
			set_fd_set(n, exp, fds.res_ex))
			ret = -EFAULT;
	
	/* 释放之前申请的空间 */
	out:
		select_bits_free(bits, size);
	/* 返回产生事件的文件描述符个数或错误码 */
	out_nofds:
		return ret;
	}
	
下面来介绍do_select

首先了解一下需要的数据结构

poll_table

	asmlinkage long
	sys_select(int n, fd_set __user *inp, fd_set __user *outp, fd_set __user *exp, struct timeval __user *tvp)
	{
		fd_set_bits fds;
		char *bits;
		long timeout;
		int ret, size, max_fdset;
	
		timeout = MAX_SCHEDULE_TIMEOUT;
		if (tvp) {
			time_t sec, usec;
	
			if ((ret = verify_area(VERIFY_READ, tvp, sizeof(*tvp)))
				|| (ret = __get_user(sec, &tvp->tv_sec))
				|| (ret = __get_user(usec, &tvp->tv_usec)))
				goto out_nofds;
	
			ret = -EINVAL;
			if (sec < 0 || usec < 0)
				goto out_nofds;
	
			if ((unsigned long) sec < MAX_SELECT_SECONDS) {
				timeout = ROUND_UP(usec, 1000000/HZ);
				timeout += sec * (unsigned long) HZ;
			}
		}
	
		ret = -EINVAL;
		/* 判断文件描述符数量 */
		if (n < 0)
			goto out_nofds;
	
		/* max_fdset can increase, so grab it once to avoid race */
		/* 获取打开文件的最大文件描述符数量 */
		max_fdset = current->files->max_fdset;
		/* 如果超过最大值，n改为最大值 */
		if (n > max_fdset)
			n = max_fdset;
		
		/*
		 * We need 6 bitmaps (in/out/ex for both incoming and outgoing),
		 * since we used fdset we need to allocate memory in units of
		 * long-words. 
		 */
		ret = -ENOMEM;
		/* 获取需要字节大小 */
		size = FDS_BYTES(n);
		/* 调用kmalloc分配6*size大小的空间 */
		bits = select_bits_alloc(size);
		/* 检查分配结果，失败的话退出 */
		if (!bits)
			goto out_nofds;
		/* 用bits填充fds中的指针，将bits对应空间分为6份，后边直接拿fds来进行操作 */
		fds.in      = (unsigned long *)  bits;
		fds.out     = (unsigned long *) (bits +   size);
		fds.ex      = (unsigned long *) (bits + 2*size);
		fds.res_in  = (unsigned long *) (bits + 3*size);
		fds.res_out = (unsigned long *) (bits + 4*size);
		fds.res_ex  = (unsigned long *) (bits + 5*size);
		/* 从用户空间拷贝数据，即获得fd信息 */
		if ((ret = get_fd_set(n, inp, fds.in)) ||
			(ret = get_fd_set(n, outp, fds.out)) ||
			(ret = get_fd_set(n, exp, fds.ex)))
			goto out;
		/* 对返回的所有标志位清零，接下来将要设置其标志位 */
		zero_fd_set(n, fds.res_in);
		zero_fd_set(n, fds.res_out);
		zero_fd_set(n, fds.res_ex);
	
		/* 执行do_select ,接下来将介绍do_select */
		ret = do_select(n, &fds, &timeout);
	
		if (tvp && !(current->personality & STICKY_TIMEOUTS)) {
			time_t sec = 0, usec = 0;
			if (timeout) {
				sec = timeout / HZ;
				usec = timeout % HZ;
				usec *= (1000000/HZ);
			}
			put_user(sec, &tvp->tv_sec);
			put_user(usec, &tvp->tv_usec);
		}
	
		if (ret < 0)
			goto out;
		if (!ret) {
			ret = -ERESTARTNOHAND;
			if (signal_pending(current))
				goto out;
			ret = 0;
		}
	
		/* 结果在fds中，将结果返回给用户空间 */
		if (set_fd_set(n, inp, fds.res_in) ||
			set_fd_set(n, outp, fds.res_out) ||
			set_fd_set(n, exp, fds.res_ex))
			ret = -EFAULT;
	
	/* 释放之前申请的空间 */
	out:
		select_bits_free(bits, size);
	/* 返回产生事件的文件描述符个数或错误码 */
	out_nofds:
		return ret;
	}