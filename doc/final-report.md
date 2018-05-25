# final report goes here


## 已有相关工作介绍

### kAFL

#### 背景介绍

[kAFL: Hardware-Assisted Feedback Fuzzing for OS Kernels](https://github.com/RUB-SysSec/kAFL)是另一个fuzzing方向的工作。在本文中，作者考虑到Intel PT技术产生的trace信息数量庞大，因此使用了硬件支持的方法：将trace解析处理在机器内完成，以减少机器与外界的数据传输量。

不过，实际上作者并没有真正做出一个具有解析工具的硬件，而是使用了qemu虚拟硬件的方式完成所需功能。借由改写qemu源码（下称qemu-PT），实现了从虚拟机中获取数据、并通过共享内存的方式提供给外部程序使用。这部分正是我们为了实现fuzzing所需要的功能。

#### 工作原理

kAFL使用qemu-kvm作为工具，利用hypercall机制完成硬件反馈。hypercall是linux kernel中kvm模块所提供的调用方法，依靠于Intel x86硬件提供的vmx硬件虚拟化功能。在硬件虚拟化执行过程中，指令实际上是在host硬件中直接运行的，这样可以大大减小模拟器模拟指令运行的计算量，提高运行效率。

在运行过程中，x86硬件提供了完整的虚拟机控制功能。例如，可以使用vmcall这一指令触发VM Exit操作，从虚拟机中离开。此时，linux内核将捕获这一操作，然后回到虚拟机中，即触发VM Entry。而VM Entry进入点在qemu中，因此qemu可以获取hypercall的内容做出反应。这就是hypercall执行的全过程。

原工作中，作者为qemu添加了一个自定义的虚拟设备，这一设备并不直接和硬件相连，而更像是把“接入设备”这一设置作为模块加载的途径，用于在qemu启动时传入必要的参数：在qemu命令行中为这一设备设置参数，将host上的一文件名传入，即可通过对这一文件名进行mmap来完成进程间内存共享。为了完成fuzzer所需的各种远程调用，kAFL实现了相当多的hypercall指令。对我们而言，只需要其中内存拷贝的功能。





## 代码修改

### kAFL

我们修改了kAFL中qemu与linux kvm模块的实现，以供我们使用。在原本的设计中，fuzzer与executor共有4条数据交互的渠道：

1. syz-fuzzer -> executor的管道(in pipe)，负责发送指令（包括flags, 一次执行的指令长度，配置等）。启动时会进行一次handshake，发送一个magic number，之后每次执行一段测试都会发送执行命令。如果共享内存不可用（如Windows等），这个管道也可以用来传输fuzzing程序本体数据。
2. executor -> syz-fuzzer的管道(out pipe)，负责在程序执行完毕后对in pipe发来的指令进行回复，只有简单的信息，即执行结果、executor状态。
3. syz-fuzzer -> executor的共享内存(in memory)，负责放置运行的fuzzing程序指令数据。
4. executor -> syz-fuzzer的共享内存(out memory)，负责存放coverage数据以及统计数据。

因此，我们需要提供4个数据传输接口，其中in/out pipe两个管道注重实时性，而in/out memory则更注重传输速度。不过为了便捷，我们都统一使用kAFL的共享内存机制实现。

为了使用kAFL功能，在host OS - host linux kernel - qemu - guest OS四处都要添加支持。其中，host OS，guest OS这两部分需要修改syzkaller原本的数据传输方式，在之后再另予说明。

#### host linux kernel

需要修改linux kvm模块，在捕获VM Exit事件时查看由x86硬件填充的VMCS结构，根据传入的rax, rbx, rcx寄存器填充exit reason字段；这一部分kAFL原本就已经有了实现，将数据传输接口扩容到4个即可。

#### qemu

这一部分需要修改的比较多。首先，需要将涉及到Intel PT的大量代码从运行流程中删去，因为我们不支持使用这一功能；此外，也需要把接口和数据缓冲区扩容到4个。

其具体实现其实非常简单，qemu将外部命令行传入的文件名打开并mmap映射到一片内存上，在接收到hypercall时就从虚拟机的虚拟内存中读取、写入对应段的数据。因此，虽然进程间是内存共享的，但是实际上要传入/读入数据时必须guest OS主动调用才可以将host上的数据更新到guest的虚拟内存当中。

#### 测试

在ucore+的用户库中添加了hypercall支持文件，并添加了hypercalltest.c以及一个测试脚本用于进行数据传输测试：

在ucore目录下执行`test_mmap.py`，打开qemu；在qemu中执行/bin/hypercalltest. 命令行与qemu中都应显示Hello world，说明双向传输成功。



### syz-manager

如前文所述，syz-manager是syzkaller的入口程序，负责开启子进程、对工程整体进行管理。为了修改运行架构，我们需要对这一部分进行修改。

修改的部分主要在子程序调用部分。原本syz-manager是通过scp将二进制文件拷贝入guest OS中，并使用ssh远程运行，另外使用RPC与远端进行数据交互。由于我们将syz-fuzzer移出，不再需要远程启动程序， 直接在本地运行即可。但RPC机制则不受影响，可以完全保留。

此外，也需要对vm/qemu模块进行修改，需要使用我们生成的qemu-PT，并需要传入kAFL虚拟设备的命令行参数。

除此之外，都无需进行修改。



### syz-fuzzer

这一部分，我们需要更改fuzzer与executor的交互逻辑。如前文所述，syz-fuzzer和executor的数据通路改为用共享内存完成。因此，这里做的就是将设备文件打开，并使用mmap形成共享内存。

此外，用共享内存实现pipe的逻辑会稍有不同，不能直接使用pipe的读、写操作，无数据读入时也不会产生阻塞。因此，改用在一个协程中轮询访问的方式进行读写。

另外，syz-fuzzer本应运行在虚拟机内，因此其启动时会试图检测操作系统提供的功能（例如，访问/sys/kernel/debug/kmemleak，以确认是否有memleak检测功能；访问kcov，确认是否有覆盖率检测工具等），这些功能我们只能暂时放弃，毕竟ucore+并不能提供其查询的功能。



### Syscall description

另一部分需要完成的是系统调用描述生成工作。

系统调用需要使用syzkaller规定的语法存于sys/ucore包中，该语法在[此文档](https://github.com/maoyuchaxue/syzkaller/blob/master/docs/syscall_descriptions_syntax.md)中有详细说明。系统调用描述的主要目的是为了为各系统调用的参数指定类型以及相互关系。

我们目前纳入测试范围的系统调用描述在[这里](https://github.com/maoyuchaxue/syzkaller/blob/ucore_porting/sys/ucore/basic.txt)以及[这里](https://github.com/maoyuchaxue/syzkaller/blob/ucore_porting/sys/ucore/signal.txt)。涵盖了常用的系统调用。

我们以open作为一个系统调用的例子来说明描述语法:

```
include <file.h>
resource fd[int32]: 0xffffffffffffffff, 0
open_flags = O_RDONLY, O_WRONLY, O_RDWR, O_APPEND, O_FSYNC, O_ASYNC, O_CREAT, O_EXCL, O_NOCTTY, O_NONBLOCK, O_SYNC, O_TRUNC
SYS_open(file ptr[in, filename], flags flags[open_flags]) fd
```

其中，resource fd指明了文件描述符为一种资源，并根据`SYS_open`的定义，得知该系统调用需要传入文件名字符串指针、open flags，并返回一个文件描述符fd。这样，在随机生成调用序列时，就会将之前open得到的返回值作为新的fd传入write, read等其他文件相关系统调用中。

此外，实际工作时syzkaller会先根据描述文件产生一个c文件并编译运行，其功能是对描述文件中指明的include头文件引入，然后输出描述中涉及的各常量的值(如上面的`O_RDONLY`)，以供之后使用。

写完描述文件之后，还需要使用syzkaller提供的工具，将描述文件翻译为syz-fuzzer所需的gen.go文件以及executor所需的syscalls\_ucore.h文件。这一工具(syz-extract)也需要我们补充完成。基本上，由于ucore+与Linux类似，照抄linux平台的生成脚本即可。最关键的是需要指明头文件的include路径。



## 实验结果

syzkaller作为不注重语义的fuzzing工具，寻找的bug多半为鲁棒性角度的bug，例如对于非法输入无法处理、导致数据被破坏或直接导致内核陷入panic等，都属于bug的一类。

我们完成移植的时间较晚，因此能够来得及确认原因的bug并不多。不过，我们确认了的bug，都添加了会重现bug的测试代码、并进行了修复。在此我们把找出的bug列出：

+ `sys_mmap`与`sys_shmem`在请求空间重叠时错误：

```
a = 0x4000000000;
ret = sys_mmap(&a, 100, 0);
b = a - 1;
ret = sys_mmap(&b, 101, 0);
```

这里第二次请求的内存区间包含第一次请求区间，导致kernel panic。原代码中仅对第一次请求区间包含第二次请求区间时进行了判断，但反之则没有。

+ `vfs_lookup_parent`当路径为'/'时错误：

该函数广泛用于open, unlink, rename, link, symlink, mkdir等各种系统调用中。其代码如下:

```
int vfs_lookup_parent(char *path, struct inode **node_store, char **endp)
{
  int ret;
	struct inode *node;
  //TODO: Security issue: this may lead to buffer overflow.
  static char subpath[1024];
	if ((ret = get_device(path, subpath, &node)) != 0) {
		return ret;
	}
	ret =
	    (*path != '\0') ? vop_lookup_parent(node, subpath, node_store,
						endp) : -E_INVAL;
	vop_ref_dec(node);
	return ret;
}
```

可见，其仅判断了`path != '\0'`，而其调用的`sfs_lookup_parent`则是：

```
static int
sfs_lookup_parent(struct inode *node, char *path, struct inode **node_store,
		  char **endp)
{
	struct sfs_fs *sfs = fsop_info(vop_fs(node), sfs);
	assert(*path != '\0' && *path != '/');
    // ....
}
```

这里要求`path != '/'`. 调用者和被调用者对条件的理解不一致，导致了这个错误。

+ `sys_dup`当fd2非法时报错

`sys_dup`的实现如下:

```
int file_dup(int fd1, int fd2)
{
	int ret;
	struct file *file;

  //If fd1 is invalid, return -E_BADF.
	if ((ret = fd2file(fd1, &file)) != 0) {
		return ret;
	}

  //fd1 and fd2 cannot be the same
  if (fd1 == fd2) {
    return -E_INVAL;
  }

  struct file_desc_table *desc_table = fs_get_desc_table(current->fs_struct);

  //If fd2 is an opened file, close it first. This is what dup2 on linux does.
  struct file *file2 = file_desc_table_get_file(desc_table, fd2);
  if(file2 != NULL) {
		kprintf("file_desc_table_get_unused called by dup2!\n");
    		file_desc_table_dissociate(desc_table, fd2);
  }

  //If fd2 is NO_FD, a new fd will be assigned.
  if (fd2 == NO_FD) {
    fd2 = file_desc_table_get_unused(desc_table);
  }

  //Now let fd2 become a duplication for fd1.
  file_desc_table_associate(desc_table, fd2, file);
  

  //fd2 is returned.
	return fd2;
}
```

该函数未对fd2进行任何检查就调用了`file_desc_table_dissociate(desc_table, fd2)`, 当fd2=-1时会导致panic.

+ `sys_linux_sigaction`栈溢出攻击漏洞

该系统调用实现如下:

```
int do_sigaction(int sign, const struct sigaction *act, struct sigaction *old)
{
	assert(get_si(current)->sighand);
#ifdef __SIGDEBUG
	kprintf("do_sigaction(): sign = %d, pid = %d\n", sign, current->pid);
#endif
	struct sigaction *k = &(get_si(current)->sighand->action[sign - 1]); // vulnerable
	if (k == NULL) {
		panic("kernel thread call sigaction (i guess)\n");
	}
	int ret = 0;
	struct mm_struct *mm = current->mm;
	lock_mm(mm);
	if (old != NULL && !copy_to_user(mm, old, k, sizeof(struct sigaction))) {
		unlock_mm(mm);
		ret = -E_INVAL;
		goto out;
	}
  //....
}
```

这里，`sighand->action`是一个长度为64的数组，当`sign>64`时，该设置操作会导致操作系统正常的数据遭到覆盖。

+ `sys_linux_mmap`当fd无效时错误

同样地，该系统调用未对传入的fd进行检查，当无效的文件描述符传入时将导致kernel panic.

+ `sys_read, sys_write`使用用户态无效内存时错误

当用户态调用这些系统调用，并传入过大的len参数时，易于导致访问未分配给用户的地址空间。这一操作本应使操作终止并产生SIGSEGV信号，但ucore+并没有这一机制，而是直接进入kernel panic.



## 后续设想

这一工作还有很多不足，有很多方面可以进行扩展。例如：

+ 添加更多系统调用的测试，修改框架使得其可以执行fast syscall；
+ 添加kcov功能，正确获取分支条件、覆盖率等信息，使得syzkaller可以更有方向地生成测例；
+ 添加更多方向的测试，例如多线程的测试、网络栈的测试、多文件系统的测试等；



## 实验过程日志

见[相关wiki页面](http://os.cs.tsinghua.edu.cn/oscourse/OS2018spring/projects/g07).



## 实验总结

总体来说我们这次实验还有相当的不足，syzkaller原本有着完成从多线程、多虚拟机fuzzing到结果统计、存储、重现的一整套完整流程的能力，在阅读源码、分析实现时也感受到了其功能的全面。但由于ucore实验条件、时间上的各种限制，很多功能我们只能舍弃掉，最后只留下了核心的测试功能。

此外，在实验中遇到了不少之前没能预料到的问题，例如ucore+极少的用户库支持、薄弱的基本功能等等，解决这些问题也消耗了不少时间。另外，在这种运行环境下完全无法使用gdb来调试ucore+的运行情况，使得调试工作比较困难。不过，最后寻找bug、修复问题的过程还是非常有趣的。
