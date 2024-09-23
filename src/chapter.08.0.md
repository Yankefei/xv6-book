# 第八章：文件系统篇





risc-v 操作系统里面的文件系统，分为下面这些逻辑分层

![](/home/uto_ykf/work_os/xv6-book/src/images/file_system_1.png)

在磁盘中，block块的分布如下：

![](/home/uto_ykf/work_os/xv6-book/src/images/file_system_2.png)

1.  Disk                     磁盘驱动.  物理设备
2.  Buffer Cache        用户读写磁盘过程中设置的缓存
3.  logging                 

​            第一：用于将多次的读写过程，进行事务化，统一commit, 减少磁盘IO的操作次数

​            第二：也有利于事务失败后，对数据进行恢复的操作

1. Inode            主要是磁盘中的dinode结构, 以及内存中的inode结构
2. Directory       目录，信息，也会保存在磁盘最后面的data block结构中
3. Pathname    由一层层的目录所组成的文件目录结构，是对上一个 directory的封装
4. File descriptor

