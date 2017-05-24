# execve()系统调用剖析

内核入口sys_execve()

regs.ebx为第一个参数，参数为指向字符串的指针  
字符串为绝对路径，系统堆栈约7KB，所以分配缓冲区存字符串

找到可执行文件并打开 open_exec()

装入可执行程序  
struct linux_binprm 变量名为bprm  
bprm保存打开的可执行文件的file指针  
bprm.sh_bang 可执行文件的性质 二进制文件 shell脚本文件  

处理参数和环境变量  
每个参数的最大长度为一个物理页面
bprm有一个页面指针数组
count()统计arg[]参数个数

当前进程是否有对目标文件操作的权限以及目标文件的可执行属性
读入开头128字节存入bprm缓冲区
argv[] envp[]从用户空间拷贝到bprm

formats队列中的成员逐个尝试load_binary()函数

linux_binfmt
a.out开头为struct exec

N_TEXTOFF()取得正文在目标文件的的起始位置

放弃从父进程继承下来的用户空间
信号处理表
exec_mmp()
分配mm_struct以及页面目录
activate_mm()切换到新的用户空间
对已打开文件的处理

magic number 为OMAGIC时，表示可执行代码为“非纯代码”
do_brk()为正文段和数据段合在一起分配空间，然后读进来
非OMAGIC的三种纯代码，直接将可执行文件映射到进程的用户空间。
do_mmap()分别将正文段、数据段映射到用户空间。

do_brk() bss段初始为0

堆栈段
一组页面为执行参数和环境变量，一个页面为一个变量

start_thread()

以上为a.out文件，即二进制文件的执行，接下来为字符文件（shell文件）的执行
递归

