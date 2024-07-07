## Lab cow

Copy-on-write fork is a very common optimaztion of fork in opterating systems. Generally, when you are using `fork()` to fork a child process, operating system will allocate new pages for the child process and copy all the information of the parent process(registers statuses, pagetable and text and etc.). But when the information needed to copy is too large or the child process will execute the `exec()` after the fork, it will waste lots of memory and time. Instead, copy-on-write fork will only copy the  memory when it is requested to write to the memory by using the page fault mechanism in xv6. The following is the implementation of this mechanism.

First we need to modify the `uvmcopy()` to enable the fork to map the pages of the child process to the same physical address with the parent process:

```C
// Given a parent process's page table, copy
// its memory into a child's page table.
// Copies both the page table and the
// physical memory.
// returns 0 on success, -1 on failure.
// frees any allocated pages on failure.
int
uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
  pte_t *pte;
  uint64 pa, i;
  uint flags;
//  char *mem;
  /*
   * 1. clear the valid bit
   * 2. map the child page table to the same address as the parent
   * 3. count the reference number of the address
   */
  for(i = 0; i < sz; i += PGSIZE) {
    if ((pte = walk(old, i, 0)) == 0)
      panic("uvmcopy: pte should exist");
    if ((*pte & PTE_V) == 0)
      panic("uvmcopy: page not present");
    *pte &= ~PTE_W; // clear the valid bit in the pagetable so that it can occur the page fault
    *pte |= PTE_COW; //mark it as cow page
    pa = PTE2PA(*pte);
    flags = PTE_FLAGS(*pte);
//    if((mem = kalloc()) == 0)
//      goto err;
//    memmove(mem, (char*)pa, PGSIZE);
    if (mappages(new, i, PGSIZE, pa, flags) != 0) {
//      kfree(mem);
      goto err;
    }
    kaddrefcnt((void*)pa);
  }
  return 0;
 err:
  uvmunmap(new, 0, i / PGSIZE, 1);
  return -1;
}
```

Then we need to record the reference count of the same physical address by using a count array in `kalloc.c`:

```C
struct page_ref {
 struct spinlock lock;
 int page_cnt[PHYSTOP / PGSIZE];
} page_ref;
```

Then we need to modify the `kalloc()` and the `kfree()` with the reference count

```C
// Free the page of physical memory pointed at by v,
// which normally should have been returned by a
// call to kalloc().  (The exception is when
// initializing the allocator; see kinit above.)
void
kfree(void *pa)
{
  struct run *r;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  acquire(&page_ref.lock);
  if (--page_ref.page_cnt[(uint64)pa / PGSIZE] == 0) {
    release(&page_ref.lock);
    // Fill with junk to catch dangling refs.
    memset(pa, 1, PGSIZE);

    r = (struct run*)pa;

    acquire(&kmem.lock);
    r->next = kmem.freelist;
    kmem.freelist = r;
    release(&kmem.lock);
  } else {
    release(&page_ref.lock);
  }
}

// Allocate one 4096-byte page of physical memory.
// Returns a pointer that the kernel can use.
// Returns 0 if the memory cannot be allocated.
void *
kalloc(void)
{
  struct run *r;

  acquire(&kmem.lock);
  r = kmem.freelist;
  if(r) {
      kmem.freelist = r->next;
      acquire(&page_ref.lock);
      page_ref.page_cnt[(uint64)r / 4096] = 1;
      release(&page_ref.lock);
  }
  release(&kmem.lock);

  if(r)
    memset((char*)r, 5, PGSIZE); // fill with junk
  return (void*)r;
}



int
kaddrefcnt(void* pa){
    if ((uint64) pa % PGSIZE != 0 || (char*) pa < end || (uint64) pa > PHYSTOP) {
        return -1;
    }
    acquire(&page_ref.lock);
    page_ref.page_cnt[(uint64)pa / PGSIZE]++;
    release(&page_ref.lock);
    return 0;
}

int
krefcnt(uint64 pa){
  if (pa % PGSIZE != 0) {
    return -1;
  }
  return page_ref.page_cnt[pa / PGSIZE];
}
```

We should add some functions that will be used in the allocation of the page fault process in `vm.c`

```C
/*
 * check if a va in a cow page
 * return 0 is a cow page
 */
int
is_cow_page(pagetable_t pagetable, uint64 va){
  if (va >= MAXVA) {
    return -1;
  }
  pte_t* pte = walk(pagetable, va, 0);
  if (*pte == 0) {
    return -1;
  }
  if ((*pte & PTE_V) == 0) {
    return -1;
  }
  return (*pte & PTE_COW) ? 0 : -1;
}

void*
cow_alloc(pagetable_t pagetable, uint64 round_addr){
  if (round_addr % PGSIZE != 0){
    return 0;
  }

  uint64 pa = walkaddr(pagetable, round_addr);
  pte_t *pte = walk(pagetable, round_addr, 0);
  if (krefcnt(pa) == -1) {
    return 0;
  }
  if (krefcnt(pa) == 1) { //just mapped once
    *pte |= PTE_W;
    *pte &= ~PTE_COW;
    return (void*)pa;
  } else {
    char *mem = kalloc();
    if (mem == 0) {
      return 0;
    }
    memmove(mem, (char*)pa, PGSIZE);
    *pte &= ~PTE_V;
    if (mappages(pagetable, round_addr, PGSIZE, (uint64)mem, (PTE_FLAGS(*pte) | PTE_W) & ~PTE_COW) != 0){
      kfree(mem);
      *pte |= PTE_V;
      return 0;
    }

    kfree((char*)(PGROUNDDOWN(pa)));
    return mem;
  }
}
```

And then in the `trap.c`, we implement the logic when we encounter a page fault from a cow page:

```C
//
// handle an interrupt, exception, or system call from user space.
// called from trampoline.S
//
void
usertrap(void)
{
  int which_dev = 0;

  if((r_sstatus() & SSTATUS_SPP) != 0)
    panic("usertrap: not from user mode");

  // send interrupts and exceptions to kerneltrap(),
  // since we're now in the kernel.
  w_stvec((uint64)kernelvec);

  struct proc *p = myproc();
  pagetable_t pagetable = p->pagetable;
  // save user program counter.
  p->trapframe->epc = r_sepc();
  
  if(r_scause() == 8){
    // system call

    if(p->killed)
      exit(-1);

    // sepc points to the ecall instruction,
    // but we want to return to the next instruction.
    p->trapframe->epc += 4;

    // an interrupt will change sstatus &c registers,
    // so don't enable until done with those registers.
    intr_on();

    syscall();
  } else if((which_dev = devintr()) != 0){
    // ok
  } else if (r_scause() == 13 || r_scause() == 15) {
    uint64 fault_addr = r_stval();
    if (is_cow_page(pagetable, fault_addr) != 0){
      printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
      printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
      p->killed = 1;
    }
    if (fault_addr > p->sz || cow_alloc(pagetable, PGROUNDDOWN(fault_addr)) == 0) {
      p->killed = 1;
    }
  } else {
    printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
    printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
    p->killed = 1;
  }

  if(p->killed)
    exit(-1);

  // give up the CPU if this is a timer interrupt.
  if(which_dev == 2)
    yield();

  usertrapret();
}
```

