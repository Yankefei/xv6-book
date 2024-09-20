# 5.6: 常用寄存器整理：

## 1. 专用寄存器：

### **stvec**

保存trap handler address

### **sepc**

sepc 将是一个后面后面trap结束后，任务开始的地方，注意，这里的pc(program counter)可能不会自动保存 到sepc中，需要手动将pc存储在sepc中。

### **scause**

描述 trap 的原因. 

在RISC-V架构中，`scause`寄存器记录了最后一次异常或中断的原因。这个寄存器的值可以帮助异常处理程序（trap handler）确定发生了什么类型的事件，从而采取相应的处理措施。`scause`寄存器的值包含两部分：最高位（MSB）表示异常类型（1表示中断，0表示异常），其余位表示异常或中断的具体原因。

下面是一些可能被赋值到`scause`寄存器的状态：

异常（Exception）

异常是由于程序执行中的错误或特定指令触发的同步事件。对于异常，`scause`寄存器的MSB为0。

![](./images/trap_2.png)



- Instruction Address Misaligned (0x0)：指令地址未对齐
- Instruction Access Fault (0x1)：指令访问错误
- Illegal Instruction (0x2)：非法指令
- Breakpoint (0x3)：断点（由`ebreak`指令触发）
- Load Address Misaligned (0x4)：加载地址未对齐
- Load Access Fault (0x5)：加载访问错误
- Store/AMO Address Misaligned (0x6)：存储/原子内存操作地址未对齐
- Store/AMO Access Fault (0x7)：存储/原子内存操作访问错误
- Environment Call from **U-mode (0x8)** / **S-mode (0x9)** / M-mode (0xb)：来自U模式/S模式/M模式的环境调用（由`ecall`指令触发）
- Instruction Page Fault (0xc)：指令页错误, 12
- **Load Page Fault (0xd)**：加载页错误, 13

常见于内存页的权限缺失，比如只有U，没有W,R 等权限

- Store/AMO Page Fault (0xf)：存储/原子内存操作页错误, 15

常见于内存页的无法写入

详细描述：

**Load Page Fault (0xD)**：当处理器尝试加载一个虚拟地址的内容时，如果页表条目不存在或不允许读取，会触发此异常。

**Store/AMO Page Fault (0xF)**：当处理器尝试存储一个虚拟地址的内容或进行AMO操作时，如果页表条目不存在或不允许写入，会触发此异常。



中断（Interrupt）

中断是由外部事件触发的异步事件。对于中断，`scause`寄存器的MSB为1。（下面是32位的信息，64位的话，也是最高位设置为1）

- User Software Interrupt (0x80000000)：用户软件中断
- Supervisor Software Interrupt (0x80000001)：监督者软件中断
- Machine Software Interrupt (0x80000003)：机器软件中断
- User Timer Interrupt (0x80000004)：用户定时器中断
- Supervisor Timer Interrupt (0x80000005)：监督者定时器中断
- Machine Timer Interrupt (0x80000007)：机器定时器中断
- User External Interrupt (0x80000008)：用户外部中断
- Supervisor External Interrupt (0x80000009)：监督者外部中断
- Machine External Interrupt (0x8000000b)：机器外部中断

以上列出的是一些标准的异常和中断原因代码，具体可能会根据RISC-V的不同扩展和特定的硬件实现有所变化。了解这些`scause`寄存器的值对于开发操作系统和异常处理程序至关重要。

### **sscratch**

一个寄存器的通用缓存，避免在保存用户寄存器之前覆盖它们,



### **sstatus**

表示supervisor status register 的状态

一个表示状态的寄存器，主要用下面两个bit位

SIE: 控制是否 device interrupt 打开 

SPP: 控制trap来自user mode 还是 supevisor mode, 以及用什么模式从 sret返回

请注意：这个字段只是说明了trap来自哪个模式，但没有明确表示当前位于哪个模式，具体看当前处于U还是S， 需要看看当前执行时，处于哪个地址空间

```C
// Supervisor Status Register, sstatus

#define SSTATUS_SPP (1L << 8)  // Previous mode, 1=Supervisor, 0=User  256 

// 这个0x20的位感觉在qemu中，会莫名其妙地被赋值，暂时忽略，只关注下面 （SSTATUS_SIE）0x2 的值
#define SSTATUS_SPIE (1L << 5) // Supervisor Previous Interrupt Enable 32  
#define SSTATUS_UPIE (1L << 4) // User Previous Interrupt Enable       16

#define SSTATUS_SIE (1L << 1)  // Supervisor Interrupt Enable          2
#define SSTATUS_UIE (1L << 0)  // User Interrupt Enable                1


// 进入 user space 的时候，首先会将SPP设置为0，并且将 SPIE打开，表示开启 supervisor 的中断

// 下面的三个函数，也会对 ssstatus的状态进行修改和调整

// enable device interrupts
// 打开SIE, 
static inline void
intr_on()
{
  w_sstatus(r_sstatus() | SSTATUS_SIE);
}

// disable device interrupts
static inline void
intr_off()
{
  w_sstatus(r_sstatus() & ~SSTATUS_SIE);
}

// are device interrupts enabled?
static inline int
intr_get()
{
  uint64 x = r_sstatus();
  return (x & SSTATUS_SIE) != 0;
}
```



#### SPP

`SPP`位的功能：

- `SPP`位是`sstatus`寄存器的一部分，用于记录进入S模式异常处理程序之前CPU所处的权限模式。`SPP`位只有一位，具有两种可能的状态：
  - `0`：表示异常或中断发生前CPU处于U模式。
  - `1`：表示异常或中断发生前CPU处于S模式。

`SSTATUS_SPP`赋值时机：

当从用户模式（U模式）或监督者模式（S模式）触发异常或中断，并且系统进入监督者模式下的异常处理程序时，

**硬件会自动设置** **`SSTATUS_SPP`** 位!!!!!!!!!!!!

这个过程主要发生在两种情况下：

1. **从U模式进入S模式的异常处理程序：** 如果异常或中断发生时CPU处于U模式，`SSTATUS_SPP`将被硬件设置为`0`，表示异常处理程序完成后应返回到U模式。
2. **从S模式进入S模式的异常处理程序：** 如果异常或中断发生时CPU已经处于S模式，`SSTATUS_SPP`将被硬件设置为`1`，表示异常处理程序完成后应返回到S模式。

`SSTATUS_SPP`清空时机：

当异常处理程序执行完毕，准备通过`sret`指令返回到之前的模式（U模式或S模式）时，`sret`指令的执行会根据`SSTATUS_SPP`位的值恢复CPU的权限模式，并清空`SSTATUS_SPP`位。具体行为如下：

- 如果`SSTATUS_SPP`为`0`，CPU返回到U模式，`SSTATUS_SPP`随后被清空。
- 如果`SSTATUS_SPP`为`1`，CPU返回到S模式，`SSTATUS_SPP`随后被清空。

为下一次异常或中断处理准备。

这个机制确保了异常或中断处理的正确返回行为，允许操作系统安全地管理权限模式的切换，并保证在处理完异常或中断后能够恢复到正确的执行上下文。

总结来说，`SSTATUS_SPP`位的赋值和清空由硬件在进入和退出异常处理时自动管理，这是RISC-V体系结构中处理异常和中断的重要机制之一。



#### SIE

SSTATUS_SIE（Supervisor Interrupt Enable）        1L << 1

- 功能：`SSTATUS_SIE`位用于控制在S模式下中断是否被允许。如果此位设置为1，则表示在S模式下中断可以被接受和处理；如果设置为0，则中断被禁止，即使中断线被激活也不会触发中断处理程序。
- 用途：在正常操作中控制S模式的中断接受状态。例如，操作系统可能在执行关键代码时暂时禁用中断，以避免中断处理程序中断关键操作。

SSTATUS_SPIE（Supervisor Previous Interrupt Enable）    1L << 5

- 功能：`SSTATUS_SPIE`位用来保存进入中断或异常处理前`SSTATUS_SIE`的状态。也就是说，当异常或中断发生，导致处理器进入异常处理模式时，当前的`SSTATUS_SIE`值会被自动保存到`SSTATUS_SPIE`。
- 用途：当执行`SRET`指令从异常或中断处理程序返回到常规执行流程时，`SSTATUS_SPIE`的值会被**自动恢复**到`SSTATUS_SIE`，这样可以恢复到进入异常或中断处理前的中断使能状态。这是为了保证系统的稳定性和中断处理的正确性，确保在处理完中断或异常后，系统能够回到原先的中断使能状态。

关键区别

- 使用时机：`SSTATUS_SIE`是当前的中断使能位，直接影响当前S模式下是否接受中断；而`SSTATUS_SPIE`是用于保存进入异常或中断处理前`SSTATUS_SIE`的值，用于在之后恢复状态。
- 行为：**在发生异常或中断时，****`SSTATUS_SIE`****会被清零（禁用中断）**，防止新的中断打断异常处理程序，而`SSTATUS_SIE`的原始值则会被存储到`SSTATUS_SPIE`。当从异常或中断返回时，`SSTATUS_SIE`会被设置为`SSTATUS_SPIE`的值，恢复原来的中断使能状态。



### **mstatus**

表示machine status register 的状态

```C
// Machine Status Register, mstatus

#define MSTATUS_MPP_MASK (3L << 11) // previous mode.      6144
#define MSTATUS_MPP_M (3L << 11)                        // 6144
// 也就是在从start开始， mstatus 将设置为下面的这个值
#define MSTATUS_MPP_S (1L << 11)                        // 2048
#define MSTATUS_MPP_U (0L << 11)                        // 0
// 同时也将这个值打开，表示将触发 machine-mode 下的interrupt的开关
#define MSTATUS_MIE (1L << 3)    // machine-mode interrupt enable. // 8
```

### ra 

在RISC-V架构中，`ra`寄存器（返回地址寄存器）主要用于存储函数调用时的返回地址。这允许程序在函数执行完成后返回到调用点继续执行。`ra`寄存器的使用遵循RISC-V的调用约定

也就是执行完当前函数后，退出，应该去的地方

### pc：

pc寄存器是表示当前进程执行的代码段的位置

### stval :

在RISC-V架构中，`stval`寄存器是用于存储发生访存异常时的导致异常的存储地址。当发生存储异常（如访问未映射的地址）时，处理器将异常地址存储在`stval`寄存器中，以便于异常处理程序可以了解到导致异常的具体存储地址。

主要用于 **copy-on-write fork** 缺页中断时的功能



## 2. 通用寄存器：

> The kernel trap code saves user registers to the current process’s trap frame, where kernel code can find them.

- a0

系统调用的返回值，目前在用户空间中，会返回给 trapframe的a0，最终会设置给a0寄存器

- a1

- a2

- a3

- a4

- a5

上面的a0 到 a5 均是用于系统调用传参所使用，可以传值，也可以传地址

- a7

常用于系统调用时，传入系统调用编号





## 3. 常用汇编指令的区别

- la   

la a0, init              // 将 init的**地址**加载到a0

- sd  a3,   16(a0)   // 将 a3 保存到  a0 + 16的**地址**中

- ld   a1   24(a0)   //   # 将地址寄存器 a0 中的值加上 24，然后将得到的**地址处的内容**加载到a1中

- li

li a7, SYS_exec     // 将 SYS_exec的**值**加载到a7

- csrr

csrr a1, mhartid

是 RISC-V 汇编语言中的一条特权指令，用于将当前硬件线程的 Hart ID（硬件线程标识符）加载到通用寄存器 a1 中。

`csrr`: 这是 RISC-V 汇编指令中的一个缩写，表示 "CSR read"，用于从控制状态寄存器（CSR，Control and Status Register）中读取内容。

- ret

在RISC-V汇编语言中，`ret`指令是一个伪指令，用于从函数返回到调用者。实际上，`ret`伪指令等价于`jalr`指令的一个特定用法，即跳转回到`ra`（返回地址寄存器，通常是`x1`寄存器）中保存的地址。因此，执行`ret`指令意味着当前执行流将返回到`ra`寄存器中存储的地址，也就是之前调用当前函数的地点。

用于xv6的  `swtch` 的过程：

- sret

`sret`指令

- `sret`（Supervisor Return）指令用于从监督者模式（S模式）的异常或中断处理程序返回到之前的特权级别。这是一条特权指令，用于在处理S模式下的异常或中断后，根据`sstatus`寄存器中的SPP（Supervisor Previous Privilege）位返回到S模式或U模式。
- `sret`指令会根据`sstatus`寄存器的SPP位来恢复CPU的特权级别。如果SPP位设置为0，表明在异常或中断发生前CPU处于U模式，那么执行`sret`后CPU将返回到U模式。如果SPP位为1，CPU将保持在S模式。
- `sret`执行时还会恢复其他由`sstatus`寄存器控制的状态，如中断使能状态。

ret 和 sret 的差异

主要差异

- 使用上下文：`ret`主要用于常规函数调用的返回，而`sret`用于从S模式的异常或中断处理程序返回到之前的执行模式。
- 特权级别：`ret`不涉及特权级别的改变，`sret`则用于在特权级别之间切换，尤其是从S模式返回到U模式或保持在S模式。
- 控制寄存器：`ret`基于`ra`寄存器进行跳转，而`sret`的行为由`sstatus`寄存器的状态（如SPP位）控制。

理解这两个指令的区别对于编写和理解在不同特权级别运行的RISC-V程序很重要，尤其是在涉及异常和中断处理时。



