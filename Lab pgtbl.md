## 1. Speed up `getpid()` syscall

### Why storing the `pid` in user page table could speed up the `getpid()`？

First let us look at the `struct proc` in `proc.h`. This is the definition of the `PCB`.(Process Control Block):

```C
struct proc {
  struct spinlock lock;

  // p->lock must be held when using these:
  enum procstate state;        // Process state
  void *chan;                  // If non-zero, sleeping on chan
  int killed;                  // If non-zero, have been killed
  int xstate;                  // Exit status to be returned to parent's wait
  int pid;                     // Process ID

  // wait_lock must be held when using this:
  struct proc *parent;         // Parent process

  // these are private to the process, so p->lock need not be held.
  uint64 kstack;               // Virtual address of kernel stack
  uint64 sz;                   // Size of process memory (bytes)
  pagetable_t pagetable;       // User page table
  struct trapframe *trapframe; // data page for trampoline.S
  struct context context;      // swtch() here to run process
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)
};
```

As we all know, the PCB is part of the kernel code. So after we call the `getpid()` syscall, we will trap into kernel and we use kernel page table to map the virtual address of `pid` into the physical address, then we copy in/out from kernel to user space. So if we allocate a  read-only page table between user and kernel space then we can eliminate the copy in/out process so we can speed up the system call.

## 2. Print page table information

About the judgement statement in `freewalk()` function to check if this `pte` is a pointer to next level pte:

```C
void
freewalk(pagetable_t pagetable)
{
  // there are 2^9 = 512 PTEs in a page table.
  for(int i = 0; i < 512; i++){
    pte_t pte = pagetable[i];
    if((pte & PTE_V) && (pte & (PTE_R|PTE_W|PTE_X)) == 0){
      // this PTE points to a lower-level page table.
      uint64 child = PTE2PA(pte);
      freewalk((pagetable_t)child);
      pagetable[i] = 0;
    } else if(pte & PTE_V){
      panic("freewalk: leaf");
    }
  }
  kfree((void*)pagetable);
}

```

The `pte & (PTE_R|PTE_W|PTE_X)) == 0` tells if this pte is a next level point. We can refer the RISC-V manual:

|  X   |  W   |  R   | Meaning                              |
| :--: | :--: | :--: | :----------------------------------- |
|  0   |  0   |  0   | Pointer to next level of page table. |
|  0   |  0   |  1   | Read-only page.                      |
|  0   |  1   |  0   | *Reserved for future use.*           |
|  0   |  1   |  1   | Read-write page.                     |
|  1   |  0   |  0   | Execute-only page.                   |
|  1   |  0   |  1   | Read-execute page.                   |
|  1   |  1   |  0   | *Reserved for future use.*           |
|  1   |  1   |  1   | Read-write-execute page.             |

The solution would be:

```c
void walk_and_print(pagetable_t pagetable, int level){
  for(int i = 0; i < 512; i++){
    pte_t pte = pagetable[i];
    if((pte & PTE_V) && (pte & (PTE_R|PTE_W|PTE_X)) == 0){
      //which means this pte has low level child
      uint64 child = PTE2PA(pte);
      for(int i = 0; i < level + 1; i++){
        printf(" ..");
      }
      printf("%d: pte %p pa %p\n", i, pte, child);
      walk_and_print((pagetable_t)child, level + 1);
    } else if(pte & PTE_V){
      //leaf node
      uint64 child = PTE2PA(pte);
      for(int i = 0; i < level + 1; i++){
        printf(" ..");
      }
      printf("%d: pte %p pa %p\n", i, pte, child);
    }
  }
  return;
}

void vmprint(pagetable_t pagetable){
  walk_and_print(pagetable, 0);
  return;
}
```



## 3. Detecting which pages have been accessed 

1. The flags in `PTE` and their meanings

![img](https://five-embeddev.com/riscv-priv-isa-manual/Priv-v1.12/supervisor_19.svg)

2. For every va, the address translation process always travels all three level page table.
3. One PTE conrresponds to one physical page in memory.

The solution would be:

```c
int
sys_pgaccess(void)
{
  // lab pgtbl: your code here.
  uint64 va;
  int page_limit;
  uint64 abitmask;
  if(argaddr(0, &va) < 0){
    return -1;
  }
  if(argint(1, &page_limit) < 0){
    return -1;
  }
  if(argaddr(2, &abitmask) < 0){
    return -1;
  }
  if (page_limit > 64){
    return -1;
  }
  //get the pte address of the start check address
  pagetable_t pagetable = myproc()->pagetable;
  uint64 bitmask = 0;
  pte_t *pte = walk(pagetable, va, 0);
  for(int i = 0; i < page_limit; i++){
    if (*pte & PTE_A){
      *pte = (*pte) & (~PTE_A); //clear the A flag
      bitmask |= (1L << i);
    }
    pte += 1;// next page
  }
  if(copyout(pagetable, abitmask, (char*)&bitmask, sizeof(bitmask)) < 0){
    return -1;
  }
  return 0;
}
```

## User page and kernel page

1. User page mapping

   ![userpage](/Users/kyrie/Desktop/s6.081/images/userpage.jpg)

2. Kernel page mapping

   ![kernelpage](/Users/kyrie/Desktop/s6.081/images/kernelpage.png)

The target architecture is assumed to be riscv:rv64
.gdbinit:3: Error in sourced command file:              
Remote communication error.  Target disconnected.: connection reset by peer.
