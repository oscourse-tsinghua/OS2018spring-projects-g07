# syzkaller in ucore



## 1. 实验概述

​	Bug，一个由短短的三个字母组成的词汇，却使得世界上所有的程序员恨之入骨，为此咬牙切齿，愁眉苦脸。一个bug的存在也许来源于一个设计的漏洞，一两行代码的错误，亦或是对一个边界情况的疏忽，但无论源自何方，它们却都可能具备着毁天灭地的能力。正因如此，bug已经当之无愧地成为了所有程序员共同面对的终极大敌。

​	历史的规律告诉我们，bug所能带来的麻烦往往和代码量的平方成正比，对于一个100行的算法小程序而言，一个错误所带来的威胁往往可以接受（比赛的情况除外），然而一旦涉及到大规模的代码工程，一个bug可能导致整个系统的崩溃，后果令人不堪设想。另一方面，程序代码量的提升为bug带来了更加充裕的生存空间，在每个函数的角落里，每个语句的前前后后，都可能是一片藏污纳垢之地，潜藏着危机和不定，难以发现和清除。随着信息技术的发展，许多大型工程的代码量已经相当之高，通过人工对其调试的方法也逐渐变得无法接受，在这样的需求下，聪明的工程师们提出了一种全新的思路：利用软件方法进行debug，即通过程序来调试程序。

​	在众多大型工程的行列之中，操作系统属于最为重要的成员之一，它的复杂度极高，运用范围广，对bug的容忍度非常之低，因此如何对操作系统进行测试，一直是众多研究者们的重点关注对象。我们此次实验中所使用的Syzkaller，正是一套用于测试操作系统内核的工具，由Google团队开发，目前仍在维护建设中。Syzkaller主要针对的操作系统是linux，并在此基础上，逐渐加入了对Windows、FreeBSD、Akaros等其他操作系统的支持。

​	以上的操作系统都是目前市面上主流的操作系统，其完成度较高，支持的功能也十分丰富，Syzkaller的设计思路也主要是针对在较为成熟的操作系统中来进行测试而考虑的。我们的工作将Syzkaller其移植到了uCore操作系统之中，相比之前的系统而言，这是一个设计简单、功能较少、稳定度相对较差的操作系统。为了能够让其在uCore中得以运行，我们对Syzkaller的结构进行了大量的调整，并引进了另一个用于测试的项目——kAFL中的部分工作，最终实现了在uCore中进行测试的功能。由于设计缘故，Syzkaller的一部分代码（executor）需要在uCore的用户态中运行，这可能也是迄今为止uCore中所跑过的最大的用户程序（包括3个进程，数千行代码）。经过一定时间的测试，我们目前已经从uCore中找到了共6个bug，并完成了对这些bug的调试与修复。

​	由于uCore本身支持的功能实在过少，导致我们在移植的过程中删掉了很多重要的功能模块，使得测试工具的综合能力被大打折扣。在接下来的工作中，如果能将uCore的相应支持予以实现，那么可以进一步加入这些功能，从而对uCore进行更全面更深入的测试。总体而言，Syzkaller有着非常巨大的潜力还尚未发掘，值得我们对其进一步探索和完善。

​	接下来，我们将详细介绍在此次实验中的原理、移植策略、分工、代码实现细节、以及面临的主要挑战等几个主要的实验部分。



##2. 已有相关工作介绍

### Syzkaller

​	Syzkaller是本次实验中的基础和主要部分，因此首先必须对其原理进行一定的介绍。大体而言，Syzkaller是一套对内核进行模糊测试的工具，所谓模糊测试，即通过一系列向目标系统提供非预期的输入并监视异常结果从而发现漏洞的方法[1]。进行模糊测试往往需要两个不同的操作系统——主操作系统（host）和目标操作系统（guest，又称target）共同协作，两个系统中都需要运行相应的程序，测试的过程则一般为：主操作系统中的程序不断生成并发送输入文件，目标操作系统中的程序接受程序、执行并返回结果，再通过主操作系统程序进行检查。Syzkaller同样采用了这样的模式，下图是它的基本结构[2]：

![process_structure](process_structure.png)

​	总体来说，最终用于测试Syzkaller的程序部分一共分为三块，分别被称为syz-manager、syz-fuzzer和syz-executor。其中，syz-manager在host操作系统中运行，其余两者则在target操作系统中运行。当用户开始运行起manager时，它会依次建立起syz-manager、corpus数据库（用于存放崩溃的报告和一些特殊的程序）、以及一个RPC服务器（用于和fuzzer通信）、一个http服务器（用于向用户展示结果）。在这些事务处理完成后，manager将会建立一到多个VM（在这里采用的是qemu）搭载target操作系统，并利用target操作系统中的ssh接口触发syz-fuzzer。当fuzzer运行之后，会主动和manager建立起RPC协议链接，然后开始运行syz-executor。当一切准备就绪后，syz-fuzzer将会通过在编译时已经生成的关于target操作系统的系统调用描述（下称description）自动生成出用于测试的输入程序（本质上由一系列的syscall组成），然后送至syz-executor中运行。最后，syz-fuzzer会通过syz-executor的运行情况得出是否发生了错误，同时通过target系统中的KCOV功能得到代码覆盖的情况，并将其返送给syz-manager。

​	因为涉及到很多多进程、ipc操作，Syzkaller的大部分程序是利用go语言写成的。syz-executor是唯一的例外，它由C++语言写成，原因主要是C++语言可以直接使用系统调用的函数，这对于测试系统调用而言是必须拥有的一点要求。

​	在Syzkaller一整套的运行系统当中，最为至关重要的一环在于如何生成测试所用的输入数据。首先，用户需要手动将target操作系统中支持的所有系统按照Syzkaller提供的格式进行描述（具体实现细节见后部分），以确定target系统中的所有调用和这些调用中每个参数的属性。Syzkaller会将其和target操作系统中的代码进行结合，转换为数个自动生成的文件，作为syz-fuzzer和syz-executor的组成部分。在syz-fuzzer中，将会通过之前填入的属性生成随机参数，然后由多个syscall组成一个execute输入文件送给syz-executor。后者收到后，会检查syz-fuzzer输入过来的execute中的每条syscall对应的具体是哪一个用户函数，然后将其执行。

​	最后是关于代码覆盖率的维护。Syzkaller关于代码覆盖率的实现主要通过KCOV（一套代码覆盖测试工具）而完成。因为KCOV已经集成在linux中，因此只需要由syz-fuzzer进行相关的调用即可。

### kAFL

#### 背景介绍

[kAFL: Hardware-Assisted Feedback Fuzzing for OS Kernels](https://github.com/RUB-SysSec/kAFL)[4]是另一个fuzzing方向的工作。在本文中，作者考虑到Intel PT技术产生的trace信息数量庞大，因此使用了硬件支持的方法：将trace解析处理在机器内完成，以减少机器与外界的数据传输量。

不过，实际上作者并没有真正做出一个具有解析工具的硬件，而是使用了qemu虚拟硬件的方式完成所需功能。借由改写qemu源码（下称qemu-PT），实现了从虚拟机中获取数据、并通过共享内存的方式提供给外部程序使用。这部分正是我们为了实现fuzzing所需要的功能。

#### 工作原理

kAFL使用qemu-kvm作为工具，利用hypercall机制完成硬件反馈。hypercall是linux kernel中kvm模块所提供的调用方法，依靠于Intel x86硬件提供的vmx硬件虚拟化功能。在硬件虚拟化执行过程中，指令实际上是在host硬件中直接运行的，这样可以大大减小模拟器模拟指令运行的计算量，提高运行效率。

在运行过程中，x86硬件提供了完整的虚拟机控制功能。例如，可以使用vmcall这一指令触发VM Exit操作，从虚拟机中离开。此时，linux内核将捕获这一操作，然后回到虚拟机中，即触发VM Entry。而VM Entry进入点在qemu中，因此qemu可以获取hypercall的内容做出反应。这就是hypercall执行的全过程。

原工作中，作者为qemu添加了一个自定义的虚拟设备，这一设备并不直接和硬件相连，而更像是把“接入设备”这一设置作为模块加载的途径，用于在qemu启动时传入必要的参数：在qemu命令行中为这一设备设置参数，将host上的一文件名传入，即可通过对这一文件名进行mmap来完成进程间内存共享。为了完成fuzzer所需的各种远程调用，kAFL实现了相当多的hypercall指令。对我们而言，只需要其中内存拷贝的功能。



##3. 实验设计

​	在上一章节中，我们已经介绍了Syzkaller在对Linux系统进行测试的情况。为了检验Syzkaller移植的可行性，首先需要分析Syzkaller运行所需提供的主要支持，以及它们在uCore中的实现情况：

* 支持C++语言 无（支持C语言）

* 支持Golang语言 无

* 支持ssh协议 无（有lwIP网络栈）

* 支持RPC协议 无（有lwIP网络栈）

* 支持ext4文件系统 无（支持不完全的FAT文件系统）

* （可选）KCOV工具 无

* （可选）支持多进程、多线程 有，不完善

   可以看出，如果要在移植的过程中保持Syzkaller原结构不变的话，就需要uCore满足以上的众多支持。其中，第一、第五项支持比较容易替换，第六、第七项支持可以暂时忽略，而要实现剩下的部分是相当棘手的。满足第三、第四项支持需要加入新的模块，尽管ssl协议已经有了在lwIP上的实现，但这和我们的要求相比还相距甚远。而最为致命的，在于第二项需求。关于Go语言在uCore上的移植，在数年之前已经有过简单（Hello world+信号量）的实现，但当时的Go版本仅是一个测试版本[3]，和Syzkaller所需的版本（1.8）相比已有极大的差距。综合以上的分析，我们最终放弃了这一思路。

   幸运的是，由于syz-executor的功能限制，它采用的是C++语言实现，因此将其移植入uCore中是可行的。然而对于syz-fuzzer来说，考虑到它庞大的代码量和复杂程度（主要是由Go语言的性质所致），我们也基本不太可能无法对其进行C语言重构。但我们却可以采用更加省事的一个办法解决这个问题——将其移动到host操作系统中运行。

   从上面的原理图中可以看出，在syz-fuzzer移出uCore之后，ssh和RPC的交流也变成了host操作系统内部的操作，因此我们不再需要使用uCore的网络支持了。由于暂时忽略了KCOV，关于coverage info的读取操作也可以省掉，剩下唯一的问题便是在syz-fuzzer和syz-executor之间的交流了。syz-fuzzer和syz-executor的交流涉及到输入程序，因此从数量上来讲是相当大的，考虑到这个原因，我们需要采用一种较为高效、稳定的通信机制完成这一部分。

   一种常见的想法是通过vm的stdin和stdout进行交流，但这种方式很快便被我们否决了。主要的原因在于，syz-fuzzer和syz-executor两者在工作之时均有着多个进程在运行，而这些进程不可能仅仅通过一个文件管道进行交流。在Syzkaller的实现方式中，两者的交流主要是通过mmap来实现的，为了能够直接利用这点性质，我们最终采用了另一个项目的成果：kAFL中的hypercall技术。这一技术能够通过修改qemu和linux，从而target操作系统和host操作系统的mmap内存共享机制。采用了这一技术之后，我们的主要问题也都得到了解决。下图是我们最终采纳的设计方案：

   ​

   ![newsyzkaller](/home/kzf/osLab/doc/newsyzkaller.png)



## 4. 小组分工

详见http://os.cs.tsinghua.edu.cn/oscourse/OS2018spring/projects/g07




## 5. 具体实现

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



### syz-executor
​	有关syz-executor的具体情况，请查看文档[executor-in-ucore](executor-in-ucore.md)，下面的内容节选自其中：

#### 起始进程

​	当syz-executor开始运行的时候，会先依次建立以下几步操作，然后才开始创建sandbox：

​	首先是建立和syz-fuzzer的通信，关于这一点，linux版本和我们的版本有着很大的区别。在linux版本中，由于syz-executor和fuzzer都运行于目标操作系统，syz-executor是作为fuzzer子进程的形式出现的。而通过子进程共享的文件描述符（基于Copy-on-Write获得的特性），syz-executor只需通过mmap就可以利用文件直接和fuzzer进行交流了。而在uCore版本中，syz-executor和fuzzer运行于不同的两个操作系统中，他们的交流方式则是通过取自与kAFL的hypercall来实现的，因此在这儿要进行一些修改。关于两种不同通信方式的具体实现，详见我们的通信设计文档。

​	确定通信之后的下一件事也相当重要，它是为了解决一种特定的异常情况而出现的。在Syzkaller的测试中包含大量的内存测试，而由于内存地址普遍通过随机生成，因此很有可能会生成违法地址。考虑到效率等各方面的问题，syz-executor的设计者并不希望直接因此导致测试的进程被杀死，而是希望通过类似于try/exception的方式，用自己设计的handler来处理这种情况。C不提供try机制，因此只能通过信号量来完成。幸运的是，uCore+包含信号量的功能，但在具体的异常处理过程中还需要使用longjmp/setjmp函数来完成跨函数的跳转，这一点并没有相应的实现。为求功能最小化，我们暂时删掉了这一功能，但我们已经了解了这两个函数的原理，会在之后将其完成。

​	接下来的步骤基本没有改动，直至开始建立sandbox进程。Syzkaller一共提供了三种不同的sandbox模式：sandbox-none、sandbox-uid、sandbox-namespace模式。这三种模式主要是为了通过隔离权限来防止不应当出现的错误（按照官方文档的说法，可以消除掉找出错误的bug，即false-positive），但是ucore根本没有用户组，也没有namespace隔离机制，所以我们直接采用了sandbox-none，直接生成sandbox进程。

#### Sandbox进程

​	sandbox进程首先也会设定一些环境条件，涉及到cgroup、进程组、会话的操作，uCore版本中全部删去了，只保留了设置进程占用资源大小的操作。在环境准备完成之后，需要在已经建立起的通信pipe上和fuzzer进行握手，向其通知syz-executor已经准备就绪。这点除通信方式外无需做改动。然后就可以开始处理由fuzzer发过来的execute了。

​	sandbox会进入不断的循环，每次建立起新的文件夹和syz-executor进程，然后开始等待。一个syz-executor对应的是一轮execute指令，如果等待的过程超时，它会将这个子进程杀死。无论如何，在syz-executor进程关闭后，再由它负责向fuzzer回复syz-executor执行完毕的信号，然后删除建立的文件夹，进入下一轮循环。循环的过程基本没有修改，但其中的一些部分和细节有着不小的改动（比如atomic操作，exeternal fuzzing等），这个之后会在提到。

#### Execute进程

​	syz-executor进程是一个相当复杂且重要的部分，它的执行步骤主要分为三个步骤：从syz-executor输入中读取数据和指令；分配执行该指令的资源并执行；以及返回结果。第一部分主要涉及到通信和数据的读写，这点只需按照之前的方式修改即可。在执行指令的部分中，linux版本的程序采用了线程池的方法进行管理，一共采用了16个线程来运行不同的系统调用。也正因为这个缘故，它还可以对多线程运行中可能导致的数据竞争进行测试。uCore中虽然有多线程的库，但为了简化程序，我们先删除了这一部分功能。而对于最后的部分，主要的改动涉及到两方面，一个是返回什么样的结果，另一个则和代码覆盖相关，这些之后都会统一说明。

​	以上是对于三个进程在运行步骤上的大致修改情况，但是除此之外还做了很多其它的改动，其中的一些相比上述的改动而言一点不小。这些改动需要单独拿出来进一步说民，一方面是因为贯穿了整个syz-executor执行的过程，另一方面则是它们往往对应的是一整套功能。

​	

### uCore

​	关于uCore的改动主要集中在和系统调用直接相关的代码部分，而这其中修改较多的在用户库的部分。

​	事实上，在修改executor的过程中我们发现uCore的一些用户库函数是存在错误的，有的甚至完全不能运行。除了之前提到的mmap以外，例如waitpid函数等函数还存在着一些问题，其中还存在着一些。

​	另外，很多已经通过系统调用实现了的功能并没有在用户库当中得到实现，却在executor中被使用，这其中包括sertlimit、lstat等函数。我们将其统一进行了更新。也因为这个原因，一些需要使用的数据结构并没有在用户库中被定义，例如linux_lstat、linux_rlimit、sigset_t等，也都进行了更新。

​	对于用户库的主要修改就是这些，而对于kernel而言，针对一点做了较大的修改。可能是由于历史缘故，uCore中的系统调用分为两套：一套是uCore自带的系统调用，另一套则是linux标准下的部分系统调用。两套系统调用并不能通过同一种syscall方式来调用，为了解决这一问题我们将要使用的linux syscall都移到了常规syscall列表中，方便在executor使用。




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



## 6. 实验结果

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



## 7. 后续设想

这一工作还有很多不足，有很多方面可以进行扩展。例如：

+ 添加更多系统调用的测试，修改框架使得其可以执行fast syscall；
+ 添加kcov功能，正确获取分支条件、覆盖率等信息，使得syzkaller可以更有方向地生成测例；
+ 添加更多方向的测试，例如多线程的测试、网络栈的测试、多文件系统的测试等；





## 8. 实验过程日志

见[相关wiki页面](http://os.cs.tsinghua.edu.cn/oscourse/OS2018spring/projects/g07).



## 9. 实验总结

总体来说我们这次实验还有相当的不足，syzkaller原本有着完成从多线程、多虚拟机fuzzing到结果统计、存储、重现的一整套完整流程的能力，在阅读源码、分析实现时也感受到了其功能的全面。但由于ucore实验条件、时间上的各种限制，很多功能我们只能舍弃掉，最后只留下了核心的测试功能。

此外，在实验中遇到了不少之前没能预料到的问题，例如ucore+极少的用户库支持、薄弱的基本功能等等，解决这些问题也消耗了不少时间。另外，在这种运行环境下完全无法使用gdb来调试ucore+的运行情况，使得调试工作比较困难。不过，最后寻找bug、修复问题的过程还是非常有趣的。	



## 10. 参考文献

[1] https://blog.csdn.net/baidu_27386223/article/details/47404871

[2] https://github.com/google/syzkaller/blob/master/docs/internals.md

[3] https://code.google.com/archive/p/u12proj

[4] https://github.com/RUB-SysSec/kAFL
