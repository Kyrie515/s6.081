## Lab traps

1. The xv6 registers graph

   <img src="/Users/kyrie/Desktop/s6.081/images/image.png" alt="image" style="zoom:100%;" />

2. The function of `PGROUNDUP` and `PGROUNDDOWN` in the file `riscv.h`:

   ```c
   #define PGSIZE 4096 // bytes per page
   #define PGSHIFT 12  // bits of offset within a page
   
   #define PGROUNDUP(sz)  (((sz)+PGSIZE-1) & ~(PGSIZE-1))
   #define PGROUNDDOWN(a) (((a)) & ~(PGSIZE-1))
   ```

   - `PGROUNDUP`: This function returns the nearest upper boundary of of a page where `sz` address is
   - `PGROUNDDOWN`: This function return the nearest lower boundary of a page where `a` address is

3. The knowledge of the registers

   - `stvec`: `stvec(supervisor trap vector)` is used to define the base address of the trap handler. When a trap occurs, the RISC-V processor uses the value in the `stvec` register to determine where to transfer control in order to handle the trap.
   - `sepc`: `sepc(supervisor exception program counter)` is used to store the address of the instruction that caused the trap. This register is crucial for returning control to the appropriate point in the program after the trap has been handled.

4. The difference between `ret`, `sret` and `ecall` instructions

   - `ret` instruction will load the `ra` (return address) register to the `pc` .
   - `sret` instruction will load `sepc`register to the `pc`. And it may change the mode.
   - `ecall` instruction will load the s`stvec` register to the `pc`. And it may change the mode.

### Backtrace

For this task, all you need to know is just the return address value is the pc saved by previous caller. And you should also know the function calls stack data structure like:

<img src="/Users/kyrie/Library/Application Support/typora-user-images/截屏2024-05-17 15.31.23.png" alt="截屏2024-05-17 15.31.23" style="zoom:80%;" />

so in `kernel/printf.c`:

```C
/*
 * backtrace the functions stack
*/
void
backtrace(void)
{
  printf("backtrace:\n");
  uint64 fp = r_fp();
  uint64 stack_bottom = PGROUNDUP(fp);
  //uint64 stack_upper = PGROUNDDOWN(fp);
  while (fp < stack_bottom){
    //first get the return address of current function
    uint64 ra = fp - 8;
    //get the previous fp
    uint64 previous_fp = fp - 16;
    //then we can get the address of caller by read the value from the ra
    printf("%p\n", *((uint64*)ra));
    fp = *((uint64*)previous_fp);
  }
  return;
}
```

2. The figure of processes involved in traps:![截屏2024-05-18 10.51.02](/Users/kyrie/Library/Application Support/typora-user-images/截屏2024-05-18 10.51.02.png)
