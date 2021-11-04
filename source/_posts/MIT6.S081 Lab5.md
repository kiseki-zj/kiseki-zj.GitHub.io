---
title: MIT6.S081 Lab5 lazy allocation
date: 2021-04-24 12:00:00
tags: 6.S081
---

## 实验前准备

> 阅读xv6 book chapter5，尤其是page fault的部分

## Eliminate allocation from sbrk()

lazy allocation就是在用户程序通过sbrk系统调用申请内存空间时，并不当场给它分配内存，而仅仅是增加p->sz。当用户程序使用申请的内存时会产生page fault而trap到内核态，此时再分配内存。

第一个实验就是把sbrk中分配内存的过程去掉，观察产生的结果：

![img](https://pic2.zhimg.com/80/v2-caf0f6b5fd4f81ed375c9cfd9eb89da9_720w.jpg)

这里去掉了分配内存的部分，增加了proc的sz

![img](https://pic3.zhimg.com/80/v2-0b9bf1ec112ed8cf8508f97fff8fbff2_720w.jpg)

之后运行xv6，执行echo hi会产生page fault。这是因为执行echo hi时系统会使用sbrk分配一些内存，当访问到这一部分内存时，由于我们修改了的sbrk并没有真正分配物理内存，就会导致访问的页面不存在。

这里的输出包含一些信息：

> 这里输出了SCAUSE寄存器内容，我们可以看到它的值是15，表明这是一个store page fault。
> 产生page fault的进程pid是3
> 我们还可以看到SEPC寄存器的值，是0x12a4，表明产生page fault的指令的地址。
> 最后还可以看到出错的虚拟内存地址，也就是STVAL寄存器的内容，是0x4008。

产生page fault后看到有个panic。这是因为产生页面中断后，内核的做法会杀死当前进程（在usertrap（）中）。进程退出的时候需要free页表，free页表首先会把页表对应项unmap再free物理内存。unmap的时候是从虚拟地址0到p->sz逐页unmap的，在我们修改的sbrk函数中增加了p->sz，对应的内存并没有分配并map。因此在uvmunmap的逻辑中就会panic。

## Lazy allocation

在trap.c中加上处理page fault的逻辑，使得echo hi能够正常运行。

比较简单，仿照课程视频的就行：

> 在usertrap中处理page fault，如果kalloc失败或者访问的地址va不合法，杀死进程

```c
...
else if (r_scause() == 13 || r_scause() == 15) {
    uint64 va = r_stval();
    uint64 ka = (uint64)kalloc();
    if (ka == 0) {
      p->killed = 1;
    }
    else if(isValid(p, va) == 0) {
      kfree((void*)ka);
      p->killed = 1;
    }
    else {
      memset((void*)ka, 0, PGSIZE);
      if (mappages(p->pagetable, PGROUNDDOWN(va), PGSIZE, ka, PTE_U | PTE_R | PTE_W) != 0) {
        kfree((void*)ka);
        p->killed = 1;
      }
    }
  }
...
```

> 判断va是否合法：如果va大于sz或者访问了guard page，则不合法。

```c
int isValid(struct proc *p, uint64 va) {
  uint64 stackbase = PGROUNDDOWN(p->trapframe->sp);
  if (va >= p->sz || (va < stackbase+PGSIZE)) 
    return 0;
  return 1;
}
```

> 处理unmap的panic：如果某一页并没有分配或并没有map，那么可以跳过这一页而不是panic

```c
...
for(a = va; a < va + npages*PGSIZE; a += PGSIZE){
    if((pte = walk(pagetable, a, 0)) == 0)
      //panic("uvmunmap: walk");
      continue;
    if((*pte & PTE_V) == 0)
      //panic("uvmunmap: not mapped");
      continue;
    if(PTE_FLAGS(*pte) == PTE_V)
      panic("uvmunmap: not a leaf");
    if(do_free){
      uint64 pa = PTE2PA(*pte);
      kfree((void*)pa);
    }
    *pte = 0;
  }
...
```

## Lazytests and Usertests

完善lazy allocation，保证：

1.处理sbrk的负数参数

2.如果va大于sz，杀死进程

3.在fork中正确处理父进程到子进程的复制过程

4.处理这样一种情况：系统调用（比如write）传入的虚拟地址对应的内存并没有被分配。

5.物理内存不足时杀死进程

6.处理va在user stack以下的情况

其中2，5，6在上一个实验中已经完成了。isValid函数即处理了2和6。

> sys_sbrk()中处理负数参数的情况
> 如果传入的n为负数，则把p->sz+n到p->sz的地址范围都dealloc
> 这里加上了两个判断，传入sbrk的参数不合法（即sz+n大于trapframe或小于user stack），sbrk失败，返回-1.
> 之前修改过unmap的部分，dealloc的时候如果某一页不存在，不会panic而是会跳过这一页。

```c
uint64
sys_sbrk(void)
{
  int addr;
  int n;
  struct proc *p = myproc();
  addr = myproc()->sz;
  uint64 stackbase = PGROUNDDOWN(p->trapframe->sp);
  if(argint(0, &n) < 0)
    return -1;
  if (n > 0) {
    p->sz += n;
    if (p->sz > MAXVA-2*PGSIZE)
      return -1;
  }
  else if (n < 0) {
    if (p->sz < stackbase+PGSIZE-n) {
      return -1;
    }
    else p->sz = uvmdealloc(p->pagetable, p->sz, p->sz + n);
  }
  return addr;
}
```

> 在fork时会调用uvmcopy复制一份父进程的内存，在lazy allocation中可能0->sz中有部分没有真正分配，在uvmcopy中就会导致panic。累次uvmunmap，修改uvmcopy使得在页面不存在时跳过这一页。

```c
for(i = 0; i < sz; i += PGSIZE){
    if((pte = walk(old, i, 0)) == 0)
      //panic("uvmcopy: pte should exist");
      continue;
    if((*pte & PTE_V) == 0)
      //panic("uvmcopy: page not present");
      continue;
    pa = PTE2PA(*pte);
    flags = PTE_FLAGS(*pte);
    if((mem = kalloc()) == 0)
      goto err;
    memmove(mem, (char*)pa, PGSIZE);
    if(mappages(new, i, PGSIZE, (uint64)mem, flags) != 0){
      kfree(mem);
      goto err;
    }
  }
  return 0;
```

> 处理第4种情况，即系统调用（比如write）传入的虚拟地址对应的内存并没有被分配。
> 首先搞清楚函数执行流程，在调用write后系统trap到内核态，执行copyin来把用户程序va处的内容复制到内核空间，此时若va处并未分配内存，walkaddr会返回0导致系统调用失败。因此我们要做的就是在walkaddr中分配内存。

```c
uint64
walkaddr(pagetable_t pagetable, uint64 va)
{
  pte_t *pte;
  uint64 pa;
      struct proc *p = myproc();
  if(va >= MAXVA)
    return 0;

  pte = walk(pagetable, va, 0);
  if(pte == 0 || (*pte & PTE_V) == 0) {
    uint64 ka = (uint64)kalloc();
    if (ka == 0) {
      return 0;
    }
    else if (isValid(p, va) == 0) {
      kfree((void*)ka);              //注意这里也要kfree，不然会导致内存泄漏
      return 0;
    }
    else {
      memset((void*)ka, 0, PGSIZE);
      if (mappages(p->pagetable, PGROUNDDOWN(va), PGSIZE, ka, PTE_U | PTE_R | PTE_W) != 0) {
        kfree((void*)ka);
        return 0;
      }
      return ka;
    }
  }

  if((*pte & PTE_U) == 0)
    return 0;
  pa = PTE2PA(*pte);
  return pa;
}
```

## 结果

![img](https://pic2.zhimg.com/80/v2-4625c42a0a109262f033de943e938a39_720w.jpg)

## 心得

usertests失败的时候可以看看test代码是怎么写的从而定位错误。

测试usertests前记得make clean一下，不然会导致部分test创建文件失败。