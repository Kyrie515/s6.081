## GDB使用说明

### 1. 为qemu准备GDB环境

首先我们需要在gdb模式下编译qemu：

```c
make qemu-gdb
```

编译完成之后qemu会卡在shell之前的一个部分，接下来，用tmux的split窗口打开一个新终端

```c
tmux a -t kyli //创建新的tmux会话

ctrl B + % //这个命令产生一条垂直线分裂窗口出一个新终端
```

在新终端输入

```c
gdb-multiarch
```

进入到调试环境，此时可以看到原来qemu窗口进入shell。若此时我们想调试`user/ls.c`下面的文件，我们可以在gdb窗口输入

```
file user/_ls //添加用户ls.c文件
b main
```

### 2. 常用GDB命令

```python
b # 打断点 (e.g.     b main | b *0x30)
c # continue

layout split # view src-code & asm-code

ni # 单步执行汇编(不进函数)
si # 单步执行汇编(有函数则进入函数)
n # 单步执行源码
s # 单步执行源码

p # print
p $a0 # 打印a0寄存器的值
p/x 1536 # 以16进制的格式打印1536
i r a0 # info registers a0
x/i 0x630 # 查看0x630地址处的指令
x/g 0x80000000 # 查看0x80000000地址处的值（g表示值的长度有64位）
```

### 3. 如何从在Debug的时候观察到从用户态向内核态的转变

首先我们要知道用户陷入内核有三种方式: `systemcall`, `exception` and `device interrupt`.每个从用户态陷入内核的过程都会调用`ecall`指令，这个指令会将pc跳转到执行`trampoline.S`汇编代码里的`uservec`函数。这个函数的后面几行会进入到内核态。

```assembly
csrw satp, t1
sfence.vma zero, zero

# a0 is no longer valid, since the kernel page
# table does not specially map p->tf.

# jump to usertrap(), which does not return
jr t0
```

根据这个过程，我们可以在用户调用`syscall`的用户接口的时候打一个断点。然后由于操作系统是通过找到`stvec`寄存器的值找到`trampoline.S`的入口，所以我们再进入到`syscall`的函数之后需要找到`stvec`寄存器的地址然后再在这个地址打断点就可以成功进入到`trampoline.S`的代码里了。例如在lab trap中：

```C
(gdb) file user/_bttest
(gdb) b main #在bttest的main函数打断点
(gdb) c
(gdb) i r stvec
 #1 = 0x3ffffff000
(gdb) b *0x3ffffff000
(gdb) c
```

之后进入到`trampoline.S`之后，在`jr t0`处`si`就会进入到内核态，此时因为没有加在内核文件所以看不到源码，只需要`file kernel/kernel`就可以了。
