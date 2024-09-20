# 8.3 Logging

## 1. 一句话描述：

日志记录层，用于将多个 transaction 事务，统一进行提交。如果对同一个block重复进行写入buf，那么只需要写入最后一次buf内容即可，相当于做了写入层的优化。如果一**次写入多个data block时**发生了异常，可以进行数据回滚恢复，**不会让data block中出现写入了一半的data block块的情况。**

回滚的原理：

磁盘中有一个专门用于存放提交事务block的区域，有30个block, 也就是可以最多记录30个区块

写入流程是：

1. 将待修改的缓存，写入log区域的磁盘中，然后再将log区域磁盘的头部区块进行更新
2. 之后才是真正的往数据区的block磁盘区域写入
3. 最后才是清理掉log区域磁盘的头部区域

一般来说，如果前一次有磁盘没有真正写入数据磁盘取，也是首先读取log区头部的区块，然后按照头部区块的block_id数组，将log区域的磁盘数据，恢复到buf中，然后再写入实际的数据磁盘取

最后将头部区块清零



**问题：logging 和 buffer cache，对于重复读写的优化，作用分别如何**？

1. buffer cache 对重复读写磁盘的优化，主要是是在磁盘层面，如果同时有多个进程读取同一个磁盘的内容，如果第一次读取的buf还有效，即没有被brelease的话，可以被重复使用。暂时没有对重复写入做优化



2. log 对重复读写磁盘的优化，主要是在函数调用层面

基本上是集中在 begin_op 和 end_op之间，因为调用一次 begin_op和 end_op， 必然会commit 一次数据，优化只是如果有其他的调用发生在begin_op 和 end_op之间，那么将共同commit 一次，而不是两次。减少了commit的次数。

而针对具体的磁盘操作的话。在执行一次begin_op和 end_op的过程中，针对当不同的进程对同一个文件buf进行操作，那么将在log层面进行一次过滤，避免连续针对同一个buf调用 bwrite。

也就是说，如果两个进程是针对不同的磁盘文件进行操作，那么log层面最多保证将两次commit合并为一次，但是具体在buf层面，是无法进一步做优化的，调用磁盘的次数不会变化



**问题：在多大程度上，可以保证写入数据的完整性呢**？

一个commit的过程如下：

```C++
static void
commit()
{
  if (log.lh.n > 0) {
    write_log();     // Write modified blocks from cache to log
    write_head();    // Write header to disk -- the real commit
    install_trans(0); // Now install writes to home locations
    log.lh.n = 0;
    write_head();    // Erase the transaction from the log
  }
}
```

当完成 write_log + write_head之后，基本上即使执行install_trans时，系统退出，下一次启动，也会将这部分数据在 recover_from_log 中，完成写入到对应的data block中。而不会让待操作的多个data block存在半截数据的状态

如果write_head 没有调用完毕，那么基本上系统退出后，下一次启动，也是会到调用之前的状态。

而如果write_log 和 write_head 之后，已经执行了install_trans，那么如果最后write_head没有被执行掉，而系统退出，那么下一次启动，最多会重复写入log里面暂存的数据，最终的状态是完整的。

所以，commit中的 write_head是分界线，可以尽量保证data block里面的数据不会出现无效的不确定状态，增加了文件系统的确定性。

如果深究，那么就是write_head里面的 bwrite 操作是否完成决定的，**本质上相当于将一次多个data block的操作完整性，由一个log head block块的写入的完成已否来标志。大大提高了系统的可靠性。因为磁盘的一个block基本上只有写成功与没有写成功两种**。





## 2. Logging 的使用范式：

```C++
begin_op()

...

bp = bread(...)
modify bp->data[]
log_write(bp)
brelse(bp)

...

end_op();
```





## 3. 上层调用方式：

### 1. log_wirte

这是一种调用的方式，主要作用是将buf 结构体需要变动的消息告知log层，

```C++
bp = bread(...)
modify bp->data[]
log_write(bp)
brelse(bp)
```

因为在一开始的地方，会判断当前  log.outstanding 必须大于1，所以推测，这个函数的调用，必须要在 begin_op 和 end_op 之间进行，

### 2. begin_op   + end_op

经过查看代码，基本上可以确定，代码中，所有需要调用log_write的地方，都需要用 begin_op  和  end_op 来进行包裹才可以。

给我的一个启发：当设计一种接口时，可以先把内部的接口关系全部理顺，比如 log_write 和  begin_op 以及 end_op，这样，就可以像这里的使用一样，只要在所有 log_write出现的地方，都包裹上 begin_op 和 end_op， 则必然不会出错。这样也给使用者以方便，可以不管中间出现的非常复杂的场景，只需要把一头一尾处理好即可，后面修改和维护内部代码，也更简单，无需考虑太多东西

可以推测，当时作者也是这样处理的，先把将所有 log_write的地方都写完，然后在sys_call的函数头尾增加  begin_op 以及 end_op, 后面有遗漏，直接添加就可以，这样  log层 和系统调用层之间就可以完全不去细究逻辑到底如何。让整体的代码结构更加简单



## 4. log 结构体

```C
// Simple logging that allows concurrent（同时发生的）FS system calls.
//
// A log transaction contains the updates of multiple FS system
// calls. The logging system only commits when there are
// no FS system calls active. 
// Thus there is never
// any reasoning required about whether a commit might
// write an uncommitted system call's updates to disk.
// 因此，无需推理提交是否会将未提交的系统调用的更新写入磁盘。

// A system call should call begin_op()/end_op() to mark
// its start and end. Usually begin_op() just increments
// the count of in-progress FS system calls and returns.
// But if it thinks the log is close to(接近) running out(用完), it
// sleeps until the last outstanding end_op() commits.
//
// The log is a physical re-do log containing disk blocks.
// The on-disk log format:
//   header block, containing block #s for block A, B, C, ...
//   block A
//   block B
//   block C
//   ...
// Log appends are synchronous.  // log 的追加是同步的
```



下面的 log 结构体，仅仅作为缓存在

```C
// 头块的内容，用于磁盘上的头块以及在提交之前在内存中跟踪已记录的块号。
// Contents of the header block, used for both the on-disk header block
// and to keep track in memory of logged block# before commit.
struct logheader {
  int n;              // 表示：
  int block[LOGSIZE];    // 30  磁盘中的log block 也是30
};

struct log {
  struct spinlock lock;
  int start;
  int size;
  int outstanding; // how many FS sys calls are executing.
  int committing;  // in commit(), please wait.
  int dev;         // // 理论上，一种文件设备，比如磁盘，就要有一种对应的log
  struct logheader lh;  // 大小不超过  BSIZE  1024， 因为logheader就放在一个buf的data里面，大小1024
};
```



## 5. Log在磁盘中的布局：

```C++
 [   boot block   |   super block   |                    log                    |   
 
         1                  1                             30         
```

super block 的序号位于 1

log的block的序号位于 2，也就是从第三个block开始， 然后在log中，第一个是保存 struct logheader 的内容，余下的才是真正的 log block ，所以后面遍历，都需要 额外 + 1

而且从遍历的方式看，缓存的log是按顺利从头开始放置的

和 logheader 里面的block数组的block_id 顺序一致



## 6. 关键函数分析:

### 1. begin_op 函数

```C
// called at the start of each FS system call.
void
begin_op(void)
{
  acquire(&log.lock);
  while(1){
    if(log.committing){
      sleep(&log, &log.lock);

  // #define LOGSIZE      (MAXOPBLOCKS*3)  // max data blocks in on-disk log,  LOGSIZE:30
 // 保证本次的操作，不会最终超过LOGSIZE的大小，所以是在当前所有次数  log.outstanding 的基础上，增加1，来进行计算
    } else if(log.lh.n + (log.outstanding+1)*MAXOPBLOCKS > LOGSIZE){
      // this op might exhaust log space; wait for commit.
      sleep(&log, &log.lock);
    } else {
      log.outstanding += 1;
      release(&log.lock);
      break;
    }
  }
}
```



### 2. end_op 函数

```C
// called at the end of each FS system call.
// commits if this was the last outstanding operation.
void
end_op(void)
{
  int do_commit = 0;

  acquire(&log.lock);
  log.outstanding -= 1;
  if(log.committing)
    panic("log.committing");
  if(log.outstanding == 0){
    do_commit = 1;
    log.committing = 1;
  } else {
    // begin_op() may be waiting for log space,
    // and decrementing log.outstanding has decreased
    // the amount of reserved space.
    wakeup(&log);
  }
  release(&log.lock);

  if(do_commit){
    // call commit w/o holding locks, since not allowed
    // to sleep with locks.
    
    commit();
    acquire(&log.lock);
    log.committing = 0;
    wakeup(&log);
    release(&log.lock);
  }
}
```



**问题：为何这里的commit 无需加锁**?

首先，commit 里面的操作都是直接针对 log.lh.n  和  log.lh.block, 而另外一个对这两个变量操作的函数是  log_write。

可以明确的是 log_write 里面是加锁的，所以需要保证  commit 在执行的时候，log_write 是不能执行的。

通过log_write里面一开始对 log.outstanding 的判断来看，必须是 log.outstanding >= 1,  而outstanding >=1 的时候，必然是  begin_op 和 end_op 的执行期间。而且是有锁的。所以也就保证了commit 的直接操作，可以通过无锁的形式来做。



### 3. commit 函数

```C++
/**
 * 先缓存
 * 1. 先将缓存buf中的数据，转移到 log的磁盘位置，并写入磁盘
 * 2. 更新log磁盘位置的 logheader结构体，并写入磁盘
 * 真正写入
 * 3. 按照logheader的数据，真正写入实际的data blocks区域的磁盘
 *    这里为何要将目标 data buf refcnt 进行递减操作？ 是因为一开始在 log_write 过程中，已经将ref递增过一次，所以这里进行递减
 * 4. 将log.lh.n的数据清零
 * 5. 更新log磁盘位置的 logheader结构体，并写入磁盘
*/

static void
commit()
{
  if (log.lh.n > 0) {
    write_log();     // Write modified blocks from cache to log
    write_head();    // Write header to disk -- the real commit
    install_trans(0); // Now install writes to home locations, from log to cache
    log.lh.n = 0;
    write_head();    // Erase the transaction from the log
  }
}
```



### 4. log_write 函数

用于当使用者修改了 buffer里面的数据，然后需要在logging中记录这个被修改的block的number信息，并将buffer里面的 **refcnt** 字段加1。

```C
// Caller has modified b->data and is done with the buffer.
// Record the block number and pin in the cache by increasing refcnt.
// commit()/write_log() will do the disk write.
//
// log_write() replaces bwrite(); a typical use is:
//   bp = bread(...)
//   modify bp->data[]
//   log_write(bp)
//   brelse(bp)
void
log_write(struct buf *b)
{
  int i;

  acquire(&log.lock);
  if (log.lh.n >= LOGSIZE || log.lh.n >= log.size - 1)
    panic("too big a transaction");
  if (log.outstanding < 1)
    panic("log_write outside of trans");

  for (i = 0; i < log.lh.n; i++) {
    if (log.lh.block[i] == b->blockno)   // log absorption
      break;
  }
  log.lh.block[i] = b->blockno;
  if (i == log.lh.n) {  // Add new block to log?
    bpin(b);
    log.lh.n++; // 第一次记录到  log 中，需要增加buf的ref, 避免之后执行brelse后，缓存失效
  }
  release(&log.lock);
}
```



