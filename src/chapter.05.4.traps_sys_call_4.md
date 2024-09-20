# 5.5 其他trap代码分析

## 1. trampoline.S代码：

```Assembly
        #
        # low-level code to handle traps from user space into
        # the kernel, and returns from kernel to user.
        #
        # 注意这里的描述，kernel.ld 可以加载这段代码都一个page的起点
        # the kernel maps the page holding this code
        # at the same virtual address (TRAMPOLINE)
        # in user and kernel space so that it continues
        # to work when it switches page tables.
        # kernel.ld causes this code to start at 
        # a page boundary.
        #

#include "riscv.h"
#include "memlayout.h"

# 在汇编语言中,.section 指令用于定义一个新的代码段或数据段。在这里，.section trampsec
# 表示定义了一个名为 trampsec 的新段（section），用来标识接下来的代码或数据将被放置在这个段中。
# 这有助于将不同类型的代码或数据组织到不同的段中，提高程序的可读性并且有助于链接器在链接阶段正确地
# 处理这些段。
.section trampsec

# 在这段代码中，trampoline 是一个汇编代码段的起始位置标记，用于实现在用户态（User Mode）
# 和内核态（Kernel Mode）之间进行切换时的一些操作。具体来说，trampoline 这段代码实现了
# 两个全局符号 uservec 和 userret，分别用于用户态到内核态的切换和内核态返回用户态时的操作。

# 问题？这个trampoline 是会出现在一个固定的物理内存地址上吗？如何映射的？
# 是先由文件中一步步映射的吗？
.globl trampoline
trampoline:


.align 4

.globl uservec
uservec:
#  ...


.globl userret
userret:
#  ...
```



上面的代码中 a0 表示了一个通用寄存器

在 uservec中， 基本操作就是，使用sscratch来保存a0的内容，然后使用a0进行一些赋值操作

最后将 sscratch 也保存到 p->trapframe->a0 中。如：csrw

在 userret，a0 表示了函数的第一个参数，也就是usertrapret函数 的变量 satp

然后使用a0进行一个赋值操作，最后再将 p->trapframe->a0 的值，赋值到a0中，返回到用户空间

总结就是，使用a0 做了一些腾挪操作



## 2. devintr 函数

```C
// check if it's an external interrupt or software interrupt,
// and handle it.
// returns 2 if timer interrupt,
// 1 if other device,
// 0 if not recognized.
int
devintr()
{
  uint64 scause = r_scause();

  if((scause & 0x8000000000000000L) &&
     (scause & 0xff) == 9){
    // this is a supervisor external interrupt, via PLIC.

    // irq indicates which device interrupted.
    int irq = plic_claim();

    if(irq == UART0_IRQ){
      uartintr();
    } else if(irq == VIRTIO0_IRQ){
      virtio_disk_intr();
    } else if(irq){
      printf("unexpected interrupt irq=%d\n", irq);
    }
    
    // 当cpu的核数大于1，那么可能出现 irq 为0的情况
    // 也就是可能触发一个空的设备中断

    // the PLIC allows each device to raise at most one
    // interrupt at a time; tell the PLIC the device is
    // now allowed to interrupt again.
    if(irq)
      plic_complete(irq);

    return 1;
  } else if(scause == 0x8000000000000001L){
    // software interrupt from a machine-mode timer interrupt,
    // forwarded by timervec in kernelvec.S.

    if(cpuid() == 0){
      clockintr();
    }
    
    // acknowledge the software interrupt by clearing
    // the SSIP bit in sip.
    w_sip(r_sip() & ~2);

    return 2;
  } else {
    return 0;
  }
}
```



经过打印计数分析，在正常启动过程中，devintr主要是被  `kerneltrap` 函数来调用的，只有在 用户模式下的进程，比如加载shell, 或shell的一些命令操作，才可能触发 `usertrap` 函数的调用