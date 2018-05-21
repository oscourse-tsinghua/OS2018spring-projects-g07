Install KVM-PT & QEMU-PT with following steps:

cd kAFL

bash ./install-dependencies.sh

bash ./apply-patch.sh

bash ./download-linux.sh

bash ./apply-patch-linux.sh

reboot

if you are using Linux kernel (version != 4.6.2), then GRUB\_DEFAULT should be set to Linux 4.6.2.

ucore_plus/ucore/kvm_uCore_run.sh is an example about how QEMU-PT is used.

after compiling, add kAFL/qemu-2.9.0/x86_64-softmmu/ to PATH, and rename kAFL/qemu-2.9.0/x86_64-softmmu/qemu-system-x86_64 to kAFL/qemu-2.9.0/x86_64-softmmu/qemu-system-x86_64-kafl.
