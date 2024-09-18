# 6.1 xv6 note - scheduling 篇



scheduling 的核心函数 scheduler 是实现了一个非常精巧的**协程**，可以在一个执行流中，切换不同的进程信息，也正因为使用了协程，导致在阅读和理解这部分的代码的时候，有很多非常困惑的地方，比如不知道当前程序会跳转到哪里，以及后面实现lab的时候，debug进程的堆栈，经常发现当前的线程堆栈信息会过一会出现在其他线程中，也是这个原因。详细阅读和分析，确实能获得之前没有过的一些视角

# schedule 所面临的一些问题：

## Switch cpu 从一个进程到另外一个进程的场景：

- sleep 和 wakeup 机制中：
  - 进程等待 device 或者 pipe I/O 完成
  - 等待子进程停止
  - 等待sleep 系统调用
- 周期地强制切换，涉及进程长时间运行而没有显式调用sleep





## 实现过程中的一些挑战：

1. 如何从一个进程切换到另外一个进程，虽然用上下文切换最简单，但是实现起来却比较晦涩

2. 如何用一个方法强制切换，可以对用户进程透明？

3. 所有的CPU切换发生在同样的进程列表中，锁是必须的，来避免 races

4. 进程的内存资源必须freed, 当进程退出，它不能自己析构自己的内核栈，因为它自己还在用

5. 每一个cpu都需要记住哪个进程在执行，这样系统调用才能影响正确进程的内核状态

6. sleep 和 wakeup 允许进程放弃cpu，同时等待被其他进程或者中断唤醒，需要小心避免races, 防止对wakeup 通知的丢失。





## 一些问题：

1. Context switching 为什么需要一个额外的scheduler线程？
   1. 
   2. The xv6 scheduler has a dedicated（独立的） thread (saved registers and stack) per CPU  because it is not safe for the scheduler to execute on the old process’s kernel stack: some other core might wake the process up and run it, and it would be a disaster(灾难) to use the same stack on two different cores.

想要实现线程资源的相互交换，最好是有一个单独CPU(进程)的交换空间，这样，可以方便交换，而且可以避免在交换时，存在一些时间段，当第一个线程的寄存器和栈指针赋值给另外一个，而还没有相互交换时，造成两个CPU同时可能执行相同栈信息的可能。

​       而且如果swtch之后，在一个进程的内存空间下，设置另外一个进程的寄存器信息，本身就可能出现问题，最好是由一个独立的中间进程来进行切换和过度





2. 在交换的过程中，栈指针sp可以保存，不过栈内存段本身如何保存？还是说无需保存的必要

​      一个进程的kernel 模式的栈内存，是预先分配好的，位于内核空间中，当一个进程从用户空间切换到内核空间后，直接从这个进程对应的栈空间开始执行代码，而用户模式下的栈空间，则是在exec过程中才进行创建物理内存，以及进行虚拟地址和实际物理地址的映射，以及后面的一些压入参数的操作。

所以，也就是当执行完fork，而没有执行exec时，子进程完全用的是父进程的栈空间，那不是会有问题么？所以，在fork中，会执行`uvmcpy` 的函数，完全将用户空间的内存拷贝一份，所以可以让frok返回后，正常继续执行。

不过在exec 时，是否需要先释放之前申请的栈物理空间？因为后面exec时确实也会申请的。。。确实，在exec中，最后会将old_sz大小的所有旧的空间清理掉。

回到这个问题，当switch的过程中，确实会将sp 和ra寄存器的值进行保存，不过，等到执行的时候，寄存器和物理地址是可以对应上的。目前，将进程从 `RUNABLE` 状态，切换到 `RUNNING`状态时，  都只能在scheduler 函数中完成，而在scheduler中，是对进程列表进行遍历，来完成 switch 切换的，它肯定是用对应进程里面保存的 `context` 信息来替换CPU的寄存器的。所以，对应的寄存器是和进程能对应上，所以不牵扯栈内存本身的保存问题。







# Schedule 过程描述



## context 结构体：

```C
// Saved registers for kernel context switches.

// 为什么这里的缓存的寄存器，只有 ra, sp, 而没有 trapframe 里面的epc?
// 应该是，这个schedule过程，只会发生在 kernel space 空间中，不会让用户空间的代码参与
// 所以才无需缓存 epc 等指针
struct context {
  // ra 寄存器保存着函数调用时的返回地址，也就是swtch调用后的返回地址，而不是当前的pc指针~
  uint64 ra;
  uint64 sp;

  // callee-saved
  uint64 s0;
  uint64 s1;
  uint64 s2;
  uint64 s3;
  uint64 s4;
  uint64 s5;
  uint64 s6;
  uint64 s7;
  uint64 s8;
  uint64 s9;
  uint64 s10;
  uint64 s11;
};
```



## swtch 函数（汇编）

```C
# Context switch
#
#   void swtch(struct context *old, struct context *new);
# 
# Save current registers in old. Load from new.        

# a0  表示 old context的地址
# a1  表示 new context的地址

.globl swtch
swtch:
        sd ra, 0(a0)   #   ra -> 0(a0)
        sd sp, 8(a0)
        sd s0, 16(a0)
        sd s1, 24(a0)
        sd s2, 32(a0)
        sd s3, 40(a0)
        sd s4, 48(a0)
        sd s5, 56(a0)
        sd s6, 64(a0)
        sd s7, 72(a0)
        sd s8, 80(a0)
        sd s9, 88(a0)
        sd s10, 96(a0)
        sd s11, 104(a0)

        ld ra, 0(a1)    #  0(a1)  ->  ra
        ld sp, 8(a1)
        ld s0, 16(a1)
        ld s1, 24(a1)
        ld s2, 32(a1)
        ld s3, 40(a1)
        ld s4, 48(a1)
        ld s5, 56(a1)
        ld s6, 64(a1)
        ld s7, 72(a1)
        ld s8, 80(a1)
        ld s9, 88(a1)
        ld s10, 96(a1)
        ld s11, 104(a1)
        
        // ret 之后，执行流会跳转到  ra寄存器的地址，也就是说，会跳转到a1->ra 中
        ret
```



注意，上面的过程不是 `swap` 式的过程, 不是将 *old 的内容和 *new 的内容完全对调

而是，将当前CPU的寄存器先放在 *old里面，然后将 *new的内容，放在CPU的寄存器中，所以这里的用法，其实是相当于每一个进程都有它自己运行时刻寄存器的备份，只不过选择什么时间去替换到CPU中，作为执行的存在而已。



## scheduler 函数

```C
// Per-CPU process scheduler.
// Each CPU calls scheduler() after setting itself up.
// Scheduler never returns.  It loops, doing:
//  - choose a process to run.
//  - swtch to start running that process.
//  - eventually that process transfers control
//    via swtch back to the scheduler.
void
scheduler(void)
{
  struct proc *p;
  struct cpu *c = mycpu();
  
  c->proc = 0;
  for(;;){
    // Avoid deadlock by ensuring that devices can interrupt.
    // enable them to avoid a deadlock if all processes are waiting.
    // 非常重要！必须增加的原因
    // 
    // 因为在 进程sched 之前，acquire 会将 intr的状态信息全部保留，所以不会触发中断，而等下面的
    // loop执行中, 在acquire之后，也会收走触发中断的状态，所以就需要在cpu执行scheduler的时候，
    // 短暂地允许触发,这样，可以完成一些驱动的操作，从而再次wake 进程状态，从而才可能通过
    // if(p->state == RUNNABLE) 的判断
    // 如果不增加，将会可能导致所有的进程，都无法被wakeup，进而全部死锁
    
    // 执行前，sstatus的值是0x0, 执行后，立即变成 0x22
    intr_on();

    for(p = proc; p < &proc[NPROC]; p++) {
      // 注意，这里的acquire，必须要完整覆盖 swtch 的过程，
      
      // 经过这个 acquire 之后，intr_on的状态会保存， 然后 sstatus 的值会变成 0x20
      // 所以 0x20 这个 SSTATUS_SPIE 位，暂时可以不考虑
      acquire(&p->lock);
      if(p->state == RUNNABLE) {
        // Switch to chosen process.  It is the process's job
        // to release its lock and then reacquire it
        // before jumping back to us.
        p->state = RUNNING;
        
        // 重要，这个地方是将当前需要执行的进程信息和cpu相关联了
        c->proc = p;
        swtch(&c->context, &p->context);  // set p->context to ready, and set this time reg to mycpu->context (run proc->context)

        // Process is done running for now.
        // It should have changed its p->state before coming back.
        c->proc = 0;
      }
      release(&p->lock);
    }
  }
}
```



如果在swtch 前，没有acquire lock 的话，可能会造成一些问题：

在yield中，如果将state 赋值为 RUNNABLE，那么在调用后面 swtch之前，有可能有两个线程执行相同的堆栈信息，将会导致严重的问题。



### 问题一： 

为什么循环开始的地方执行一次  intr_on，是非常重要的？

TODO, 简单说下：

因为这儿是让业务层正常开启定时器和trap中断的地方，请参考锁部分的介绍    xv6 note - lock部分介绍，以及上面代码的注释



## sched 函数

```C
// Switch to scheduler.  Must hold only p->lock
// and have changed proc->state. Saves and restores
// intena because intena is a property of this
// kernel thread, not this CPU. It should
// be proc->intena and proc->noff, but that would
// break in the few places where a lock is held but
// there's no process.

// 在sched之前，需要进程保证：
/*
1. hold  p->lock
2. 执行过 push_off
3. state不能是RUNNING状态
4. 设备中断不能开启
*/
void
sched(void)
{
  int intena;
  struct proc *p = myproc();

  if(!holding(&p->lock))
    panic("sched p->lock");
  if(mycpu()->noff != 1)
    panic("sched locks");
  if(p->state == RUNNING)
    panic("sched running");
  if(intr_get())
    panic("sched interruptible");

  intena = mycpu()->intena;
  swtch(&p->context, &mycpu()->context);
  mycpu()->intena = intena;
}
```



The only place a kernel thread gives up its CPU is in sched



## sleep 函数

**`chan` 不是某个单词的缩写。而是在编程中通常用来指代“channel”或一个通信或同步机制中的“等待通道”。在许多操作系统的内核编程中，特别是涉及进程或线程同步的场合，`chan` 是一个常见的术语，用来标识特定的等待条件或资源。**

```C
// Atomically release lock and sleep on chan（channel）.
// Reacquires lock when awakened.
void
sleep(void *chan, struct spinlock *lk)
{
  struct proc *p = myproc();
  
  // Must acquire p->lock in order to
  // change p->state and then call sched.
  // Once we hold p->lock, we can be
  // guaranteed that we won't miss any wakeup
  // (wakeup locks p->lock),
  // so it's okay to release lk.

  // 先用 spin-lock 来锁上，p->lock， 之后释放lk
  acquire(&p->lock);  //DOC: sleeplock1
  release(lk);

  // Go to sleep.
  p->chan = chan;
  p->state = SLEEPING;   //  ?

  sched();

  // Tidy up. 整理
  p->chan = 0;

  // Reacquire original lock.
  // 问题？
  release(&p->lock);
  acquire(lk);
}
```



### 问题一

 前面acquire(&p->lock)后，如何保证后面  `release(&p->lock)` 的时候，可以保证经过sched() 后，还是在同一个cpu上执行？因为release(&p->lock) 的里面 holding函数 确实会对是否在同一个cpu进行校验的

这个问题回答起来比较复杂一些，首先，经过sleep后，执行流进入到了 scheduler 的后半段，也就是 `swtch` 函数调用之后的部分，那么这样看，acquire(p->lock)其实和scheduler最后的 release(&p->lock) 是一个完成的过程

```C
        swtch(&c->context, &p->context);  // set p->context to ready, and set this time reg to mycpu->context (run proc->context)

        // Process is done running for now.
        // It should have changed its p->state before coming back.
        c->proc = 0;
      }
      release(&p->lock);
    }
```

也就是说，每一个具体的任务进程，其实都是由scheduler中通过 swtch调度来产生的，所以一个新进程的起始位置：

```C
// A fork child's very first scheduling by scheduler()
// will swtch to forkret.
void
forkret(void)
{
  static int first = 1;

  // Still holding p->lock from scheduler.
  // yes, 先解锁
  release(&myproc()->lock);

  if (first) {
    // File system initialization must be run in the context of a
    // regular process (e.g., because it calls sleep), and thus cannot
    // be run from main().
    first = 0;
    fsinit(ROOTDEV);
  }

  usertrapret();
}
```

会先进行解锁操作，其实也就是解锁的 scheduler 函数在调用swtch前的 acquire过程。

相当于，sleep的 sched 操作，只不过是将之前从scheduler 中`抢`到的执行权限，换给了scheduler。等后面其他进程调用wakeup的时候，会重新在scheduler中来轮询，加锁后，并重新通过swtch来获取到执行流的权限，开始执行，当然，在 sched 函数执行之后，还是会进行release过程，不过这个release，其实是对应了scheduler 调用swthch前面的acuqire过程。

因为前后是的swtch过程，其实是同一个cpu所执行的，那么必然，是可以通过锁里面的holding函数的校验。

**简单回答，这个是一个配对问题，scheduler  swtch函数之前的acquire ，对应的是 sched()之后的 releaes。**

**而sched() 之前的acquire,则对应了 scheduler swtch函数之后的release.**



### 问题二：

为什么当一个执行流通过sleep等操作的swtch，将执行权限交给scheduler后，很快就可以再次获取到执行权限？

是因为：

任何一个正常的process在执行过程中，都会间隔一定时间，触发一个定时器的中断，然后对当前的执行流进行一次设置为唤醒状态的操作。(虽在在执行wakeup函数的时候，会用chan来对唤醒源进行一次判断)，那么可以保证，过一个间隔的时间，一个执行中的进程一定会被 设置为sleep的状态然后进行swtch，而且另外一个进程也一定会被调整为wakeup的状态。（后者一个执行中的进程会被设置为RUNNABLE, 然后进行swtch, 而且另外一个进程也一定会被重新设置为RUNNING）

再通过scheduler地不断轮询，那么注定任何一个进程都不会长时间地被阻塞，或者没有被调度到。也就保证了所有进程正常的执行情况。



### 问题三：

swtch可以直接更换执行流，为什么？

是因为：

1. 在执行的时候，当前的进程属于内核态中，而schduler 函数也处于内核态中，相当于它们处在同一种运行态中
2. 而且scheduler进程的栈和其他fork-exec 出来的进程栈是处于同一个内存空间中的。 Scheduler的栈空间在BSS段中（data段），所以，切换下sp以及其他的寄存器是非常容易的事情。不存在寄存器和实际物理内存不对应的问题。

​     通过打印，也确认了，在sleep调用 sched的时候，sp的指针位于： 0x0000003fffffced0，处于内核态的用户栈的位置，而scheduler在调用 swtch的时候，sp指针位于 0x000000008001f180，应该是在BSS的静态数组空间中。

​     所以，实际上发生的swtch行为，其实是在一个内存空间中完成的，因为切换前后，sp寄存器的变化，都能执行，所以才可以自由地发生切换。



## wakeup 函数

```C
// Wake up all processes sleeping on chan.(channel)
// Must be called without any p->lock.
void
wakeup(void *chan)
{
  struct proc *p;

  for(p = proc; p < &proc[NPROC]; p++) {
    // 为什么会有这个判断？
    if(p != myproc()){
      acquire(&p->lock);
      if(p->state == SLEEPING && p->chan == chan) {
        p->state = RUNNABLE;
      }
      release(&p->lock);
    }
  }
}
```

- 问题：为什么会有遍历所有进程列表时，判断是否为当前CPU所执行的进程？

存在这个判断，也就意味着：它只将当前cpu上执行的进程的状态设置为 RUNNABLE ! 也就是说，只会唤醒当前正在执行的进程



# 切换相关的一些工具函数：

## 获取当前的cpu信息以及进程信息

```C
// Must be called with interrupts disabled,
// to prevent race with process being moved
// to a different CPU.
int
cpuid()
{
  int id = r_tp();
  return id;
}

// Return this CPU's cpu struct.
// Interrupts must be disabled.
struct cpu*
mycpu(void)
{
  int id = cpuid();
  struct cpu *c = &cpus[id];
  return c;
}

// Return the current struct proc *, or zero if none.
struct proc*
myproc(void)
{
  // 必须调用 poush_off
  push_off();
  struct cpu *c = mycpu();
  struct proc *p = c->proc;
  pop_off();
  return p;
}
```