

# 8.10 综合问题：


## 1. 执行流相关

### 1. 当创建一个文件时，发生了什么？

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



### 2. 当写入一个文件时，发生了什么？

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



### 3. 当读取一个文件时，发生了什么？

1. 对f->ip 执行 ilock
2. 调用 readi
   1. 调用bmap函数，获取对应文件偏移量的block_id
   2. 调用bread 读取这个block_id 获取到 buf指针  bp
   3. 调用either_copyout， 从内核空间的bp->data 往用户空间里面拷贝数据
   4. 调用brelse 释放bp
3. 对 f->ip 执行 iunlock



### 4. 当删除这个文件时，发生了什么？

1. 加ftable.lock锁，将f->ref 设置为0
2. 解锁  ftable.lock
3. 执行begin_op, 开始进入事务
4. 执行iput函数
   1. 调用itrunc 函数，清理inode 和 磁盘中的data block区域
   2. 设置 ip->type 为0， 并更新ip 到 磁盘中
   3. 设置 ip->valid 为0
   4. 递减ip->ref 释放内存中的ip指针到数组 itable.inode
5. 执行end_op 开始正式提交事务



## 2. 其他

todo