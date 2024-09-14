

# traps相关的代码

##  1. traps from user space

the first handler instructions are usually written in assembler (rather than C) and are sometimes called a *vector*. 

比如下面这个：uservec

> A major constraint（主要的限制）on the design of xv6’s trap handling is the fact that the RISC-V hardware does not switch page tables when it forces a trap.
>
> *RISC-V的硬件不会在trap生效时，切换 page-tables.*
>
> This means that the trap handler address in *stvec* **must have a valid mapping in the user page table**, since that’s the page table in force(生效) when the trap handling code starts executing.
>
> 
>
> Xv6’s trap handling code needs to switch to the kernel page table; in order to be able to continue executing after that switch, the kernel page table must also have a mapping for the handler pointed to by *stvec*. 
>
> 当需要在trap中切换到kernel的page table时，需要在stvec 中对 kernel 的page table 进行一些mapping操作

> The trampoline page is mapped in every process’s page table at address *TRAMPOLINE*, which is at the top of the virtual address space so that it will be above memory (that programs use for themselves).
>
> 就在所有进程的 TRAMPOLINE的位置映射上 trampoline。
>
> 
>
> Because the trampoline page is mapped at the **same address** in the kernel address space, the trap handler can continue to execute after it switches to the kernel page table.
>
> trampoline 会映射到相同的地址，所以后面处理trap相关过程中，也是需要以它为基准地址来查找对应的符号地址的，下面代码中也有描述



### 步骤1：uservec 汇编函数

kerenel/trampoline.S：21

```Assembly
.globl uservec
uservec:    
        #
        # trap.c sets stvec to point here, so
        # traps from user space start here,
        # in supervisor mode, but with a
        # user page table.
        #

        # save user a0 in sscratch so
        # a0 can be used to get at TRAPFRAME.
        # 第一步：将a0 写入 sscratch
        csrw sscratch, a0

        # each process has a separate p->trapframe memory area,
        # but it's mapped to the same virtual address
        # (TRAPFRAME) in every process's user page table.
        #
        # 将 TRAPFRAME 加载到 a0，其实是为了通过 TRAPFRAME 来获取这些地址对应的page页面
        # ？为什么不能直接通过 TRAPFRAME 来进行递增？
        # 有可能是不能用立即数来做偏移操作
        li a0, TRAPFRAME

        # save the user registers in TRAPFRAME
        # 第二步，将当前寄存器的数据写入到 trapframe的page中
        # 将寄存器 ra 中的值存储到以寄存器 a0 中的地址加上 40 字节偏移量的内存位置
        # 也就是 TRAPFRAME+40
        sd ra, 40(a0)
        sd sp, 48(a0)
        sd gp, 56(a0)
        sd tp, 64(a0)
        sd t0, 72(a0)
        sd t1, 80(a0)
        sd t2, 88(a0)
        sd s0, 96(a0)
        sd s1, 104(a0)
        sd a1, 120(a0)
        sd a2, 128(a0)
        sd a3, 136(a0)
        sd a4, 144(a0)
        sd a5, 152(a0)
        sd a6, 160(a0)
        sd a7, 168(a0)
        sd s2, 176(a0)
        sd s3, 184(a0)
        sd s4, 192(a0)
        sd s5, 200(a0)
        sd s6, 208(a0)
        sd s7, 216(a0)
        sd s8, 224(a0)
        sd s9, 232(a0)
        sd s10, 240(a0)
        sd s11, 248(a0)
        sd t3, 256(a0)
        sd t4, 264(a0)
        sd t5, 272(a0)
        sd t6, 280(a0)

        # save the user a0 in p->trapframe->a0
        # 为啥不能 sd sscratch, 112(a0) ?
        csrr t0, sscratch
        sd t0, 112(a0)

        # 第三部，从已经保存的 trapframe中，读取数据，然后重新放入当前寄存器
        # 为切换到内核态做准备
        # initialize kernel stack pointer, from p->trapframe->kernel_sp
        # 将地址寄存器 a0 中的值加上 8，然后将得到的地址处的内容加载到栈指针寄存器 sp 
        ld sp, 8(a0)

        # make tp hold the current hartid, from p->trapframe->kernel_hartid
        ld tp, 32(a0)

        # load the address of usertrap(), from p->trapframe->kernel_trap
        # t0   usertrap()
        ld t0, 16(a0)


        # fetch the kernel page table address, from p->trapframe->kernel_satp.
        # t1   kernel_page_table
        ld t1, 0(a0)

        # wait for any previous memory operations to complete, so that
        # they use the user page table.
        sfence.vma zero, zero

        # install the kernel page table.
        #将 t1 写入 satp
        csrw satp, t1

        # flush now-stale user entries from the TLB.
        sfence.vma zero, zero
        
        # 第四部，跳回usertrap 中
        # jump to usertrap(), which does not return
        jr t0
```

Q: User space 的代码在什么时候可能会调用这个函数? 应该在任何都可能被调度走，真是这样吗？

A:是的





### 步骤2：usertrap 函数

```C
//
// handle an interrupt, exception, or system call from user space.
// called from trampoline.S
//
void
usertrap(void)
{
  int which_dev = 0;

  // 返回到这里后，SSTATUS_SPIE 的状态会被RISC-V给去掉，不管下面 usertrapret返回的
  // sstatus 是否携带
  if((r_sstatus() & SSTATUS_SPP) != 0)                     // 校验  sstatus
    panic("usertrap: not from user mode");

  // send interrupts and exceptions to kerneltrap(),
  // since we're now in the kernel.
  // 非常关键的地方，因为这里才设置 stvec 的寄存器，让他开始处理内核异常函数
  w_stvec((uint64)kernelvec);

  struct proc *p = myproc();
  
  // save user program counter.
  // epc 没有在uservec中进行保存
  p->trapframe->epc = r_sepc();                     // 保存 sepc 
  
  if(r_scause() == 8){                              // 校验 scause
    // system call

    if(killed(p))
      exit(-1);

    // sepc points to the ecall instruction,
    // but we want to return to the next instruction.
    //  epc 表示的是下一个指令的地址
    p->trapframe->epc += 4;

    // an interrupt will change sepc, scause, and sstatus,
    // so enable only now that we're done with those registers.
    
    // 留意~
    // 这里，执行前，sstatus的值是0x20,
    // 但是，执行后，sstatus的值就被修改为0x22
    intr_on();

    // 注意，这个地方将陷入内核态
    // 它的作用在于，这个地方之后，
    syscall();
  } else if((which_dev = devintr()) != 0){
    // ok
  } else {
    printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
    printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
    setkilled(p);
  }

  if(killed(p))
    exit(-1);

  // give up the CPU if this is a timer interrupt.
  if(which_dev == 2)
    yield();

  usertrapret();
}
```

An application that wants to invoke a kernel function (e.g., the read system call in xv6) must transition to the kernel; an application *cannot* invoke a kernel function directly. CPUs provide a special instruction that switches the CPU from user mode to supervisor mode and enters the kernel at an entry point specified by the kernel. (RISC-V provides the ***ecall*** instruction for this purpose.) 

问题： 为什么必须执行 intr_on ? 如果注释掉，会发生什么

如果注释掉，看起来会触发很多 usertrap 里面的设备中断，调用 **devintr** 函数





### 步骤3：usertrapret 函数

这个函数还有一个地方会调用，挺重要的，对于理解：

#### forkret 函数

会在 scheduler() 之后，被调度，并调用

```c
// A fork child's very first scheduling by scheduler()
// will swtch to forkret.
void
forkret(void)
{
  static int first = 1;

  // Still holding p->lock from scheduler.
  release(&myproc()->lock);

  if (first) {
    // File system initialization must be run in the context of a
    // regular process (e.g., because it calls sleep), and thus cannot
    // be run from main().
    first = 0;
    fsinit(ROOTDEV);
  }

  // 执行这个过程
  usertrapret();
}
```



```C
//
// return to user space
//
void
usertrapret(void)
{
  struct proc *p = myproc();

  // we're about(即将) to switch the destination of traps from
  // kerneltrap() to usertrap(), so turn off interrupts until
  // we're back in user space, where usertrap() is correct.
  
  // 当然，经过上面 intr_on 之后，sstatus 的值修改为0x22
  // 然后经过下面 intr_off 之后，会重新变成 0x20
  // 经过测试发现，这里不执行，也是正常的，可能是RISC-V自动进行了关闭
  // 将 0x22 变成了 0x20
  intr_off();

  // send syscalls, interrupts, and exceptions to uservec in trampoline.S
  // 计算todo: uservec - trampoline
  // 为何先 跳到 uservec，然后结束时，才执行 userret
  // 回答：跳到 uservec后，已经是用户态啦，执行 userret只是完成一些过程，switch page table
  // TRAMPOLINE +  0x0000000000000000


  // 为什么这里和下面的trampoline_userret 都要以 TRAMPOLINE作为基准进行代码段的偏移？
  // 1. 因为只有 TRAMPOLINE的虚拟地址，映射了 trampoline 代码段的数据
  //     if(mappages(pagetable, TRAMPOLINE, PGSIZE,
  //            (uint64)trampoline, PTE_R | PTE_X) < 0){
  //    所以才能解释为什么要做 uservec - trampoline  以及下面 userret - trampoline的 偏移计算
  //    是为了获取到实际 trampoline数据段中不同符号的位置
  // 2. 还有一个原因，是因为Trampoline是内核空间，用户空间映射的地址是一样的，这是risc-v的特点，参考
  //    RISC-V特性总览：关于 trap时的差别
  uint64 trampoline_uservec = TRAMPOLINE + (uservec - trampoline);
  w_stvec(trampoline_uservec);           // 恢复为用户的trap处理逻辑，即将离开内核态

  // set up trapframe values that uservec will need when
  // the process next traps into the kernel.
  p->trapframe->kernel_satp = r_satp();         // kernel page table
  p->trapframe->kernel_sp = p->kstack + PGSIZE; // process's kernel stack
  p->trapframe->kernel_trap = (uint64)usertrap;
  p->trapframe->kernel_hartid = r_tp();         // hartid for cpuid()

  // set up the registers that trampoline.S's sret will use
  // to get to user space.
  
  // set S Previous Privilege mode to User.
  unsigned long x = r_sstatus();
  x &= ~SSTATUS_SPP; // clear SPP to 0 for user mode

  // 设置是当重新进入内核中断后，退出了，可以继续enable SIE 吧？
  // 测试发现，sstatus 读取的值，包含了0x20的字段，所以下面这句话，注释掉，目前也无影响
  // 测试也发现，如果将这个 SPIE的字段清理掉，也是无影响的
  x |= SSTATUS_SPIE; // enable interrupts in user mode  ? 
  w_sstatus(x);

  // set S Exception Program Counter to the saved user pc.
  w_sepc(p->trapframe->epc);

  // tell trampoline.S the user page table to switch to.
  uint64 satp = MAKE_SATP(p->pagetable);

  // jump to userret in trampoline.S at the top of memory, which 
  // switches to the user page table, restores user registers,
  // and switches to user mode with sret.
  uint64 trampoline_userret = TRAMPOLINE + (userret - trampoline);
  // TRAMPOLINE  +   0x000000000000009c
  ((void (*)(uint64))trampoline_userret)(satp);

  // yes, 这个地方是永远不会被执行到的
  printf("&&&&&&&&&&&&&&&&&&&&&&&&&&\n");
}
```

这里面开头的 intr_off 函数，是为了在trap_handle 函数由 kerneltrap 函数转移到 usertrap，所以暂时关闭，直到进入用户空间，

问题： 为什么必须执行 intr_off 函数？注释掉会怎么样？

看起来注释掉，会卡在 usertrapret 函数返回之后

> At the end, usertrapret calls userret on the trampoline page that is mapped in both user and kernel page tables; the reason is that assembly code in userret will switch page tables.

在 usertrapret 函数的最后面，必然需要执行  ((void (*)(uint64))trampoline_userret)(satp);

因为这里会把cpu里面的寄存器信息都提前设置好，当执行完后，cpu会直接加载原有 trapframe 里面的epc 字段保存的地址，然后会从这个地方开始执行, 

所以理论上，这个代码后面打印的东西不会执行！！！

注意：**在fork之后，新进程第一次调度到这个函数的时候，这里的epc保存的其实就是调用frok函数的父进程的epc信息，所以在手动设置了 w_sepc  寄存器后，那么后面退出usertrapret时，必然执行的也会和父进程处于同一个起点**



### 步骤4：userret  汇编函数

kernel/trampoline.S：101

```Assembly
.globl userret
userret:
        # userret(pagetable)
        # called by usertrapret() in trap.c to
        # switch from kernel to user.
        # a0: user page table, for satp.

        # switch to the user page table.
        sfence.vma zero, zero
        csrw satp, a0
        sfence.vma zero, zero

        li a0, TRAPFRAME

        # restore all but a0 from TRAPFRAME
        ld ra, 40(a0)
        ld sp, 48(a0)
        ld gp, 56(a0)
        ld tp, 64(a0)
        ld t0, 72(a0)
        ld t1, 80(a0)
        ld t2, 88(a0)
        ld s0, 96(a0)
        ld s1, 104(a0)
        ld a1, 120(a0)
        ld a2, 128(a0)
        ld a3, 136(a0)
        ld a4, 144(a0)
        ld a5, 152(a0)
        ld a6, 160(a0)
        ld a7, 168(a0)
        ld s2, 176(a0)
        ld s3, 184(a0)
        ld s4, 192(a0)
        ld s5, 200(a0)
        ld s6, 208(a0)
        ld s7, 216(a0)
        ld s8, 224(a0)
        ld s9, 232(a0)
        ld s10, 240(a0)
        ld s11, 248(a0)
        ld t3, 256(a0)
        ld t4, 264(a0)
        ld t5, 272(a0)
        ld t6, 280(a0)

        # restore user a0
        ld a0, 112(a0)
        
        # return to user mode and user pc.
        # usertrapret() set up sstatus and sepc.
        sret
```

> The trampoline page mapping at the same virtual address in user and kernel page tables allows 
>
> userret to keep executing after changing satp. 



## 2：traps from kernel space

### 步骤1： kernelvec 汇编函数

需要留意一个点，在内核启动后，除了一个进程经过exec后，

```Assembly
.globl kernelvec
.align 4
kernelvec:
        # make room to save registers.
        # 将栈指针向下移动256个单位，因为立即数是负数，所以是减法操作
        addi sp, sp, -256

        # save the registers.
        # # 将寄存器 ra 中的值存储到以寄存器 sp 中的地址加上 0 字节偏移量的内存位置
        sd ra, 0(sp)
        sd sp, 8(sp)
        sd gp, 16(sp)
        sd tp, 24(sp)
        sd t0, 32(sp)
        sd t1, 40(sp)
        sd t2, 48(sp)
        sd s0, 56(sp)
        sd s1, 64(sp)
        sd a0, 72(sp)
        sd a1, 80(sp)
        sd a2, 88(sp)
        sd a3, 96(sp)
        sd a4, 104(sp)
        sd a5, 112(sp)
        sd a6, 120(sp)
        sd a7, 128(sp)
        sd s2, 136(sp)
        sd s3, 144(sp)
        sd s4, 152(sp)
        sd s5, 160(sp)
        sd s6, 168(sp)
        sd s7, 176(sp)
        sd s8, 184(sp)
        sd s9, 192(sp)
        sd s10, 200(sp)
        sd s11, 208(sp)
        sd t3, 216(sp)
        sd t4, 224(sp)
        sd t5, 232(sp)
        sd t6, 240(sp)

        # call the C trap handler in trap.c
        call kerneltrap

......
```

**kernelvec** saves the registers on the stack of the interrupted kernel thread, which makes sense because the register values belong to that thread. This is particularly important **if the trap causes a switch to a different thread – in that case the trap will actually return from the stack of the new thread**, leaving the interrupted thread’s saved registers safely on its stack.

因为有可能 kernelvec 可能会switch 线程，所以需要先保留当前寄存器的信息，后面在函数 kerneltrap中，会保持 pc寄存器的内容，所以会保证将继续执行 kernelvec剩余部分的代码

在一开始的时候 sp指针位于当前进程对应的内核栈的地方



### 步骤2：kerneltrap 函数

```C
// interrupts and exceptions from kernel code go here via kernelvec,
// on whatever the current kernel stack is.
void 
kerneltrap()
{
  int which_dev = 0;
  uint64 sepc = r_sepc();
  // printf后，目前观察到的都是 0x120 ,
  // gdb 后，观察到的，也是 0x120，为何不是0x122 ??
  uint64 sstatus = r_sstatus();
  uint64 scause = r_scause();
  
  if((sstatus & SSTATUS_SPP) == 0)
    panic("kerneltrap: not from supervisor mode");
  if(intr_get() != 0)
    panic("kerneltrap: interrupts enabled");

  if((which_dev = devintr()) == 0){
    printf("scause %p\n", scause);
    printf("sepc=%p stval=%p\n", r_sepc(), r_stval());
    panic("kerneltrap");
  }

  // give up the CPU if this is a timer interrupt.
  if(which_dev == 2 && myproc() != 0 && myproc()->state == RUNNING)
    yield();

  // the yield() may have caused some traps to occur,
  // so restore trap registers for use by kernelvec.S's sepc instruction.
  // 这里相当于将 pc 寄存器和  status 寄存器的信息进行了还原
  // 
  w_sepc(sepc);
  w_sstatus(sstatus);
}
```

问题:gdb 后，观察到的，也是 0x120，为何不是0x122 ??

可能在于，当进入这个kerneltrap函数的时候，这个 **`SSTATUS_SIE`** 是会自动清理掉的 , 等结束中断处理后，再被

 SSTATUS_SPIE 里保存的值进行自动恢复

> It’s worth thinking through how the trap return happens if kerneltrap called yield due to  a timer interrupt.





### 步骤3： 继续执行 kernelvec 汇编函数

为什么不用保存sp？ 在yield 存在的情况下，是因为yield只是让执行当前执行流暂时断开，然后执行其他 RUNNABLE 状态的进程，但是会保存所有当前执行相关的寄存器，就比如sp, 等后面某个cpu 有时间，然后继续这个暂停的进程，所以在从 kerneltrap 返回之后，可以直接用 sp来进行还原处理

```C
 ......
        # 将地址寄存器 sp 中的值加上 0，然后将得到的地址处的内容加载到栈指针寄存器 ra 
        # restore registers.
        ld ra, 0(sp)
        ld sp, 8(sp)
        ld gp, 16(sp)
        # not tp (contains hartid), in case we moved CPUs
        ld t0, 32(sp)
        ld t1, 40(sp)
        ld t2, 48(sp)
        ld s0, 56(sp)
        ld s1, 64(sp)
        ld a0, 72(sp)
        ld a1, 80(sp)
        ld a2, 88(sp)
        ld a3, 96(sp)
        ld a4, 104(sp)
        ld a5, 112(sp)
        ld a6, 120(sp)
        ld a7, 128(sp)
        ld s2, 136(sp)
        ld s3, 144(sp)
        ld s4, 152(sp)
        ld s5, 160(sp)
        ld s6, 168(sp)
        ld s7, 176(sp)
        ld s8, 184(sp)
        ld s9, 192(sp)
        ld s10, 200(sp)
        ld s11, 208(sp)
        ld t3, 216(sp)
        ld t4, 224(sp)
        ld t5, 232(sp)
        ld t6, 240(sp)

        addi sp, sp, 256

        # return to whatever we were doing in the kernel.
        sret
```



