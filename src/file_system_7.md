## 八：mkfs

一开始的文件系统，都是在这部分完成的：

```C
#define BSIZE 1024  // block size

// mkfs computes the super block and builds an initial file system. The
// super block describes the disk layout:
struct superblock {
  uint magic;        // Must be FSMAGIC
  uint size;         // Size of file system image (blocks)  2000
  uint nblocks;      // Number of data blocks               1954
  // ninodes 的总数是固定的，后面暂时都不会修改（限制的地方！） 200
  uint ninodes;      // Number of inodes.         会提前记录是有inodes的总数
  uint nlog;         // Number of log blocks            30
  uint logstart;     // Block number of first log block    2
  uint inodestart;   // Block number of first inode block    32
  uint bmapstart;    // Block number of first free map block     45
};


// On-disk inode structure
struct dinode {
  short type;           // File type
  short major;          // Major device number (T_DEVICE only)
  short minor;          // Minor device number (T_DEVICE only)
  short nlink;          // Number of links to inode in file system
  uint size;            // Size of file (bytes)
  uint addrs[NDIRECT+1];   // Data block addresses
};  // 12 + 13 * 4 = 64 byte
```



### 磁盘布局

#### 磁盘block分配的依据：

```C++
#define MAXOPBLOCKS  10  // max # of blocks any FS op writes
#define LOGSIZE      (MAXOPBLOCKS*3)  // max data blocks in on-disk log

#define FSSIZE       2000  // size of file system in blocks
#define BSIZE 1024  // block size

// Inodes per block.
#define IPB           (BSIZE / sizeof(struct dinode))   // 1024 / 64 == 16

#define NINODES 200


// Disk layout:
// [ boot block | sb block | log | inode blocks | free bit map | data blocks ]

// 这个bitmap 所占的block数量，是按照所有blocks的数量，来映射到bit位的，
int nbitmap = FSSIZE/(BSIZE*8) + 1;

// 200 / 16 + 1, 用来表示，预分配的inode的数量，能占用多少个blocks，计算过程也是这个含义
int ninodeblocks = NINODES / IPB + 1;

// 30，三倍于 MAXOPBLOCKS
int nlog = LOGSIZE;
int nmeta;    // Number of meta blocks (boot, sb, nlog, inode, bitmap)
int nblocks;  // Number of data blocks
```



```Plain
 Disk layout:
 total: 2000 block,  per block: 1024
 
inode block 区域的作用：
分配了200个 dinode节点，dinode 的作用，是
 
 也就是下面inode blocks的区域，预先分配了200个dinode(inode)的节点，每个64byte, 一个BLOCK中可以放 18个。
 所以对齐BLOCK后，一共需要消耗 BLOCK为13个 (200 * 64 / 1024 = 12.5)
 注意：内存中的itable中的inode，结构体，预分配，只有50个
 
 [   boot block   |   super block   |   log   |     inode blocks    |
 
         1                  1           30             13             
 
 
 free bit map 的作用：
 需要保证里面每一bit位都能映射覆盖整个dev文件系统的blocks，从它的计算方式也能知道这一点
 用每一bit位的1或者0，来标记这个block是否已经被使用
 在初始化时，data blocks 之前的所有block都已经被标识为已使用，所以无需考虑后面申请出非data block的节点
 
 
 可以通过dinode结构来推测映射一个dinode映射的文件总大小：
 256 + 12 = 268 个data block节点，那么：文件总大小为： 268 * 1024 = 268kB
 也就是说一个文件最多268 kB? yes
 
 一个dinode里面可以映射的block数目为 256 + 12 = 268，那么200个如果全部映射，
 最多需要的是 (268×200)53600 个block,也就是每个文件都是268kB
 最少需要（1* 200）200个, 每个文件是1kB以内。  看起来这里的data blocks的数量 是比较均衡的
 
 
 [   free bit map  |   data blocks    ]
 
        1                 1954
 
```



**虽然data_blocks区域的节点索引，从1954开始的，不过 inum的计数，是从1开始的，根目录的inode， 标号就是1 **



### iappend 函数

​        填充数据到data block区域

​         **文件大小不超过  268kb**

```C
// 向inum对应的实际data_block中按data_block的索引顺序写入数据，这个数据目前都是目录dirent信息，
// 或者是user里面工具程序的二进制文件信息
void
iappend(uint inum, void *xp, int n)
{
  char *p = (char*)xp;
  uint fbn, off, n1;
  struct dinode din;
  char buf[BSIZE];
  uint indirect[NINDIRECT];  //  256
  uint x;

  rinode(inum, &din);
  off = xint(din.size);
  // printf("append inum %d at off %d sz %d\n", inum, off, n);
  while(n > 0){
    fbn = off / BSIZE;   // 映射到dinode 里面addrs的数组索引
    assert(fbn < MAXFILE);  // 文件大小不超过  268kb
    if(fbn < NDIRECT){
      if(xint(din.addrs[fbn]) == 0){
        // 这里的freeblock 就是磁盘最后面 data blocks 的部分的索引序号
        din.addrs[fbn] = xint(freeblock++);
      }
      x = xint(din.addrs[fbn]);  // 可以相关转化
    } else {
      if(xint(din.addrs[NDIRECT]) == 0){
        // 这里将addrs的最后一个元素的地址填上一个完整的 data blocks的索引
        din.addrs[NDIRECT] = xint(freeblock++);
      }
      // 将上面选的 data blocks 的索引内容，拷贝到 indirect 里面
      // 然后每一次循环，都将里面的一个元素赋值为 blocks的索引号
      // 然后写回到 din.addrs[NDIRECT] 对应的磁盘块中
      rsect(xint(din.addrs[NDIRECT]), (char*)indirect);
      if(indirect[fbn - NDIRECT] == 0){
        indirect[fbn - NDIRECT] = xint(freeblock++);
        wsect(xint(din.addrs[NDIRECT]), (char*)indirect);
      }
      // 获取到这个地方对应的 data blocks的索引
      x = xint(indirect[fbn-NDIRECT]);
    }
    n1 = min(n, (fbn + 1) * BSIZE - off);
    rsect(x, buf);
    bcopy(p, buf + off - (fbn * BSIZE), n1); // void bcopy(const void *src, void *dest, size_t n);
    wsect(x, buf);
    n -= n1;
    off += n1;
    p += n1;
  }
  din.size = xint(off); // 更新了文件大小后，再写回去
  winode(inum, &din);
}
```



执行逻辑：

### 填充 data_blocks：

一开始先创建一个root的节点，inode: 为1，类型为目录，表示一个根目录，后面的一些工具程序文件名都会添加到后面，以dinode 这种结构

同时，这些二进制的程序文件，如 rm, mkdir, cat 等，也会添加到data blocks区域，每一个文件都会申请一个inode, 从2开始，类型为

文件。

最后，会对 user目录的数据做一次block的对齐，然后将已经使用的data blocks 信息同步到bitmap中。





## 九： 综合问题：



### 当创建一个文件时，发生了什么？

1. 进入open函数，然后执行begin_op, 开始进入事务
2. 获取到父目录的inode指针 dp，并对 dp 执行ilock
3. 申请一个新的inode指针 ip， 也执行 ilock
4. 更新 ip 里面的字段，比如  major   minor  nlink, 释放 ip的lock
5. 关联ip 的inum 到dp 中，表示目录更新
6. 解锁 dp, 并调用iput 更新dp到 磁盘中
7. 调用 filealloc 申请文件架构体  struct file指针
   1. 加 ftable.lock 的锁
   2. 找一个f->ref为0的，然后将f->ref设置为1
   3. 释放fatable.lock
8. 调用fdalloc 申请一个fd，并更新ip到f的字段中
   1. 从 p->ofile 中找第一个不为0，的元素，返回序号 fd
9. 解锁 ip
10. 执行end_op, 开始正式提交事务



### 当写入一个文件时，发生了什么？

1. 先设定成最多写入的字节数量，默认值和  MAXOPBLOCKS 的块数有关
2. 执行begin_op, 开始进入事务
3. 对f->ip  执行 ilock 加锁
4. 进入调用writei 的逻辑
   1. 调用bmap函数，获取对应文件偏移量的block_id
   2. 调用bread 读取这个block_id 获取到 buf指针  bp
   3. 调用either_copyin， 从用户空间往内核空间的bp->data 里面拷贝数据
   4. 调用log_write 将bp 记录到log.lh 同步里面，用于 事务
   5. 调用 brelse，释放 bp
   6. 调用iupdate(ip), 避免第一次写入时，ip->addr 里面存在初始化过程
5. 对f->ip 解锁
6. 执行end_op, 开始正式提交事务



### 当读取一个文件时，发生了什么？

1. 对f->ip 执行 ilock
2. 调用 readi
   1. 调用bmap函数，获取对应文件偏移量的block_id
   2. 调用bread 读取这个block_id 获取到 buf指针  bp
   3. 调用either_copyout， 从内核空间的bp->data 往用户空间里面拷贝数据
   4. 调用brelse 释放bp
3. 对 f->ip 执行 iunlock



### 当删除这个文件时，发生了什么？

1. 加ftable.lock锁，将f->ref 设置为0
2. 解锁  ftable.lock
3. 执行begin_op, 开始进入事务
4. 执行iput函数
   1. 调用itrunc 函数，清理inode 和 磁盘中的data block区域
   2. 设置 ip->type 为0， 并更新ip 到 磁盘中
   3. 设置 ip->valid 为0
   4. 递减ip->ref 释放内存中的ip指针到数组 itable.inode
5. 执行end_op 开始正式提交事务