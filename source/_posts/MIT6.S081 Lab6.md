---
title: MIT6.S081 Lab6 cow
date: 2021-10-09 12:00:00
tags: 6.S081
---

## **COW fork**

fork的时候xv6会为子进程立刻分配所需的物理内存，并把父进程的用户内存复制过去。这样造成了时空上的浪费。写时复制指的是fork时不立刻分配物理内存，而是父子进程共用父进程的地址空间。

实现这个功能主要要实现3个部分：

1.fork时将子进程地址空间逐页map到和父进程一样的地址空间，并且把父子进程页表项设置为不可写。

2.当某个进程要写一个COW map的不可写页时，此时这个进程就不能和其他进程共享这一页了，需要另外开辟新的物理内存并map，这一部分通过处理page fault实现。

3.注意物理页free的时机，如果某一物理页被多个进程共享，那么即使其中某一个进程不再使用它，它也不能被free。当最后一个使用它的进程free这一页时才真正释放。

第一个部分在uvmcopy函数中实现，fork会调用uvmcopy来把父进程地址空间复制给子进程：

> 在原uvmcopy的基础上删除了分配内存的过程。逐页地，把父进程的物理地址map到子进程的页表中去。
> 注意flag的设置，要把PTE_W位清除，并且set了flag中的保留位，用于指明这是一个由于COW fork而导致无法write的页。这是有必要的，因为操作系统不能保证用户程序不是恶意的，用户程序可能写一个它本不该写的页，如果不判断是否为COW map的页，到pagefault处理完用户程序就变得能够写这个页，这是不安全的。pte的flag如下图。
> incref用于增加pa对应页的ref count，下面会说

![img](https://pic2.zhimg.com/80/v2-707f039c178ba7db7fa04330baa66541_720w.jpg)

```c
int
uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
  pte_t *pte;
  uint64 pa, i;
  uint flags;

  for(i = 0; i < sz; i += PGSIZE){
    if((pte = walk(old, i, 0)) == 0)
      panic("uvmcopy: pte should exist");
    if((*pte & PTE_V) == 0)
      panic("uvmcopy: page not present");
    pa = PTE2PA(*pte);
    flags = PTE_FLAGS(*pte);
    flags = (flags & ~PTE_W) | PTE_RESERVE;
    *pte = (~(*pte ^ ~PTE_W)) | PTE_RESERVE;
    if (mappages(new, i, PGSIZE, (uint64)pa, flags) != 0) {
      goto err;
    }
    incref(pa);
  }
  return 0;

  err:
   uvmunmap(new, 0, i / PGSIZE, 1);
   return -1;
}
```



第二部分实现物理内存的分配，有两种情况会导致写共享页：用户程序主动写和内核缓存复制到用户空间（如read系统调用）。第一种情况需要在usertrap中处理pagefault：

> 首先判断用户要写的这一页是否为COWmap页，如果不是则杀死进程，然后分配物理内存并map

```c
... 
else if (r_scause() == 15) {
      uint64 va = r_stval();
      pte_t *pte = walk(p->pagetable, va, 0);
      if (!(PTE_FLAGS(*pte)&PTE_RESERVE)) {
        p->killed = 1;
      }
      else {
        va = PGROUNDDOWN(va);
        uint64 ka = (uint64)kalloc();
        if (ka == 0) {
          p->killed = 1;
        }
        else {
          uint flags = PTE_FLAGS(*pte);
          flags = flags & ~PTE_RESERVE;
          uint64 pa = walkaddr(p->pagetable, va);
          memmove((void*)ka, (void*)pa, PGSIZE);
          uvmunmap(p->pagetable, va, 1, 1);
          mappages(p->pagetable, va, 1, ka, flags | PTE_W); 
        }
      }
  }
...
```

第二种情况在copyout中实现，和上面差不多：

```c
int
copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len)
{
  uint64 n, va0, pa0;
  if (dstva > MAXVA - len) return -1; 
  while(len > 0){
    va0 = PGROUNDDOWN(dstva);
    pte_t* pte = walk(pagetable, va0, 0);
    if (pte == 0) return -1;
    if (!(PTE_FLAGS(*pte) & PTE_W)) {
      if (!(PTE_FLAGS(*pte) & PTE_RESERVE)) {
        return -1;
      }
      uint64 va = va0;
      uint64 ka = (uint64)kalloc();
      if (ka == 0) {
        return -1;
      }
      else {
        uint flags = PTE_FLAGS(*pte);
        flags = flags & ~PTE_RESERVE;
        uint64 pa = walkaddr(pagetable, va);
        if (pa == 0) return -1;
        memmove((void*)ka, (void*)pa, PGSIZE);
        uvmunmap(pagetable, va, 1, 1);
        mappages(pagetable, va, 1, ka, flags | PTE_W); 
      }
    }
    pa0 = walkaddr(pagetable, va0);
    if(pa0 == 0)
      return -1;
    n = PGSIZE - (dstva - va0);
    if(n > len)
      n = len;
    memmove((void *)(pa0 + (dstva - va0)), src, n);

    len -= n;
    src += n;
    dstva = va0 + PGSIZE;
  }
  return 0;
}
```

最后是free物理页面的问题，我们需要一个数据结构保存每个页面的引用次数ref count，当一个页面引用次数到达0时才可真正释放，数据结构设置在kalloc.c中：

> 因为最大进程数NPROC为64，所以用1个字节保存引用数足以。每个页面用右移12位的页面序号作为index。

```text
uint8 refcount[PHYSTOP / PGSIZE];
...
void incref(uint64 va) {
  refcount[va/PGSIZE]++;
}
```

> 每次调用kfree时先减引用数，引用数为0再真正释放：

```c
void
kfree(void *pa)
{
  struct run *r;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  // Fill with junk to catch dangling refs.
  if (--refcount[(uint64)pa / PGSIZE] > 0) 
    return ;
  memset(pa, 1, PGSIZE);

  r = (struct run*)pa;

  acquire(&kmem.lock);
  r->next = kmem.freelist;
  kmem.freelist = r;
  release(&kmem.lock);
}
```

> 初始化的时候在freerange把refcount设置为0（这里设置为1再kfree就变成0了）。每次kalloc即初次分配时把refcount设置为1：

```c
void
freerange(void *pa_start, void *pa_end)
{
  char *p;
  p = (char*)PGROUNDUP((uint64)pa_start);
  for(; p + PGSIZE <= (char*)pa_end; p += PGSIZE)
  {
    refcount[(uint64)p / PGSIZE] = 1;
    kfree(p);
  }
}

void *
kalloc(void)
{
  struct run *r;

  acquire(&kmem.lock);
  r = kmem.freelist;
  if(r)
    kmem.freelist = r->next;
  release(&kmem.lock);

  if(r) {
    memset((char*)r, 5, PGSIZE); // fill with junk
    refcount[(uint64)r / PGSIZE] = 1;
  }
  return (void*)r;
}
```

结果：

![img](https://pic4.zhimg.com/80/v2-16198edaae5d2659e976b78ef9dfe693_720w.jpg)

## 心得

debug的时候可以在lab3 vmprint基础上输出一下flag以及对应的refcount查看有没有错误。