kAFL原本的工具包括qemu和kvm两部分，我们只使用qemu. 所以把原本的编译脚本和qemu相关的单独提取了出来，这样比较方便。

其中大部分依赖都可以用apt-get解决，不过有一个libcapstone在操作系统课给的虚拟机里(ubuntu 14)无法直接apt-get，需要自己去下载编译。

在[这里](https://github.com/aquynh/capstone/archive/3.0.5-rc2.tar.gz)是源文件，进入目录make, sudo make install即可。

进入kAFL目录下分别执行install-dependencies.sh, apply-patch.sh应该就可以完成qemu-pt编译了。
