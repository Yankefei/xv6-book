# 8.6 目录相关

**directory entry ！= directory**

**Directory entry 也有可能是一个普通的文件名，也有可能是是一个目录**



## 1. 关键函数解析：

### 1. nameiparent 函数

作用： 返回 path 的最后一个路径元素的 parent 的inode. 然后将最后一个path元素填充到name中。



### 2. namei 函数 

作用： 返回 path 的元素所在的inode节点



### 3. namex 函数

作用：如果 nameiparent  != 0, 那么 就是nameiparent 函数逻辑

 内部循环会停止一级返回，返回的inode是

​           如果 nameiparent == 0, 那么就是 namei 函数逻辑， name不会返回任何东西

```C
// Look up and return the inode for a path name.
// If parent != 0, return the inode for the parent and copy the final
// path element into name, which must have room for DIRSIZ bytes.
// Must be called inside a transaction since it calls iput().

static struct inode*
namex(char *path, int nameiparent, char *name)
{
  struct inode *ip, *next;

  // 从cwd 获取inode, cwd 表示当前目录，cwd的更新在于：chdir, 简称 cd
  // 注意： ip会在下面被next更新！
  if(*path == '/')
    ip = iget(ROOTDEV, ROOTINO);
  else
    ip = idup(myproc()->cwd);

  while((path = skipelem(path, name)) != 0){ // Copy the next path element from path into name.
    ilock(ip);
    if(ip->type != T_DIR){
      iunlockput(ip);
      return 0;
    }
    if(nameiparent && *path == '\0'){  // 这里是nameiparent 正常返回的位置
      // Stop one level early.
      iunlock(ip);
      return ip;
    }
    if((next = dirlookup(ip, name, 0)) == 0){  // 在当前目录中寻找
      iunlockput(ip);
      return 0;
    }
    iunlockput(ip);
    ip = next;
  }
  if(nameiparent){
    iput(ip);
    return 0;
  }
  return ip;
}
```



```Plain
没太懂：

The file system uses struct inode reference counts as a kind of shared lock that can be held 
by multiple processes, in order to avoid deadlocks that would occur if the code used ordinary locks. 
For example, the loop in namex (kernel/fs.c:652) locks the directory named by each pathname 
component in turn. However, namex must release each lock at the end of the loop, since if it held 
multiple locks it could deadlock with itself if the pathname included a dot (e.g., a/./b). It might 
also deadlock with a concurrent lookup involving the directory and ... As Chapter 8 explains, 
the solution is for the loop to carry the directory inode over to the next iteration with its reference 
count incremented, but not locked.

// 下面是如何规避死锁的
This concurrency introduces some challenges. For example, while one kernel thread is looking 
up a pathname another kernel thread may be changing the directory tree by unlinking a directory. 
A potential risk is that a lookup may be searching a directory that has been deleted by another 
kernel thread and its blocks have been re-used for another directory or file. 
Xv6 avoids such races. For example, when executing dirlookup in namex, the lookup thread 
holds the lock on the directory and dirlookup returns an inode that was obtained using iget. 
iget increases the reference count of the inode. Only after receiving the inode from dirlookup 
does namex release the lock on the directory. Now another thread may unlink the inode from the 
directory but xv6 will not delete the inode yet, because the reference count of the inode is still 
larger than zero. 
Another risk is deadlock. For example, next points to the same inode as ip when looking 
up ".". Locking next before releasing the lock on ip would result in a deadlock. To avoid this 
deadlock, namex unlocks the directory before obtaining a lock on next. Here again we see why 
the separation between iget and ilock is important. 
```



### 4. dirlookup 函数

经过实际的gdb过程，发现下面的for循环中，off会直接从0，循环到 1024，即使后面的 de.name 为空，也会持续循环查找，

第一次 sys_open 函数，打开console 的时候，就是因为没有找到，所以失败了

失败的调用链： sys_open -> namei  -> namex  ->  dirlookup.

```C++
// Look for a directory entry in a directory.
// If found, set *poff to byte offset of entry.
struct inode*
dirlookup(struct inode *dp, char *name, uint *poff)
{
  uint off, inum;
  struct dirent de;

  if(dp->type != T_DIR)
    panic("dirlookup not DIR");

  for(off = 0; off < dp->size; off += sizeof(de)){
    if(readi(dp, 0, (uint64)&de, off, sizeof(de)) != sizeof(de))
      panic("dirlookup read");
    if(de.inum == 0)
      continue;
    if(namecmp(name, de.name) == 0){
      // entry matches path element
      if(poff)
        *poff = off;
      inum = de.inum;
      return iget(dp->dev, inum);
    }
  }

  return 0;
}
```



