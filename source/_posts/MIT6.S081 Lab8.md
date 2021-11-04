---
title: MIT6.S081 Lab8 locks
date: 2021-10-14 12:00:00
tags: 6.S081
---

## Memory allocator

acquire一个spinlock的时候，如果锁已被占用线程会阻塞，这种锁的争用减弱了系统的并行度，减少了操作系统的效率。执行kalloctest，可以看到

```text
$ kalloctest
start test1
test1 results:
--- lock kmem/bcache stats
lock: kmem: #fetch-and-add 83375 #acquire() 433015
lock: bcache: #fetch-and-add 0 #acquire() 1260
--- top 5 contended locks:
lock: kmem: #fetch-and-add 83375 #acquire() 433015
lock: proc: #fetch-and-add 23737 #acquire() 130718
lock: virtio_disk: #fetch-and-add 11159 #acquire() 114
lock: proc: #fetch-and-add 5937 #acquire() 130786
lock: proc: #fetch-and-add 4080 #acquire() 130786
tot= 83375
test1 FAIL
```

acquire阻塞时对kmem锁的询问达到了83375次。这个lab要求重写分配物理内存的代码，减少锁的争用。

思路是每个cpu分配一个freelist和一个锁，每次cpu执行kalloc或kfree时访问对应的cpu锁，这样cpu之间并行分配内存时就不会争用锁。

另外，当某个cpu的freelist为空时要从别的cpu的freelist里“偷取”可用的物理内存，保证这个过程不会发生死锁。

最开始的想法是平分每个物理页面，根据页面index来分配给对应的cpu，free时根据index还给对应的cpu的freelist。这样做结果也是正确的，但问题是NCPU是cpu的最大值固定为8，实际使用的cpu数不一定为8，make qemu时用了3个hart。

实际上不用这么做，因为只有cpu0会执行kinit以及freerange，一开始把所有物理内存分给cpu0即可。这样别的cpu执行kalloc时会从cpu0“偷取”物理内存，kfree时直接还给这个cpu的freelist，即谁分配就退还给谁。而且不存在的cpu对应的freelist一直为空。

> 数据结构和kinit

```c
struct {
  struct spinlock lock;
  struct run *freelist;
} kmem[NCPU];

void
kinit()
{
  for (int i = 0; i < NCPU; i++) {
    initlock(&kmem[i].lock, "kmem");
  }
  freerange(end, (void*)PHYSTOP);
}
```

> kalloc时先获取对应cpu的锁，如果freelist为空，则循环所有cpu偷取可用内存，这一步也需要加锁。

```c
void *
kalloc(void)
{
  struct run *r;
  int c;
  push_off();
  c = cpuid();
  pop_off();
  acquire(&kmem[c].lock);
  r = kmem[c].freelist;
  if(r) {
    kmem[c].freelist = r->next;
  }
  else {
    for (int i = 0; i < NCPU; i++) {
      if (i == c) continue;
      acquire(&kmem[i].lock);
      if (kmem[i].freelist) {
        r = kmem[i].freelist;
        kmem[i].freelist = r->next;
        release(&kmem[i].lock);
        break;
      }
      release(&kmem[i].lock);
    }
  }
  if(r)
    memset((char*)r, 5, PGSIZE); // fill with junk
  release(&kmem[c].lock);
  return (void*)r;
}
```

在freelist不为空的情况下不会产生锁的争用，因为只用到了本cpu的锁；但是如果需要偷取内存，就可能产生锁的争用。但由于“偷取时对应cpu正好需要kalloc”这件事概率不高，并且acquire和release之间距离比较近，所以产生争用的可能性不大，因此kalloctest时输出的tot为0.

如果在偷取时acquire和release之间加上 for (int i = 0;i < 1000000; i++);可以看到结果是产生了争用的。

![img](https://pic2.zhimg.com/80/v2-89c3ed41f875fec6da034537d125bda5_720w.jpg)

> kfree把内存还给当前cpu。

```c
void
kfree(void *pa)
{
  struct run *r;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  // Fill with junk to catch dangling refs.
  memset(pa, 1, PGSIZE);

  r = (struct run*)pa;
  push_off();
  int c = cpuid();
  pop_off();
  acquire(&kmem[c].lock);
  r->next = kmem[c].freelist;
  kmem[c].freelist = r;
  release(&kmem[c].lock);
}
```

## Buffer cache

程序访问磁盘时首先会通过bget查看块缓存bcache中有没有所需的block，如果有就无需访问磁盘，仅仅产生内存间数据交换。如果没有需要访问磁盘并且把读取到的块代替bcache的某个buf（通过LRU）。

xv6用了一个大锁bcache.lock保证bcache同时只有一个线程使用，毫无疑问这会导致大量的争用，这个lab要求修改bio.c的代码减少争用。

思路化大锁为小锁，创建多个bucket，根据block的index把对应块hash到对应bucket中。这样每次bget时只需要获取对应bucket的锁，假如多个cpu同时进入bget，但blockno hash到了不同的bucket，这种情况是可以并行的。

hints要求替换buf时用时间戳来实现LRU算法。

> bucket数最好是个质数减少hash冲突，总的buf数就是NBUCKET*BUCKETSZ。xv6原来的buf数是30，我这里增加了一点来尽量减少争用。

```c
#define NBUCKET 13
#define BUCKETSZ 8

extern uint ticks;
struct bucket {
  struct spinlock lock;
  struct buf bufs[BUCKETSZ];
};
```

> binit初始化所有bucket的锁

```c
void
binit(void)
{
  initlock(&bcache.lock, "bcache");
  for (int i = 0; i < NBUCKET; i++) {
    initlock(&bcache.buckets[i].lock, "bcache.bucket");
    for (int j = 0; j < BUCKETSZ; j++)
      initsleeplock(&bcache.buckets[i].bufs[j].lock, "buffer");
  }
}
```

> bget首先根据blockno获取对应bucket的锁，如果cache中存在这个块，就更新timestamp，并且refcnt++表示当前多了一个线程对这个buf感兴趣。
> 如果没有对应块，从bucket中选取一个timestamp最小的（LRU）来替换这个块。

```c
static struct buf*
bget(uint dev, uint blockno)
{
  int k = blockno % NBUCKET;
  acquire(&bcache.buckets[k].lock);
  struct bucket* bkt = &bcache.buckets[k];
  struct buf* b;
  for (b = &bkt->bufs[0]; b < (struct buf*)&bkt->bufs[0] + BUCKETSZ; b++) {
    if (b->dev == dev && b->blockno == blockno) {
      b->timestamp = ticks; 
      b->refcnt++;
      release(&bcache.buckets[k].lock);
      acquiresleep(&b->lock);
      return b;
    }
  }
  uint last = -1;
  struct buf* ret = b;
  for (b = &bkt->bufs[0]; b < (struct buf*)&bkt->bufs[0] + BUCKETSZ; b++) {
    if (b->timestamp < last && b->refcnt == 0) {
      last = b->timestamp;
      ret = b;  
    }
  }
  ret->dev = dev;
  ret->blockno = blockno;
  ret->valid = 0;
  ret->timestamp = ticks;
  ret->refcnt = 1;
  release(&bcache.buckets[k].lock);
  acquiresleep(&ret->lock);
  return ret;
  panic("bget: no buffers");
}
```

> brelse不用再release大锁。
> bpin和bunpin把锁改成对应bucket的锁即可。

```c
void
brelse(struct buf *b)
{
  if(!holdingsleep(&b->lock))
    panic("brelse");
  b->refcnt--;
  releasesleep(&b->lock);
}

void
bpin(struct buf *b) {
  int k = b->blockno % NBUCKET;
  acquire(&bcache.buckets[k].lock);
  b->refcnt++;
  release(&bcache.buckets[k].lock);
}

void
bunpin(struct buf *b) {
  int k = b->blockno % NBUCKET;
  acquire(&bcache.buckets[k].lock);
  b->refcnt--;
  release(&bcache.buckets[k].lock);
}
```



bcache减少争用和kmem不一样，内存页面之间是没有区别的，因此kmem可以每个cpu一个锁，如果不考虑“偷取”，kmem可以达到0争用。但是bcache是所有cpu共享的，并且每个buf内容不一样不能随意分配，所以不可能做到0争用。

但是由于bcachetest中，不同进程不会访问同一磁盘块，并且不会同时miss cache，所以争用发生的可能性不大，结果显示tot=0.（实际上这个lab只要求tot<500）

![img](https://pic4.zhimg.com/80/v2-79bc8d636b1fd85126eae88f8c6d2e73_720w.jpg)

这里实现LRU的方法是循环遍历每个buf查看timestamp，这样的效率还不如初始链表实现的做法。hints里要求我们使用timestamp且要删除初始链表，所以我觉得lab的要求其实是要实现bucket之间的“偷取”。当当前bucket需要找替代块的时候，应该要从所有的buf中找到timestamp最小的buf替换，而不是仅仅在当前bucket找。但是我的做法已经能通过测试，就没有改进了。

## 结果

![img](https://pic3.zhimg.com/80/v2-4983007f0ba5e2221c5783a478477476_720w.jpg)

## 心得

第2个lab测试前最好先make clean。