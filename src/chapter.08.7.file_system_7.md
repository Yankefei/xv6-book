# 8.7 File descriptor



**问题：file的ref 引用计数作用是什么？有哪些好处**？

当 **open** 以及 **pipealloc** 的时候，会创建，以及递增这个ref,

系统调用 dup， 以及 fork 的时候，会增加ref

进程 exit, 系统调用close，以及在文件操作异常退出，清理资源时会调用,

这个ref在处理mmap的lab的时候，作用非常大，因为有文件被unlink后，还需要使用的场景，就需要用f->ref来判断了



目前看，ref的维护策略很简单粗暴，直接在 ftable.file 数组中遍历为0的分配，总大小是100个。

也只有当前进程在dup以及fork的时候，才会增加引用计数。

所以这个file descriptor里面的引用计数，**是仅限于进程内部使用**，维护单个进程内部多次引用文件的情况



## 1. 关键函数分析：



### 1. file 结构体

```C
struct file {
  enum { FD_NONE, FD_PIPE, FD_INODE, FD_DEVICE } type;
  int ref; // reference count
  char readable;
  char writable;
  struct pipe *pipe; // FD_PIPE
  struct inode *ip;  // FD_INODE and FD_DEVICE
  uint off;          // FD_INODE
  short major;       // FD_DEVICE
};
```



### 2. filestat 函数

```C
// Get metadata about file f.
// addr is a user virtual address, pointing to a struct stat.
int
filestat(struct file *f, uint64 addr)
{
  struct proc *p = myproc();
  struct stat st;
  
  if(f->type == FD_INODE || f->type == FD_DEVICE){
    ilock(f->ip);
    stati(f->ip, &st);
    iunlock(f->ip);
    // Copy from kernel to user.
    // Copy len bytes from src to virtual address dstva in a given page table.
    // Return 0 on success, -1 on error.
    //// int copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len)
    if(copyout(p->pagetable, addr, (char *)&st, sizeof(st)) < 0)
      return -1;
    return 0;
  }
  return -1;
}
```



### 3. filewrite 函数

```C++
// Write to file f.
// addr is a user virtual address.
int
filewrite(struct file *f, uint64 addr, int n)
{
  int r, ret = 0;

  if(f->writable == 0)
    return -1;

  if(f->type == FD_PIPE){
    ret = pipewrite(f->pipe, addr, n);
  } else if(f->type == FD_DEVICE){
    if(f->major < 0 || f->major >= NDEV || !devsw[f->major].write)
      return -1;
    ret = devsw[f->major].write(1, addr, n);
  } else if(f->type == FD_INODE){
    // write a few blocks at a time to avoid exceeding
    // the maximum log transaction size, including
    // i-node, indirect block, allocation blocks,
    // and 2 blocks of slop for non-aligned writes.
    // this really belongs lower down, since writei()
    // might be writing a device like the console.

    // MAXOPBLOCKS 的作用？  10
    // 可能是一个经验判断的预估值，尤其是最后的 /2操作，应该是按照经验判断的
    int max = ((MAXOPBLOCKS-1-1-2) / 2) * BSIZE;
    int i = 0;
    while(i < n){
      int n1 = n - i;
      if(n1 > max)
        n1 = max;

      begin_op();
      ilock(f->ip);
      if ((r = writei(f->ip, 1, addr + i, f->off, n1)) > 0)
        f->off += r;
      iunlock(f->ip);
      end_op();

      if(r != n1){
        // error from writei
        break;
      }
      i += r;
    }
    ret = (i == n ? n : -1);
  } else {
    panic("filewrite");
  }

  return ret;
}
```



