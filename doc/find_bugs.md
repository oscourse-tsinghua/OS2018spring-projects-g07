1. `sys_mmap` fails when overlapping memory is requested. e.g. 

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

