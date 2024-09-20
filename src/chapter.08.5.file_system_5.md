# 8.5 Directory

```C
// Directory is a file containing a sequence of dirent structures.
#define DIRSIZ 14

struct dirent {
  ushort inum;    // 文件对应的inode节点ID
  char name[DIRSIZ];
};
```



## 1. dirlookup 函数

**疑问：**

​       1. 函数内部，是如何通过 dp的inode指针，以及一个文件名，来找到这个文件对应的信息，比如inode，还有off?

```Plain
inode 函数是通过iget 来返回的，也就是先通过name，找到对应的dirent结构信息，然后才能获取里面保存到的inum信息，最后再将实际的inode指针返回回来，off就是 在buff中寻找dirent数组中的对应元素
```

2. dirent 目录结构是保存在哪里的？缺少一个整体的布局

​	 保存在最底层磁盘中的buf的 data 结构中

3. Inode 结构是保存在哪里？

 	放在一个全局的itable列表中，保存在内存中



```C
// Look for a directory entry in a directory.
// If found, set *poff to byte offset of entry.
struct inode*
dirlookup(struct inode *dp, char *name, uint *poff)
{
  uint off, inum;
  struct dirent de;

  // 必须是DIR才可以，因为下面会按照目录的方式来遍历磁盘中的数据
  if(dp->type != T_DIR)
    panic("dirlookup not DIR");

  // 根据里面的遍历方式，我可以推测，buf里面的dirent 结构是以数组的形式排列的，这样才可以
  // 按顺序递增
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
      // 所有获取 inode指针的方式，都是要通过inum 的方式才可以
      return iget(dp->dev, inum);
    }
  }

  return 0;
}
```



