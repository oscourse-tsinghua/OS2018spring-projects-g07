# syzkaller系统测试工具在ucore+上的移植-课程设计方案报告

计52 王纪霆 匡正非

## 项目概述

我们的目标是将[syzkaller](https://github.com/google/syzkaller)移植到ucore+上，并以此为工具对ucore+的系统调用进行调试，发现其中的bug。然而经过调研，这一目标过于庞大，不可能在一学期间完成，因此我们将项目拆分成较小的部分，目标是尽量实现syzkaller所需要的软件环境。

具体而言，各级目标包括:

+ go1.8+在ucore+上的基本移植；
+ 实现syzkaller所需的golang package；
+ 实现syzkaller所需的其他操作系统功能（外部工具、文件系统）
+ 完成最终调试

## 已有工作调研

### golang移植的已有工作

golang目前已有向ucore的移植，但其

### syzkaller的已有工作



## 工具分析

## 设计方案

## 小组分工

## 目前工作

