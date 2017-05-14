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

	/* 判断文件描述符的事件，返回结果事件数量 */
	int do_select(int n, fd_set_bits *fds, long *timeout)
	{
		struct poll_wqueues table;
		poll_table *wait;
		int retval, i;
		long __timeout = *timeout;
		
		/* 加锁获取值最大的文件描述符 */
	 	spin_lock(&current->files->file_lock);
		retval = max_select_fd(n, fds);
		spin_unlock(&current->files->file_lock);
	
		if (retval < 0)
			return retval;
		/* n作为上限 */
		n = retval;
	
		/* 注册回调函数 */
		poll_initwait(&table);
		wait = &table.pt;
	
		if (!__timeout)
			wait = NULL;
		/* 初始化返回值为0，即无事件 */
		retval = 0;
		for (;;) {
			unsigned long *rinp, *routp, *rexp, *inp, *outp, *exp;
	
			set_current_state(TASK_INTERRUPTIBLE);
			
			/* 将之前在sys_select中设置的fds值，赋值给对应的指针 */
			inp = fds->in; outp = fds->out; exp = fds->ex;
			rinp = fds->res_in; routp = fds->res_out; rexp = fds->res_ex;
	
			/* 循环遍历事件，注意：rinp指针每次增加，单位为long */
			for (i = 0; i < n; ++rinp, ++routp, ++rexp) {
				unsigned long in, out, ex, all_bits, bit = 1, mask, j;
				unsigned long res_in = 0, res_out = 0, res_ex = 0;
				struct file_operations *f_op = NULL;
				struct file *file = NULL;
	
				in = *inp++; out = *outp++; ex = *exp++;
				/* 此段内（long长度）所有位的结果汇总 */
				all_bits = in | out | ex;
				/* 如果此段内的所有位都为0，直接下一次循环 */
				if (all_bits == 0) {
					i += __NFDBITS;
					continue;
				}
	
				for (j = 0; j < __NFDBITS; ++j, ++i, bit <<= 1) {
					/* 查看i是否越界 */
					if (i >= n)
						break;
					/* bit从第一个位开始验证all_bits的每一个位，如果当前位为0，直接下一次循环 */
					if (!(bit & all_bits))
						continue;
					/* ??? */
					file = fget(i);
					if (file) {
						/* ??? */
						f_op = file->f_op;
						mask = DEFAULT_POLLMASK;
						if (f_op && f_op->poll)
							/* retval第一次为0，将填入wait，后续有事件后，填入NULL */
							mask = (*f_op->poll)(file, retval ? NULL : wait);
						fput(file);
						/* 判断当前的文件描述符是否有读事件*/
						if ((mask & POLLIN_SET) && (in & bit)) {
							/* 返回结果中设置对应的位，并对返回结果增一 */
							res_in |= bit;
							retval++;
						}
						/* 查看写事件 */
						if ((mask & POLLOUT_SET) && (out & bit)) {
							res_out |= bit;
							retval++;
						}
						/* 异常 */
						if ((mask & POLLEX_SET) && (ex & bit)) {
							res_ex |= bit;
							retval++;
						}
					}
					/* 是否重新调度 */
					cond_resched();
				}
				/* 如果有读事件，将标志位的信息拷贝到fds里，后两者相同 */
				if (res_in)
					*rinp = res_in;
				if (res_out)
					*routp = res_out;
				if (res_ex)
					*rexp = res_ex;
			}
			wait = NULL;
			/* 遍历完成后，有事件产生，将退出外层循环 */
			if (retval || !__timeout || signal_pending(current))
				break;
			if(table.error) {
				retval = table.error;
				break;
			}
			__timeout = schedule_timeout(__timeout);
		}
		__set_current_state(TASK_RUNNING);
		/* ??? */
		poll_freewait(&table);
	
		/*
		 * Up-to-date the caller timeout.
		 */
		*timeout = __timeout;
		/* 返回产生事件的文件描述符个数 */
		return retval;
	}