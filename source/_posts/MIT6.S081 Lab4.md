---
title: MIT6.S081 Lab4 traps
date: 2021-04-16 12:00:00
tags: 6.S081
---

## 实验前准备

> 阅读trap.c，trampoline.s

## RISC-V assembly

回答几个问题，阅读call.c和它的汇编代码理解函数的calling conventions即可

> call.c

```c
#include "kernel/param.h"
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int g(int x) {
  return x+3;
}

int f(int x) {
  return g(x);
}

void main(void) {
  printf("%d %d\n", f(8)+1, 13);
  exit(0);
}
```

> call.asm

```jasmin
int g(int x) {
   0:	1141                	addi	sp,sp,-16
   2:	e422                	sd	s0,8(sp)
   4:	0800                	addi	s0,sp,16
  return x+3;
}
   6:	250d                	addiw	a0,a0,3
   8:	6422                	ld	s0,8(sp)
   a:	0141                	addi	sp,sp,16
   c:	8082                	ret

000000000000000e <f>:

int f(int x) {
   e:	1141                	addi	sp,sp,-16
  10:	e422                	sd	s0,8(sp)
  12:	0800                	addi	s0,sp,16
  return g(x);
}
  14:	250d                	addiw	a0,a0,3
  16:	6422                	ld	s0,8(sp)
  18:	0141                	addi	sp,sp,16
  1a:	8082                	ret

000000000000001c <main>:

void main(void) {
  1c:	1141                	addi	sp,sp,-16
  1e:	e406                	sd	ra,8(sp)
  20:	e022                	sd	s0,0(sp)
  22:	0800                	addi	s0,sp,16
  printf("%d %d\n", f(8)+1, 13);
  24:	4635                	li	a2,13
  26:	45b1                	li	a1,12
  28:	00000517          	auipc	a0,0x0
  2c:	7b850513          	addi	a0,a0,1976 # 7e0 <malloc+0xea>
  30:	00000097          	auipc	ra,0x0
  34:	608080e7          	jalr	1544(ra) # 638 <printf>
  exit(0);
  38:	4501                	li	a0,0
  3a:	00000097          	auipc	ra,0x0
  3e:	276080e7          	jalr	630(ra) # 2b0 <exit>
```

> Q1.哪个寄存器保存函数参数，例如哪个保存参数13?

A：a0，a1，a2...保存传入的参数，13保存在a2中

> Q2.哪里调用了函数f和g？

A：没有调用，编译器把两个函数内联了。

> Q3.函数printf的地址？

A：观察汇编代码有

```text
  30:	00000097          	auipc	ra,0x0
  34:	608080e7          	jalr	1544(ra) # 638 <printf>
```

auipc的作用是把立即数左移12位，低12位补0，和pc相加赋给指定寄存器。这里立即数是0，指定寄存器是ra，即ra=pc=0x30=48。jalr作用是跳转到立即数+指定寄存器处并且把ra的值+8。因此jalr会跳转到1544+48=1592=0x638处，观察汇编代码发现：

```text
0000000000000638 <printf>:

void
printf(const char *fmt, ...)
{
 638:	711d                	addi	sp,sp,-96
 63a:	ec06                	sd	ra,24(sp)
 63c:	e822                	sd	s0,16(sp)
 63e:	1000                	addi	s0,sp,32
 640:	e40c                	sd	a1,8(s0)
 642:	e810                	sd	a2,16(s0)
```

确实在0x638

> Q4.在printf之后ra的值？

A.如上述，为0x30+8=0x38，用gdb也可以看到

![img](https://pic1.zhimg.com/80/v2-7f3932002f682a177fe484725656a190_720w.jpg)

执行完jalr后ra从0x30变为0x38

> Q5.运行以下代码，输出是什么？这个输出基于riscv是小端保存，如果是大端保存，怎么设置i才能获得相同输出？

```c
unsigned int i = 0x00646c72;
printf("H%x Wo%s", 57616, &i);
```

A：输出为HE110 World

因为riscv为小端存储，从&i开始字节分别为0x72，0x6c，0x64， 0x00.分别对应'r','l','d'，'0'的ascii码，0x00作为字符串结束标志。

57616=0xE110

i应该设置为0x726c6400

> Q6.运行printf("x=%d y=%d", 3);在y=后面输出什么？为什么会这样？

A：输出y=1。取决于寄存器a2（第3个参数）的值。



## Backtrace

要求完成一个backtrace函数，在系统调用sys_sleep里调用这个函数，使其从下往上逐一输出内核栈各个stackframe的ra的值。

主要是弄懂调用函数的stack frame的结构，如下图：

![img](https://pic4.zhimg.com/80/v2-4351f298147c4da1f1b72aef8db25093_720w.jpg)

sp指向当前frame底部（栈是从上往下增长），fp指向当前frame顶部。栈顶部保存了函数返回地址ra以及上个frame的fp。具体可以看任意一个函数的汇编代码：

![img](https://pic4.zhimg.com/80/v2-03e9232fd2cdaa8b6cae56499e39ac6b_720w.jpg)

一个函数在开始执行时会把ra，s0（即fp）的值压栈，然后保存一些寄存器的值，如果参数过多也会把参数保存在栈里，还会保存局部变量。

backtrace的实现比较简单，模拟函数返回的步骤，输出ra（fp-8），然后找到上一个frame的fp（fp-16），以此类推。注意要求只输出内核态的frame信息，fp如果跳出内核栈的范围到了用户空间循环就要停止了。

> backtrace()

```c
void backtrace(void) {
  uint64 fp = r_fp();
  struct proc *p = myproc();
  
  while (1) {
    uint64 *ra_addr = (uint64*)(fp - 8);
    uint64 *fp_addr = (uint64*)(fp - 16);
    fp = *fp_addr;
    //printf("fp=%p\n", *fp_addr);
    if (PGROUNDUP(fp) != p->kstack+PGSIZE) 
      break;
    printf("%p\n", *ra_addr);
  }
  return ;
}
```

## Alarm

实现系统调用sigalarm(n, fn)，执行这个系统调用之后，用户程序在执行的时候能够每隔n个ticks执行一次用户态函数fn。

riscv收到timer trap的情况有两种，在用户态收到trap和在内核态收到trap，分别会执行usertrap和kerneltrap处理函数。实验要求是在用户态收到n个ticks后执行函数，不用修改kerneltrap的代码。

实验分两个步骤。test0要求程序执行了fn即可，不用管控制流有没有正常返回以及寄存器的值是否变动，test1和test2要求程序在执行fn后能够通过编写的系统调用sigreturn返回到产生trap的用户代码处，并且程序上下文没有改变。

> proc.h
> 给struct添加几个部分，包括需要执行的用户态函数p->handler，执行函数的间隔p->interval，距离上一次执行handler后已经产生的tick数p->tick（在allocproc处初始化为0）。
> 然后是用户态产生trap时的所有寄存器的值。由于执行handler时可能改变某些寄存器，再返回到原先的代码处时需要恢复现场。
> p->flag用于判断当前时钟中断是否是在执行handler时产生的，如果是，则不需要执行handler。

```as3
struct proc {
  struct spinlock lock;

  // p->lock must be held when using these:
  enum procstate state;        // Process state
  struct proc *parent;         // Parent process
  void *chan;                  // If non-zero, sleeping on chan
  int killed;                  // If non-zero, have been killed
  int xstate;                  // Exit status to be returned to parent's wait
  int pid;                     // Process ID

  // these are private to the process, so p->lock need not be held.
  uint64 kstack;               // Virtual address of kernel stack
  uint64 sz;                   // Size of process memory (bytes)
  pagetable_t pagetable;       // User page table
  struct trapframe *trapframe; // data page for trampoline.S
  struct context context;      // swtch() here to run process
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)
  int interval;
  uint64 handler;
  int tick;
  uint64 u_epc;
  uint64 u_ra;
  uint64 u_sp;
  uint64 u_gp;
  uint64 u_tp;
  uint64 u_t0;
  uint64 u_t1;
  uint64 u_t2;
  uint64 u_s0;
  uint64 u_s1;
  uint64 u_a0;
  uint64 u_a1;
  uint64 u_a2;
  uint64 u_a3;
  uint64 u_a4;
  uint64 u_a5;
  uint64 u_a6;
  uint64 u_a7;
  uint64 u_s2;
  uint64 u_s3;
  uint64 u_s4;
  uint64 u_s5;
  uint64 u_s6;
  uint64 u_s7;
  uint64 u_s8;
  uint64 u_s9;
  uint64 u_s10;
  uint64 u_s11;
  uint64 u_t3;
  uint64 u_t4;
  uint64 u_t5;
  uint64 u_t6;
  uint64 flag;
};
```

> 在kernel目录新建alarm.c用于编写所需的两个系统调用。
> sigalarm在调用时设置好proc的inteval以及handler的值，注意传入的函数地址也可以为0（因为在用户空间，代码段从虚拟地址0开始）。

```c
uint64 sys_sigalarm(void) {
    int itv;
    uint64 handler;
    struct proc* p = myproc();
    argint(0, &itv);
    argaddr(1, &handler);
    p->interval = itv;
    p->handler = handler;
    p->flag = 0;
    return 0;
}
```

> usertrap()：
> 在每次产生时钟中断时p->tick加1，然后判断是否等于intervel。如果不等于就当成普通的时钟中断处理，如果等于并且当前用户态代码不在执行handler（flag==0）则执行以下操作：
> 把所有寄存器的值保存，把trampframe->epc的值设置为handler，这样一来在usertrapret时就会把epc设置为handler的第一句代码处，userret（trampoline.S）时执行sret就会返回到handler，从而执行handler。记住把flag设为1.

```c
if(which_dev == 2) {
    //printf("ticks=%d\n", ticks);
    p->tick++;
    if (p->tick == p->interval) {
      if (p->flag == 0) {
        p->u_epc = p->trapframe->epc;
        p->u_ra = p->trapframe->ra;
        p->u_sp = p->trapframe->sp;
        p->u_gp = p->trapframe->gp;
        p->u_tp = p->trapframe->tp;
        p->u_t0 = p->trapframe->t0;
        p->u_t1 = p->trapframe->t1;
        p->u_t2 = p->trapframe->t2;
        p->u_s0 = p->trapframe->s0;
        p->u_s1 = p->trapframe->s1;
        p->u_a0 = p->trapframe->a0;
        p->u_a1 = p->trapframe->a1;
        p->u_a2 = p->trapframe->a2;
        p->u_a3 = p->trapframe->a3;
        p->u_a4 = p->trapframe->a4;
        p->u_a5 = p->trapframe->a5;
        p->u_a6 = p->trapframe->a6;
        p->u_a7 = p->trapframe->a7;
        p->u_s2 = p->trapframe->s2;
        p->u_s3 = p->trapframe->s3;
        p->u_s4 = p->trapframe->s4;
        p->u_s5 = p->trapframe->s5;
        p->u_s6 = p->trapframe->s6;
        p->u_s7 = p->trapframe->s7;
        p->u_s8 = p->trapframe->s8;
        p->u_s9 = p->trapframe->s9;
        p->u_s10 = p->trapframe->s10;
        p->u_s11 = p->trapframe->s11;
        p->u_t3 = p->trapframe->t3;
        p->u_t4 = p->trapframe->t4;
        p->u_t5 = p->trapframe->t5;
        p->u_t6 = p->trapframe->t6;
      
        p->trapframe->epc = (uint64)p->handler;
        p->flag = 1;
      }
      p->tick = 0;
    }
    yield();
  }
```

> sigreturn()：在handler最后调用，保证控制流返回到产生trap的那句用户态代码处。
> 把trapframe->epc设置为time trap时保存的epc值，这样保证返回到正确的代码处。
> 恢复所有保存的寄存器。
> handler里也会执行系统调用从而陷入内核态，trapframe会重新进行保存，所以不能保证handler函数的执行过程中进程的trampframe不会产生改变，但p->u_只有在时钟中断时并且tick=interval时（即进入handler之前）才会进行修改，因此可以保证返回到正确的上下文。

```c
uint64 sys_sigreturn(void) {
    struct proc* p = myproc();
    p->trapframe->epc = p->u_epc;
    p->trapframe->ra = p->u_ra;
    p->trapframe->sp = p->u_sp;
    p->trapframe->gp = p->u_gp;
    p->trapframe->tp = p->u_tp;
    p->trapframe->t0 = p->u_t0;
    p->trapframe->t1 = p->u_t1;
    p->trapframe->t2 = p->u_t2;
    p->trapframe->s0 = p->u_s0;
    p->trapframe->s1 = p->u_s1;
    p->trapframe->a0 = p->u_a0;
    p->trapframe->a1 = p->u_a1;
    p->trapframe->a2 = p->u_a2;
    p->trapframe->a3 = p->u_a3;
    p->trapframe->a4 = p->u_a4;
    p->trapframe->a5 = p->u_a5;
    p->trapframe->a6 = p->u_a6;
    p->trapframe->a7 = p->u_a7;
    p->trapframe->s2 = p->u_s2;
    p->trapframe->s3 = p->u_s3;
    p->trapframe->s4 = p->u_s4;
    p->trapframe->s5 = p->u_s5;
    p->trapframe->s6 = p->u_s6;
    p->trapframe->s7 = p->u_s7;
    p->trapframe->s8 = p->u_s8;
    p->trapframe->s9 = p->u_s9;
    p->trapframe->s10 = p->u_s10;
    p->trapframe->s11 = p->u_s11;
    p->trapframe->t3 = p->u_t3;
    p->trapframe->t4 = p->u_t4;
    p->trapframe->t5 = p->u_t5;
    p->trapframe->t6 = p->u_t6;
    p->flag = 0;
    return 0;
}
```

这是部分核心代码，其他的就是把sigalarm和sigreturn注册成系统调用，编写makefile等，不再赘述。