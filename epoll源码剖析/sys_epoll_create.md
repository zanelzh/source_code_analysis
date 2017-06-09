# sys_epoll_create

![](pic/sys_epoll_create.png)
首先参数size，其实后边并没有实际使用，所以，用户设置的size并不会影响实际存储的fd数量。

![](pic/sys_epoll_create2.png)

接下来进入ep_getfd函数。

![](pic/ep_getfd.png)
首先，获取一个struct file，底层通过slab分配。

![](pic/ep_getfd2.png)
从eventpoll文件系统分配一个inode结点。

![](pic/ep_getfd3.png)
分配一个空闲的文件描述符，留着后边将之前的struct file绑定。

![](pic/ep_getfd4.png)
分配dentry，配置struct file，将fd和file绑定。

回到sys_epoll_create

![](pic/sys_epoll_create3.png)
接下来进入ep_file_init

进入函数前先了解一下struct eventpoll
![](pic/eventpoll.png)
sem  读写信号量
wq sys_epoll_wait的等待队列
poll_wait  poll方法的等待队列  
rdllist 就绪文件描述符链  
rbr 红黑树根节点  

![](pic/ep_file_init.png)
通过kmalloc分配struct eventpoll的内存空间，然后memset置零。  
初始化ep各个成员，将其赋给file的private_data  

最后sys_epoll_create返回其文件描述符。