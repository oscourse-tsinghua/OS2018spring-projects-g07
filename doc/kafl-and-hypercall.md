## kAFL结构简介

### 总体架构

kAFL是使用qemu-kvm完成硬件反馈的工具。虽然实际上需要使用Intel PT, 但是我们使用的syzkaller框架并不使用这个功能，所以这个功能被我们删去。

为了实现内存共享，在host OS - host linux kernel - qemu - guest OS四处都要添加支持。

### hypercall及硬件虚拟化介绍

在说明具体修改之前，先阐述一下hypercall的原理。

hypercall是linux kernel中kvm模块所提供的调用方法，依靠于Intel x86硬件提供的vmx硬件虚拟化功能。在硬件虚拟化执行过程中，指令实际上是在host上直接运行。

在运行过程中，可以使用vmcall这一指令触发VM Exit，从虚拟机中离开。此时，linux内核将捕获这一操作，然后回到虚拟机中，即触发VM Entry。而VM Entry进入点在qemu中，因此qemu可以获取hypercall的内容做出反应。这就是hypercall执行的全过程。

接下来说明为了实现kAFL，各处所需要的支持:

### Guest OS

OS本身不需要任何支持。不过，需要用户库中提供调用hypercall的接口。

### Linux kernel

需要在VM Exit事件捕获时，根据vmcall时的rax, rbx, rcx寄存器设置VMCS数据。

具体来说，就是实现和内存分享相关的几个调用指令的处理。

### Qemu

需要在VM Exit返回时，根据调用类型进行相关处理。

包括：设置共享内存；对虚拟页进行读写，完成host/guest间数据传输。

另外，还添加了一个新的设备(kafl)，可以让host在调用时把参数传入。

### Host OS

host需要传入qemu需要的参数，kAFL具体做的是：在host中使用mmap将文件映射到内存中，然后将文件名传给qemu.

## 具体接口

见ucore/libs-user-ucore/hypercall.h

大部分接口都是无需使用的。需要用的只有：

```
HYPERCALL_KAFL_GET_PAYLOAD
HYPERCALL_KAFL_NEXT_PAYLOAD
HYPERCALL_KAFL_INFO
```

PAYLOAD是fuzzer输入executor的程序段，INFO是executor输出到fuzzer的反馈段。

在虚拟机中调用GET/NEXT PAYLOAD可以获取程序段，调用INFO可以传出覆盖信息。都需要一个参数置于rcx中，这个参数是内存起始段的地址。