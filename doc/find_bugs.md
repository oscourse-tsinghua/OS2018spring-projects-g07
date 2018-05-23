1. `sys_mmap` and `sys_shmem` fails when overlapping memory is requested

e.g. 

```
    a = 0x4000000000;
	ret = sys_mmap(&a, 100, 0);
    cprintf("test 2, a: %lld, ret %d\n", a, ret);
    b = a - 1;
    ret = sys_mmap(&b, 101, 0);
    cprintf("test 2, b: %lld, ret %d\n", b, ret);
```

syscalls above causes kernel panic:

```
kernel panic at vmm.c:273:
    assertion failed: prev->vm_end <= next->vm_start
```


2. `vfs_lookup_parent` fails when path = '/'

in almost all file operations (open, unlink, rename, link, symlink, mkdir), `vfs_lookup_parent` is called. when path = '/', it would fail:

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

while in sfs\_inode.c, `sfs_lookup_parent` assumes that path != '\0' && path != '/'.

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