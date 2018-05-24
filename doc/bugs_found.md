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


3. `sys_dup` fails when fd2 is invalid

in fs/file.c, `file_dup` is implemented as following: 

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

This does not check if fd2 is valid, while in `file_desc_table_dissociate` fd2 is assumed to be a valid file descriptor.

```
int file_desc_table_dissociate(struct file_desc_table *table, int file_desc)
{
  assert(file_desc >= 0 && file_desc < table->capacity);
  struct file *file = table->entries[file_desc].opened_file;
  assert(file != NULL);
  fopen_count_dec(file);
  table->entries[file_desc].opened_file = NULL;
  return 0;
}
```

4. `sys_linux_sigaction` fails for invalid signal number.

The code below handles `sys_linux_sigaction` syscall:

```
int do_sigaction(int sign, const struct sigaction *act, struct sigaction *old)
{
	assert(get_si(current)->sighand);
#ifdef __SIGDEBUG
	kprintf("do_sigaction(): sign = %d, pid = %d\n", sign, current->pid);
#endif
	struct sigaction *k = &(get_si(current)->sighand->action[sign - 1]);

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

  ....
}
```

Obviously `sign` is unchecked, which results in possible data corruption. e.g.

```
    char corrupt_data[20];
    int i = 0;
    for (; i < 20; i++) {
        corrupt_data[i] = 0xff;
    }
    sigaction(0x41, (struct sigaction *)(&corrupt_data[0]), 0);
```


5. `sys_linux_mmap` fails when fd is invalid.

the value of fd is unchecked. If an invalid argument is sent in, `vma_mapfile` fails. e.g.


```
sys_linux_mmap(0x7, 0x3, 0x2, 0xf90586166d82955a, 0xffffffffffffffff, 0x48) ;
```