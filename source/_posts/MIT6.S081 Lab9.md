---
title: MIT6.S081 Lab9 FileSystem
date: 2021-11-05 12:00:00
tags: 6.S081
---

## Large files

这个实验要求扩展xv6文件大小，从268个block（12direct * 1+1single indirect * 256）扩展到65803个block（11derect * 1+1single indirect * 256+1double indirect * 256 * 256）。

需要看懂bmap函数的作用，bmap接收一个inode和逻辑块号bn，通过寻址把bn映射为物理块号返回。

> inode的addrs大小不变仍为13，direct缩小为11，最后一位改为double indirect，dinode同理。

```c
static uint
bmap(struct inode *ip, uint bn)
{
  uint addr, *a;
  struct buf *bp;

  if(bn < NDIRECT){
    if((addr = ip->addrs[bn]) == 0)
      ip->addrs[bn] = addr = balloc(ip->dev);
    return addr;
  }
  bn -= NDIRECT;

  if(bn < NINDIRECT){
    // Load indirect block, allocating if necessary.
    if((addr = ip->addrs[NDIRECT]) == 0)
      ip->addrs[NDIRECT] = addr = balloc(ip->dev);
    bp = bread(ip->dev, addr);
    a = (uint*)bp->data;
    if((addr = a[bn]) == 0){
      a[bn] = addr = balloc(ip->dev);
      log_write(bp);
    }
    brelse(bp);
    return addr;
  }
  bn -= NINDIRECT;
  if (bn < NDBDIRECT) {
    if ((addr = ip->addrs[NDIRECT + 1]) == 0)
      ip->addrs[NDIRECT + 1] = addr = balloc(ip->dev);
    bp = bread(ip->dev, addr);
    a = (uint*)bp->data;
    if ((addr = a[bn / NINDIRECT]) == 0) {
      a[bn / NINDIRECT] = addr = balloc(ip->dev);
      log_write(bp);
    }
    brelse(bp);
    bp = bread(ip->dev, addr);
    a = (uint*)bp->data;
    if ((addr = a[bn % NINDIRECT]) == 0) {
      a[bn % NINDIRECT] = addr = balloc(ip->dev);
      log_write(bp);
    }
    brelse(bp);
    return addr;
  }
  panic("bmap: out of range");
}
```

> 修改itrunc使得能够释放double indirect的块。

```c

void
itrunc(struct inode *ip)
{
  int i, j;
  struct buf *bp;
  uint *a;

  for(i = 0; i < NDIRECT; i++){
    if(ip->addrs[i]){
      bfree(ip->dev, ip->addrs[i]);
      ip->addrs[i] = 0;
    }
  }

  if(ip->addrs[NDIRECT]){
    bp = bread(ip->dev, ip->addrs[NDIRECT]);
    a = (uint*)bp->data;
    for(j = 0; j < NINDIRECT; j++){
      if(a[j])
        bfree(ip->dev, a[j]);
    }
    brelse(bp);
    bfree(ip->dev, ip->addrs[NDIRECT]);
    ip->addrs[NDIRECT] = 0;
  }
  if (ip->addrs[NDIRECT + 1]) {
    bp = bread(ip->dev, ip->addrs[NDIRECT + 1]);
    a = (uint*)bp->data;
    for(j = 0; j < NINDIRECT; j++) {
      if (a[j]) {
        uint addr, *data;
        struct buf *bbp;
        addr = a[j];
        bbp = bread(ip->dev, addr);
        data = (uint*)bbp->data;
        for (int k = 0; k < NINDIRECT; k++) {
          if (data[k])
            bfree(ip->dev, data[k]);
        }
        brelse(bbp);
        bfree(ip->dev, addr);
      }
    }
    brelse(bp);
    bfree(ip->dev, ip->addrs[NDIRECT + 1]);
    ip->addrs[NDIRECT + 1] = 0;
  }
  ip->size = 0;
  iupdate(ip);
}
```

## Symbolic links

硬链接就是几个不同的direntry拥有相同的inum。软链接是一个特殊的文件，文件内容是一个路径指向所链接的文件。

xv6实现了硬链接，这个lab要求实现软链接。

> 首先实现系统调用symlink来创建一个软链接文件。

```c
uint64
sys_symlink(void) {
  char target[MAXPATH], path[MAXPATH];
  struct inode *ip;
  if (argstr(0, target, MAXPATH) < 0 || argstr(1, path, MAXPATH) < 0) {
    return -1;
  }
  begin_op();

  if ((ip = create(path, T_SYMLINK, 0, 0)) == 0) {
    end_op();
    return -1;
  }
  writei(ip, 0, (uint64)target, 0, MAXPATH);
  iunlockput(ip);
  end_op();

  return 0;
}
```

> 修改系统调用open用来打开软链接文件。如果O_NOFOLLOW则直接返回这个软链接的inode。
>
> 否则返回它指向的文件的inode，如果指向的文件也是一个软链接，继续打开软链接指向的文件直到指向一个inode或形成环。
>
> 这里需要注意iunlock和iput的区别。iget/iput用于增加/减少ip的ref，用于表示inode cache中这个inode有多少个指针正在指向它，如果没有一个进程指向这个inode，则可以从cache中释放。
>
> ilock/iunlock用于给inode加锁保证线程同步。
>
> 一般的调用操作顺序是iget->ilock->iunlock->iput.在open系统调用中，如果成功找到一个ip，返回时只需要到iunlock即可，不需要iput，因为open文件表示这个进程正在使用这个inode。直到close才能iput。
>
> 另外注意有的函数中间会执行iget/ilock，例如namei成功调用时返回的ip是已经经过iget操作的。
>
> 任何不需要的inode（open操作失败，或者递归打开软链接）都应该iunlockput。

```c
uint64
sys_open(void)
{
  char path[MAXPATH];
  int fd, omode;
  struct file *f;
  struct inode *ip;
  int n;

  if((n = argstr(0, path, MAXPATH)) < 0 || argint(1, &omode) < 0)
    return -1;

  begin_op();

  if(omode & O_CREATE){
    ip = create(path, T_FILE, 0, 0);
    if(ip == 0){
      end_op();
      return -1;
    }
  } else {
    if((ip = namei(path)) == 0){
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
  int depth = 0;
  if (ip->type == T_SYMLINK && !(omode&O_NOFOLLOW)) {
    while (ip->type == T_SYMLINK && depth < 10) {
      readi(ip, 0, (uint64)path, 0, MAXPATH);
      iunlockput(ip);
      if ((ip = namei(path)) == 0) {
        end_op();
        return -1;
      }
      ilock(ip);
      depth++;
    }
    if (depth == 10) {
      iunlockput(ip);
      end_op();
      return -1;
    }
  }
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
    itrunc(ip);
  }

  iunlock(ip);
  end_op();

  return fd;
}
```

## 结果

![](/images/1.png)
