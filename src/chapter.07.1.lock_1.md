# 7.1 锁相关

lock在整个risc-v 操作系统代码中，有非常重要的地位，因为多进程，多核处理的实现，中断触发后的处理，都很大程度依赖锁，这里对一些锁的种类，以及使用做一些介绍

# spin-lock 部分

 在保持 spinlock的时候， yielding cpu 是非法的，如果第二个线程尝试acquire spinlock时，可能会造成死锁，所以就需要一种锁，当它持有时，既可以 yield cpu资源，也可以允许中断发生。而且在 waiting to acquire时，也可以yield cpu资源，就是下面的 sleep-lock了



## acquire 函数

type __sync_lock_test_and_set(type *ptr, type value);

`ptr`：指向要设置的变量的指针。

- `value`：要设置的新值。
- 返回值：变量设置前的旧值。



函数尝试通过设置`lock`变量为1来获取锁。如果`lock`已经是1（表示锁已经被其他线程获取），`__sync_lock_test_and_set`将返回1，循环将继续，表示当前线程将继续等待直到锁被释放。

```C
// Acquire the lock.
// Loops (spins) until the lock is acquired.
void
acquire(struct spinlock *lk)
{
  push_off(); // disable interrupts to avoid deadlock.
  if(holding(lk))
    panic("acquire");

  // On RISC-V, sync_lock_test_and_set turns into an atomic swap:
  //   a5 = 1
  //   s1 = &lk->locked
  //   amoswap.w.aq a5, a5, (s1)
  // 
  // ptr：指向要设置的变量的指针。
  // value：要设置的新值。
  // 返回值：变量设置前的旧值。
  // 
  while(__sync_lock_test_and_set(&lk->locked, 1) != 0)
    ;

  // Tell the C compiler and the processor to not move loads or stores
  // past this point, to ensure that the critical section's memory
  // references happen strictly after the lock is acquired.
  // On RISC-V, this emits a fence instruction.
  // 内存屏障
  __sync_synchronize();

  // Record info about lock acquisition for holding() and debugging.
  lk->cpu = mycpu();
}
```

注意：

**acquire 里面， push_off 一定遭遇 lk->locked， 因为避免在获取锁后，关闭中断前，触发了中断，就会引起上面描述的死锁现象。同理 下面函数的 release 也一定遭遇 pop_off **





## release 函数

__sync_lock_release 将`lock`变量重置为0，这样其他线程就可以获取锁了。

```C
// Release the lock.
void
release(struct spinlock *lk)
{
  if(!holding(lk))
    panic("release");

  lk->cpu = 0;

  // Tell the C compiler and the CPU to not move loads or stores
  // past this point, to ensure that all the stores in the critical
  // section are visible to other CPUs before the lock is released,
  // and that loads in the critical section occur strictly before
  // the lock is released.
  // On RISC-V, this emits a fence instruction.
  // 内存屏障
  __sync_synchronize();

  // Release the lock, equivalent to lk->locked = 0.
  // This code doesn't use a C assignment, since the C standard
  // implies that an assignment might be implemented with
  // multiple store instructions.
  // On RISC-V, sync_lock_release turns into an atomic swap:
  //   s1 = &lk->locked
  //   amoswap.w zero, zero, (s1)
  //
  // 
  __sync_lock_release(&lk->locked);

  pop_off();
}
```



## holding 函数

```C
// Check whether this cpu is holding the lock.
// Interrupts must be off.
int
holding(struct spinlock *lk)
{
  int r;
  r = (lk->locked && lk->cpu == mycpu());
  return r;
}
```

通过它的判断，我们可以得到：

**它的锁是以cpu为单位的, 因为它判断了cpu_id**

# sleep-lock 部分

因为 sleep-lock 允许 interrupt 开启，所以不能用于 interrupt handle, 因为acquiresleep 可能 yield cpu信息。

Sleep-lock 不能放在 spinlock 保护的关键部分里面，

而 spinlock 可以放在 sleep-lock保护的关键部分里面。

所以也可以看到， interrupt handle 就是一种多线程的乱序执行，所以对锁的影响非常大

> Spin-locks are best suited to short critical sections, since waiting for them wastes CPU time; 
>
> sleep-locks work well for lengthy operations.



## acquiresleep 函数

```C
void
acquiresleep(struct sleeplock *lk)
{
  acquire(&lk->lk);
  while (lk->locked) {
    // 这里的sleep, 会自动 yield cpu 和释放 spin_lock
    // 所以当处于 wait时，其他线程可以继续执行
    sleep(lk, &lk->lk);
  }
  lk->locked = 1;
  lk->pid = myproc()->pid;
  release(&lk->lk);
}
```



## releasesleep 函数

```C
void
releasesleep(struct sleeplock *lk)
{
  acquire(&lk->lk);
  lk->locked = 0;
  lk->pid = 0;
  wakeup(lk);
  release(&lk->lk);
}
```



## holdingsleep 函数

```C
int
holdingsleep(struct sleeplock *lk)
{
  int r;
  
  acquire(&lk->lk);
  r = lk->locked && (lk->pid == myproc()->pid);
  release(&lk->lk);
  return r;
}
```



通过它的判断可以得出：

**它的锁是以进程为单位的，因为它判断了pid**





# 在interrupt handle 中的使用

一些 spin-lock 同时被用于 threads 和 interrupt handle，比如下面的 tickslock, 对ticks进行加锁处理

Interrupt handle:

```C
// timer interrupt handle

void
clockintr()
{
  acquire(&tickslock);
  ticks++;
  wakeup(&ticks);
  release(&tickslock);
}
```

Kernel thread

```C
uint64
sys_sleep(void)
{
  int n;
  uint ticks0;

  argint(0, &n);
  acquire(&tickslock);
  ticks0 = ticks;
  while(ticks - ticks0 < n){
    if(killed(myproc())){
      release(&tickslock);
      return -1;
    }
    sleep(&ticks, &tickslock);
  }
  release(&tickslock);
  return 0;
}
```



存在一种可能，

1. sys_sleep hold  tickslock
2. Cpu  is interrupted by a timer interrupt.
3. `clockintr` 尝试获取锁，但是需要等到 sys_sleep 释放
4. 实际上 sys_sleep 永远不会释放，因为需要等待timer interrupt 中的 clockintr 结束
5. 然后造成了事实上的CPU 死锁

为了解决这个问题，xv6规定：

**如果 spinlock 被用于一个 interrupt handler， CPU绝对不能在获取锁的时候，允许 interrupt 开启**

**if a spinlock is used by an interrupt handler, a CPU must never hold that lock with interrupts enabled.**



 Xv6 is more conservative: **when a CPU acquires any lock, xv6 always disables interrupts on that CPU**. Interrupts may still occur on other CPUs, so an interrupt’s  acquire can wait for a thread to release a spinlock; just not on the same CPU.



### push_off  &  pop_off 函数

当没有CPU 保持 spinlock时，xv6 会开启interrupt. 所以这里存在一个 book-keeping 来保存嵌套的关键内容

acquire 调用 push_off，增加嵌套

release 调用 pop_off，减少嵌套

来跟踪当前CPU嵌套lock的层数，当count为0， pop_off 也就会在最外层嵌套中，开启enable interrupt.

调用了push_off ， 最后再调用pop_off ，是为了创造出一个暂时关闭了 devie_interrupts enabled （SSTATUS_SIE 被清理）的空间，之后会对现场进行还原



```C
// push_off/pop_off are like intr_off()/intr_on() except that they are matched:
// it takes two pop_off()s to undo two push_off()s.  Also, if interrupts
// are initially off, then push_off, pop_off leaves them off.

void
push_off(void)
{
  // SSTATUS_SIE enable
  int old = intr_get();

  // 清空掉 SIE
  intr_off();
  if(mycpu()->noff == 0)
    mycpu()->intena = old;
  mycpu()->noff += 1;
}

void
pop_off(void)
{
  struct cpu *c = mycpu();
  if(intr_get())
    panic("pop_off - interruptible");
 
  if(c->noff < 1)
    panic("pop_off");
  
  // 功能，应该是，当 noff 归零，也就是 push_off 和 pop_off 相抵消后，就进行现场还原
  c->noff -= 1;
  if(c->noff == 0 && c->intena)
    intr_on();
}
```



### intr_off & intr_on 函数

会调用risc-v的指令来开启和关闭中断

```C
// enable device interrupts
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
```



#### 什么时候调用 intr_on 开启函数?

- 在scheduler 函数遍历进程开始的地方，而且要每一次遍历前，都要执行一次，是为了让新fork的进程也可以开启intr_on，具体来说，在每for循环每一次acquire的时候，会先push_off 一次这个状态，然后具体到swtch后，会通过release 方法来恢复这个状态。也就是会作用于正常内核状态中，都需要来接收中断
- 在 usertrap 的时候，进入 syscall 时，需要打开，因为这表示这里是将要进入内核状态了
- push_off 恢复状态的时候，如果之前是打开装填，则需要打开



#### 什么时候调用 intr_off 关闭函数？

- 进入 push_off 的时候，需要提前关闭，后面会在 po_off 的时候恢复的
- 由 usertrapret 进入用户态时，需要关闭。



待考虑：

1. 是否在用户空间中，设备中断是不能被触发的？
2. 定时器的中断，目前是哪种中断模式?



