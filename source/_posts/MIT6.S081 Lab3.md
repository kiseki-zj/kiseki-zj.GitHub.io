---
title: MIT6.S081 Lab3 pgtbl
date: 2021-04-03 12:00:00
tags: 6.S081
---
### 实验前准备

> 阅读vm.c,proc.c,exec.c,看明白xv6怎么初始化内核页表以及怎么实现内存isolation的.阅读trap.c,trampoline.S,明白用户程序陷入内核态以及从内核态返回用户态的过程.
> 阅读xv6book chapter3&chapter4.

### Print a page table

第一个实验要求实现一个函数vmprint(pagetable_t pagetable),输出页表的内容.其实就是仿照walk模拟硬件翻译的过程,中间把页表项的值输出就可以了.比较简单:

```c
void _vmprint(pagetable_t pagetable, uint64 depth) {
  if (depth == 1) 
    printf("page table %p\n", pagetable);
  if (depth == 4) return;
  for (int i = 0; i < 512; i++) {
    pte_t pte = pagetable[i];
    if (pte & PTE_V) {
      printf("..");
      for (int j = 1; j < depth; j++)
        printf(" ..");
      printf("%d: pte %p pa %p\n", i, pte, PTE2PA(pte));
      _vmprint((pagetable_t)PTE2PA(pte), depth+1);
    }
  }
}

void vmprint(pagetable_t pagetable) {
  _vmprint(pagetable, 1);
}
```

这个实验主要是为了后面实验debug用.

### A kernel page table per process

xv6为每个进程分配了一个用户页表,在用户态时使用的是当前进程的pagetable,当陷入内核态时会把satp切换为全局的内核页表kernel_pagetable (trampoline.S 78行).kernel_pagetable使用了direct map,把虚拟内存直接map到物理内存,同时也初始化了一些设备地址(kvminit()).如下图

![img](https://pic1.zhimg.com/80/v2-d992e535c19abc307ea2230979f5c53c_720w.jpg)

当用户程序调用系统调用传入一个地址时需要在内核态进行翻译获得物理地址.举个例子,程序使用write系统调用传入一个buf,这个buf本质是一个地址,假设为a,它对应的物理地址是b.那么系统陷入到内核态后,由于此时的页表是kernel_pagetable而不是用户的pagetable,kernel_pagetable并没有地址a的对应关系,因此需要通过walk模拟MMU来获得a在用户pagetable中的物理地址即b.

这个实验和下一个实验就是要求为每个进程分配一个kpagetable,当用户陷入内核态时切换到kpagetable而不是全局的kernel_pagetable.这个kpagetable的内容要求有3个:

1.内容和kernel_pagetable基本一样使得在内核态时能够使用内核的代码和数据

2.在trampoline下面是进程的内核栈,kpagetable只能把本进程对应的内核栈map好.其他进程的不能map

3.最重要的一点,由于进程的va从0开始且连续增长,我们在kpagetable中也要保存进程va和pa的对应关系,这样在内核态时可以直接使用用户空间的虚拟地址.以上述例子来说,内核态可以直接使用地址a,因为硬件会自动把a翻译为物理地址b.

这个实验要求完成前两点,下个实验要求完成第3点.

> proc.h

```c
pagetable_t kpagetable;
```

> allocproc() 加入初始化kpagetable的过程,proc_kernel_kvminit()类似kvminit把物理内存和设备以及trampoline的地址map好.
> kprocinit()用来map本进程的内核栈

```c
...
// An empty user page table.
  p->pagetable = proc_pagetable(p);
  if(p->pagetable == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
  }
  //lab3 step2

  p->kpagetable = proc_kernel_kvminit(p);
  if(p->kpagetable == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
  }
  kprocinit(p);
...
```

> proc_kernel_kvminit()
> 这里面的proc_kvmmap和kvmmap一样,只不过加了参数指定pagetable

```c
pagetable_t
proc_kernel_kvminit(struct proc *p)
{
  pagetable_t pagetable;
  pagetable = (pagetable_t) kalloc();
  memset(pagetable, 0, PGSIZE);
  // uart registers
  proc_kvmmap(pagetable, UART0, UART0, PGSIZE, PTE_R | PTE_W);
  // virtio mmio disk interface
  proc_kvmmap(pagetable, VIRTIO0, VIRTIO0, PGSIZE, PTE_R | PTE_W);

  // CLINT
  proc_kvmmap(pagetable, CLINT, CLINT, 0x10000, PTE_R | PTE_W);

  // PLIC
  proc_kvmmap(pagetable, PLIC, PLIC, 0x400000, PTE_R | PTE_W);

  // map kernel text executable and read-only.
  proc_kvmmap(pagetable, KERNBASE, KERNBASE, (uint64)etext-KERNBASE, PTE_R | PTE_X);

  // map kernel data and the physical RAM we'll make use of.
  proc_kvmmap(pagetable, (uint64)etext, (uint64)etext, PHYSTOP-(uint64)etext, PTE_R | PTE_W);
  // map the trampoline for trap entry/exit to
  // the highest virtual address in the kernel.
  proc_kvmmap(pagetable, TRAMPOLINE, (uint64)trampoline, PGSIZE, PTE_R | PTE_X);
  return pagetable;
}
```

> void kprocinit(struct proc *p)
> 内核栈物理内存在procinit()时就已经alloc好了,所以只要在页表里map一下就行

```c
void kprocinit(struct proc *p) {
  uint64 va = KSTACK((int) (p - proc));
  uint64 pa = kvmpa(va);
  //printf("p->kstack = %p\n", KSTACK((int) (p - proc)));
  proc_kvmmap(p->kpagetable, (uint64)va, (uint64)pa, PGSIZE, PTE_R | PTE_W);
  p->kstack = va;
}
```

> freeproc()
> 加入free kpagetable的过程,首先把用户空间的地址都unmap掉(这是第三个实验要做的),然后proc_free_kpagetable用来free页表.

```c
static void
freeproc(struct proc *p)
{
  if(p->trapframe)
    kfree((void*)p->trapframe);
  p->trapframe = 0;
  if(p->pagetable)
    proc_freepagetable(p->pagetable, p->sz);

  uvmunmap(p->kpagetable, 0, PGROUNDUP(p->sz) / PGSIZE, 0);
  //lab3 step2
  if (p->kpagetable)
    proc_free_kpagetable(p);
  p->pagetable = 0;
  p->kpagetable = 0;
  p->sz = 0;
  p->pid = 0;
  p->parent = 0;
  p->name[0] = 0;
  p->chan = 0;
  p->killed = 0;
  p->xstate = 0;
  p->state = UNUSED;
}
```

> kuvmunmap和uvmunmap相比去掉了panic,这是因为第三个实验map用户地址时会把底层某些地址覆盖掉,然后如果再unmap掉部分虚拟地址,就会导致中间有的地址unmap.比如UART0是0x1000000,如果用户虚拟地址增长到了0x1001000,再sbrk(-1000)删除一页,这样0x1000000->0x1001000这部分页表是空的,此时freeproc执行kuvmunmap(p->kpagetable, UART0, 1, 0)时就会出现not mapped的panic.

```c
void proc_free_kpagetable(struct proc *p)
{
  kuvmunmap(p->kpagetable, TRAMPOLINE, 1, 0);
  kuvmunmap(p->kpagetable, KSTACK((int) (p - proc)), 1, 0);
  kuvmunmap(p->kpagetable, UART0, 1, 0);
  kuvmunmap(p->kpagetable, VIRTIO0, 1, 0);
  kuvmunmap(p->kpagetable, CLINT, 0x10000 / PGSIZE, 0);
  kuvmunmap(p->kpagetable, PLIC, 0x400000 / PGSIZE, 0);
  kuvmunmap(p->kpagetable, KERNBASE, (uint64)(etext-KERNBASE) / PGSIZE, 0);
  kuvmunmap(p->kpagetable, (uint64)etext, (PHYSTOP-(uint64)etext) / PGSIZE, 0);
  freewalk(p->kpagetable);
}
```

> scheduler()
> 切换进程前切换到对应的kpagetable,某进程退出后回到scheduler使用全局kernel_pagetable

```c
for(p = proc; p < &proc[NPROC]; p++) {
      acquire(&p->lock);
      if(p->state == RUNNABLE) {
        // Switch to chosen process.  It is the process's job
        // to release its lock and then reacquire it
        // before jumping back to us.
        p->state = RUNNING;
        c->proc = p;
        //lab3 step2
        w_satp(MAKE_SATP(p->kpagetable));
        sfence_vma();

        swtch(&c->context, &p->context);

        // Process is done running for now.
        // It should have changed its p->state before coming back.

        //lab3 step2
        w_satp(MAKE_SATP(kernel_pagetable));
        sfence_vma();

        c->proc = 0;

        found = 1;
      }
      release(&p->lock);
    }
```

### Simplify copyin/copyinstr

这个实验要完成第3个条件,在kpagetable把用户空间的虚拟地址也map好.hints已经告诉我们在exec(),fork(),growproc()里修改kpagetable就可以了,因为只有这些函数会修改用户pagetable.

> exec()
> 这里的逻辑就是把kpagetable底部用户空间的部分先全部unmap掉,再把用户pagetable的内容通过kuvmcopy复制到kpagetable里
> 之所以unmap时不需要free掉物理内存,是因为下面proc_freepagetable(oldpagetable, oldsz)把原先的pagetable释放掉同时free了对应的物理内存.

```c
...
oldpagetable = p->pagetable;
  p->pagetable = pagetable;
  p->sz = sz;
  p->trapframe->epc = elf.entry;  // initial program counter = main
  p->trapframe->sp = sp; // initial stack pointer
  //lab3 step3
  uvmunmap(p->kpagetable, 0, PGROUNDUP(oldsz) / PGSIZE, 0);
  if (kuvmcopy(p, sz, PTE_R|PTE_W|PTE_X) != 0) {
    panic("kuvmcopy");
  }
  w_satp(MAKE_SATP(p->kpagetable));
  sfence_vma();
  proc_freepagetable(oldpagetable, oldsz);
...
```

> kuvmcopy()
> 把进程new的pagetable底部(即不包含trampoline和trapframe)复制给kpagetable
> 函数mappages_remap和mappages相比去掉了remap的panic,理由在step2里提到了,kpagetable在map时可能需要覆盖某些设备的地址.

```c
int kuvmcopy(struct proc *new, uint64 sz, int perm) {
  pte_t *pte;
  uint64 pa, i;
  pagetable_t newpg = new->pagetable;
  for(i = 0; i < sz; i += PGSIZE){
    if((pte = walk(newpg, i, 0)) == 0)
      panic("uvmcopy: pte should exist");
    if((*pte & PTE_V) == 0)
      panic("uvmcopy: page not present");
    pa = PTE2PA(*pte);
    if(mappages_remap(new->kpagetable, i, PGSIZE, pa, perm) != 0){
      goto err;
    }
  }
  return 0;

 err:
  uvmunmap(new->kpagetable, 0, i / PGSIZE, 1);
  return -1;
}
```

> fork()
> 同理把fork出来的子进程的用户pagetable复制给kpagetable,注意flag不能有PTE_U,因为cpu在suprivisor模式时不能访问设置PTE_U的页

```c
...
// Copy user memory from parent to child.
  if(uvmcopy(p->pagetable, np->pagetable, p->sz) < 0){
    freeproc(np);
    release(&np->lock);
    return -1;
  }
  if(kuvmcopy(np, p->sz, PTE_R|PTE_W|PTE_X) < 0){
    freeproc(np);
    release(&np->lock);
    return -1;
...
```

> growproc()
> 这个稍微麻烦一点,用户空间增长n时要同时在pagetable和kpagetable进行map,同理n<0也要.

```c
int
growproc(int n)
{
  uint sz;
  struct proc *p = myproc();
  //lab3 step3

  uint oldsz;
  oldsz = sz = p->sz;
  if(n > 0){
    sz = kuvmalloc(p, sz, sz + n);
    if( sz == 0 ) {
      return -1;
    }
    if (sz > PLIC) {
      kuvmdealloc(p, sz, oldsz);
      return -1;
    }
  } else if(n < 0){
      sz = kuvmdealloc(p, sz, sz + n);
  } 
  p->sz = sz;
  return 0;
}
```

> kuvmalloc()
> 和uvmalloc相比加入了给kpagetable进行map的过程,注意在mappages_remap返回-1的时候要不仅要free物理内存,还要把之前mappages()时用户pagetable给unmap掉,一直过不去sbrkfail这个test的原因就是这个.
> 具体来说,比如内存还剩100,kuvmalloc分配第99页后不会触发mem=0,但是执行mappages会把物理内存剩的最后一页用于分配页表,此时执行mappages_remap就会失败(因为没有物理内存了),在这个逻辑下是不应该分配这个第99页的.如果直接执行kuvmdealloc会把用户pagetable第99页的map关系保留下来,导致最后freeproc时panic leaf map.

```c
uint64
kuvmalloc(struct proc *p, uint64 oldsz, uint64 newsz)
{
  char *mem;
  uint64 a;
  pagetable_t pagetable = p->pagetable;
  pagetable_t kpagetable = p->kpagetable;
  if(newsz < oldsz)
    return oldsz;
  oldsz = PGROUNDUP(oldsz);
  for(a = oldsz; a < newsz; a += PGSIZE){
    mem = kalloc();
    if(mem == 0){
      kuvmdealloc(p, a, oldsz);    
      return 0;
    }
    memset(mem, 0, PGSIZE);    
    if(mappages(pagetable, a, PGSIZE, (uint64)mem, PTE_W|PTE_X|PTE_R|PTE_U) != 0){
      kfree(mem);
      kuvmdealloc(p, a, oldsz);
      return 0;
    }
    if (mappages_remap(kpagetable, a, PGSIZE, (uint64)mem, PTE_W|PTE_X|PTE_R) != 0) {
      kfree(mem);
      uvmunmap(pagetable, a, 1, 0);
      kuvmdealloc(p, a, oldsz);
      return 0;
    }    
  }
  return newsz;
}
```

> kuvmdealloc()
> 仿照uvmdealloc就行

```c
uint64
kuvmdealloc(struct proc *p, uint64 oldsz, uint64 newsz)
{
  pagetable_t pagetable = p->pagetable;
  pagetable_t kpagetable = p->kpagetable;
  if(newsz >= oldsz)
    return oldsz;

  if(PGROUNDUP(newsz) < PGROUNDUP(oldsz)){
    int npages = (PGROUNDUP(oldsz) - PGROUNDUP(newsz)) / PGSIZE;
    uvmunmap(pagetable, PGROUNDUP(newsz), npages, 1);
    kuvmunmap(kpagetable, PGROUNDUP(newsz), npages, 0);
  }
  return newsz;
}
```

### 实验结果

![img](https://pic3.zhimg.com/80/v2-d0566e3093b15954bdaedd4d1fcdf192_720w.jpg)

### 心得

主要是要先看懂几个重要函数的作用,弄懂陷入内核态和返回用户态的过程,理解内核页表和用户页表的内存分布,搞清楚实验目的是什么.接下来就是耐心debug了,如果usertests某些测试一直通不过可以看一下这个test做了些什么,以此定位哪里的代码可能出错.