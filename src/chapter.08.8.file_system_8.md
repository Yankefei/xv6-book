

# 8.8 file system call

## 1. 列表：

![](/home/uto_ykf/work_os/xv6-book/src/images/file_system_5.png)

系统调用的编号：

```C
// System call numbers
#define SYS_fork    1
#define SYS_exit    2
#define SYS_wait    3
#define SYS_pipe    4
#define SYS_read    5
#define SYS_kill    6
#define SYS_exec    7
#define SYS_fstat   8
#define SYS_chdir   9
#define SYS_dup    10
#define SYS_getpid 11
#define SYS_sbrk   12
#define SYS_sleep  13
#define SYS_uptime 14
#define SYS_open   15
#define SYS_write  16
#define SYS_mknod  17
#define SYS_unlink 18
#define SYS_link   19
#define SYS_mkdir  20
#define SYS_close  21
```



## 2. 关键函数介绍：

### 1. sys_mknod 函数

init.c 函数，必须要执行的函数，因为最开始的 sys_open 函数会执行失败

```C++
  if(open("console", O_RDWR) < 0){
    mknod("console", CONSOLE, 0);
    ...
```

![](/home/uto_ykf/work_os/xv6-book/src/images/file_system_6.png)

作用是什么？

用于在根目录中，创建一个设备，创建好设备后，设备ID **major**可以传一个如**CONSOLE（1）**，就可以用设备的ID，来初始化下面的回调函数，用来处理设备的一些交互操作

```C++
// map major device number to device functions.
struct devsw {
  int (*read)(int, uint64, int);
  int (*write)(int, uint64, int);
};
```





### 2. sys_open 函数

```C
uint64
sys_open(void)
{
  char path[MAXPATH];
  int fd, omode;
  struct file *f;
  struct inode *ip;
  int n;

  argint(1, &omode);
  if((n = argstr(0, path, MAXPATH)) < 0)
    return -1;

  begin_op();

  // 只有 CREATE模式下，才会开始 create
  if(omode & O_CREATE){
    ip = create(path, T_FILE, 0, 0);
    if(ip == 0){
      end_op();
      return -1;
    }
  } else {
    if((ip = namei(path)) == 0){
      // 注意：
      // init.c 函数第一次启动的时候，会在这个地方失败
      end_op();
      return -1;
    }
    ilock(ip);
    if(ip->type == T_DIR && omode != O_RDONLY){
      iunlockput(ip);
      end_op();
      return -1;
    }
  }

  if(ip->type == T_DEVICE && (ip->major < 0 || ip->major >= NDEV)){
    iunlockput(ip);
    end_op();
    return -1;
  }

  // 首先调用  filealloc 函数 从固定file数组中，返回一个为空的file指针，数组大小 100
  // 然后调用  fdalloc 函数   会返回从0开始递增的数字，表示打开的的文件 ofile, 每个进程限制为16
  if((f = filealloc()) == 0 || (fd = fdalloc(f)) < 0){
    if(f)
      fileclose(f);
    iunlockput(ip);
    end_op();
    return -1;
  }

  if(ip->type == T_DEVICE){
    f->type = FD_DEVICE;
    f->major = ip->major;
  } else {
    f->type = FD_INODE;
    f->off = 0;
  }
  f->ip = ip;
  f->readable = !(omode & O_WRONLY);
  f->writable = (omode & O_WRONLY) || (omode & O_RDWR);

  if((omode & O_TRUNC) && ip->type == T_FILE){
    // O_TRUNC │若文件存在，则长度被截为0，属性不变
    itrunc(ip);
  }

  iunlock(ip);
  end_op();

  return fd;
}
```



**问题**：sys_open 为什么第一次在init.c 中调用，会失败? 而且需要先调用**mknod** 才可以？

通过gdb 查看，也很简单，因为 init.c 的第一次open, 执行的mod参数为 **O_RDWR**，那么在 sys_open 里面，不会直接调用create, 而是通过namei 来查找 “console“ 设备名，一开始，如果没有msys_mknod，在根目录肯定无法查找到，所以会直接报错。

```C++
  // 只有 CREATE模式下，才会开始 create
  if(omode & O_CREATE){
    ip = create(path, T_FILE, 0, 0);
    ...
  } else {
    if((ip = namei(path)) == 0){
    ...
    }
```



### 3. sys_link 函数

**一句话描述**：作用是 在旧的文件的基础上，创建一个新的名称，两者共用一个node

它的第一个参数（旧文件名），第二个参数（新文件名）

```C
// Create the path new as a link to the same inode as old.( 只能在同一个device上创建 link? ok! dp->dev != ip->dev)
uint64
sys_link(void)
{
  char name[DIRSIZ], new[MAXPATH], old[MAXPATH];
  struct inode *dp, *ip;

  if(argstr(0, old, MAXPATH) < 0 || argstr(1, new, MAXPATH) < 0)
    return -1;

  begin_op();
  if((ip = namei(old)) == 0){
    end_op();
    return -1;
  }

  ilock(ip);
  if(ip->type == T_DIR){
    iunlockput(ip);
    end_op();
    return -1;
  }

  ip->nlink++;
  iupdate(ip);
  iunlock(ip);

  if((dp = nameiparent(new, name)) == 0)
    goto bad;
  ilock(dp);
  if(dp->dev != ip->dev || dirlink(dp, name, ip->inum) < 0){ // new directory entry 指向 旧的inode(ip->inum)
    iunlockput(dp);
    goto bad;
  }
  iunlockput(dp);
  iput(ip);

  end_op();

  return 0;

bad:
  ilock(ip);
  ip->nlink--;
  iupdate(ip);
  iunlockput(ip);
  end_op();
  return -1;
}
```



### 4. sys_unlink 函数

**一句话描述**： 作用是删除文件, 但如果是 sys_link 出的文件，那么将不是删除，而是将nlink 减1

删除文件，以及文件夹？可以删除文件夹，但必须是空的文件夹

```C
uint64
sys_unlink(void)
{
  struct inode *ip, *dp;
  struct dirent de;
  char name[DIRSIZ], path[MAXPATH];
  uint off;

  if(argstr(0, path, MAXPATH) < 0)
    return -1;

  begin_op();
  if((dp = nameiparent(path, name)) == 0){  // dp -> 父目录
    end_op();
    return -1;
  }

  ilock(dp);

  // Cannot unlink "." or "..".
  if(namecmp(name, ".") == 0 || namecmp(name, "..") == 0)
    goto bad;

  // 获取到的off信息很关键！
  // 
  if((ip = dirlookup(dp, name, &off)) == 0) // 寻找name对应的inode(ip)
    goto bad;
  ilock(ip);

  if(ip->nlink < 1)
    panic("unlink: nlink < 1");
  if(ip->type == T_DIR && !isdirempty(ip)){  // 如果最终是目录，且不为空，是不能unlink的
    iunlockput(ip);
    goto bad;
  }

  memset(&de, 0, sizeof(de));
  // 写入的是什么？
  // 只是将一个空的de结构体，填充到 父目录 dp指针所对应的 buf->data中，表示删除对应的信息
  // 所以你会看到在代码中，如果遇到了de.inum的时候，如果是0，就需要 continue
  if(writei(dp, 0, (uint64)&de, off, sizeof(de)) != sizeof(de))
    panic("unlink: writei");
  //
  if(ip->type == T_DIR){
    // 这里的 dp->nlink --; 是因为每一次创建一个dir的时候，都会在父目录中增加一个nlink
    // 所以删除的时候，也需要这样处理
    dp->nlink--;  
    iupdate(dp);
  }
  iunlockput(dp);

  ip->nlink--;
  iupdate(ip);
  iunlockput(ip);

  end_op();

  return 0;

bad:
  iunlockput(dp);
  end_op();
  return -1;
}
```



### 5. create 函数

sys_mknod 和  sys_open 都会调用进来

```C
static struct inode*
create(char *path, short type, short major, short minor)
{
  struct inode *ip, *dp;
  char name[DIRSIZ];

  if((dp = nameiparent(path, name)) == 0)
    return 0;

  ilock(dp);

  if((ip = dirlookup(dp, name, 0)) != 0){
    iunlockput(dp);
    ilock(ip);
    if(type == T_FILE && (ip->type == T_FILE || ip->type == T_DEVICE))
      return ip;
    iunlockput(ip);
    return 0;
  }

  if((ip = ialloc(dp->dev, type)) == 0){
    iunlockput(dp);
    return 0;
  }

  // ip 是新创建的inode节点，所以这里加锁之后，后面都没有进行iunlockput的解锁操作？
  // 是否会造成死锁？？
  // 目前看是不会的，因为在返回之后，会有地方进行解锁操作
  ilock(ip);   
  ip->major = major;
  ip->minor = minor;
  ip->nlink = 1;
  iupdate(ip);

  if(type == T_DIR){  // Create . and .. entries.
    // No ip->nlink++ for ".": avoid cyclic ref count.
    
    // 关键的地方！！！
    // 创建dirlink的时候， . 的inum 是 ip的
    //                而 .. 的inum 是 dp的
    // 所以，当 cd . 或者 cd .. 的时候， 必然也会访问当对应的inode 里面
    // 因为虽然cd 的参数是文件名，但实际还是根据文件名来查找对应的 inode节点
    if(dirlink(ip, ".", ip->inum) < 0 || dirlink(ip, "..", dp->inum) < 0)
      goto fail;
  }

  // 在dp父目录中，新增加一个目录节点
  if(dirlink(dp, name, ip->inum) < 0)
    goto fail;

  if(type == T_DIR){
    // now that success is guaranteed:
    // 是的，每创建一个目录时，都会增加一个nlink， 就是为了在子目录访问 .. 的时候可以成功
    // 因为本质上每一个目录的 .. ，inum指向的都是父节点的inu.
    dp->nlink++;  // for ".."
    iupdate(dp);
  }

  iunlockput(dp);

  return ip;

 fail:
  // something went wrong. de-allocate ip.
  ip->nlink = 0;
  iupdate(ip);
  iunlockput(ip);
  iunlockput(dp);
  return 0;
}
```



### 6. isdirempty 函数

```C
// Is the directory dp empty except for "." and ".." ?
static int
isdirempty(struct inode *dp)
{
  int off;
  struct dirent de;

  // 为什么要偏移 2个de, 是为了去掉 .. 和 .
  for(off=2*sizeof(de); off<dp->size; off+=sizeof(de)){
    if(readi(dp, 0, (uint64)&de, off, sizeof(de)) != sizeof(de))
      panic("isdirempty: readi");
    if(de.inum != 0)
      return 0;
  }
  return 1;
}
```



问题：

1.  cd ..   的过程是如何完成的？

```C
首先，调用 sys_chdir 的函数，参数数path,
然后通过 namei 节点，获取都对应path 的inode节点, 假设变量为ip，内部调用的是 dirlookup 函数，而且访问 .. 也是通过name的匹配完成的。
调用 iput，来将当前p->pwd的inode 的引用释放，并重置 cwd为ip.
返回
```



