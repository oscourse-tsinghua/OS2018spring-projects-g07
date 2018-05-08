Install KVM-PT & QEMU-PT with following steps:

cd kAFL
bash ./install-dependencies.sh
bash ./apply-patch.sh
bash ./download-linux.sh
bash ./apply-batch-linux.sh
reboot

if you are using Linux kernel (version != 4.6.2), then GRUB\_DEFAULT should be set to Linux 4.6.2.

ucore_plus/ucore/kvm_uCore_run.sh is an example about how QEMU-PT is used.