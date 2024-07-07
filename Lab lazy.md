## Lab lazy 

### 1. Eliminate allocation from sbrk() 

1. *page-fault exception*. When xv6 cannot translate a certain va to pa. The CPU will generate a page-fault exception. There are three kinds of page fault in xv6:

   - load page fault.
   - store page fault.
   - instruction page fault

   when a virtual address cannot be translated into a physical address, `scause` will record the type of the page fault and `stval` will record the address that cannot be translated. Here is the table of scause value:
   
   ![scause](/Users/kyrie/Desktop/s6.081/images/scause.png)

2. Implementation in the `sys_sbrk()` syscall

   ```C
   uint64
   sys_sbrk(void)
   {
     int addr;
     int n;
     struct proc *p = myproc();
     if(argint(0, &n) < 0)
       return -1;
     addr = p->sz;
     p->sz += n;
     //if(growproc(n) < 0)
       return -1;
     return addr;
   }
   ```

   

### 2. Lazy allocation

For the lazy allocation, we can have a figure of its process:

![lazy allocation](/Users/kyrie/Desktop/s6.081/images/lazy allocation.png)

So we should just allocate the page when the process needs to manipulate the memory.

So in `sys_sbrk()`, we just grow the `sz` of the process instead of actually allocating the physical page:

```C
uint64
sys_sbrk(void)
{
  int addr;
  int n;
  struct proc *p = myproc();

  if(argint(0, &n) < 0)
    return -1;
//  addr = myproc()->sz
  addr = p->sz;
  p->sz += n;
//  if(growproc(n) < 0)
//    return -1;
  return addr;
}
```

And in the `trap.c`, when we encounter a page fault related to load/store fault, which means the process is trying to manipulate related memory, then we can allocate the pages for it.

```C
 else if (r_scause() == 13 || r_scause() == 15) {
      uint64 fault_addr = r_stval();
      uint64 fault_page_addr = PGROUNDDOWN(fault_addr);
      char *mem = kalloc();
      //TODO:need to think the case if there is no mem to be allocated (mem == 0)
      memset(mem, 0, PGSIZE);
      if (mappages(p->pagetable, fault_page_addr, PGSIZE, (uint64) mem, PTE_R | PTE_W | PTE_X | PTE_U) != 0) {
          kfree(mem);
      }
  }
```

