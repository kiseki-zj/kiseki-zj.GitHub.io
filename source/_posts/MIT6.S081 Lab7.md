---
title: MIT6.S081 Lab7 Multithreading
date: 2021-10-12 12:00:00
tags: 6.S081
---

## Uthread: switching between threads

这个lab要求完善一个用户态线程库，比较简单

> 首先创建线程时把context的ra设置为传入的函数地址，这样第一次scheduler到这个线程可以返回到执行函数。
> 注意stack是从上往下增长的，所以sp的值初始化为对应stack的顶部。
> 实际上thread[0].stack是没有用到的，因为main线程一直使用的都是进程分配好的栈区域，其他stack被分配在data段。

```c
void 
thread_create(void (*func)())
{
  struct thread *t;

  for (t = all_thread; t < all_thread + MAX_THREAD; t++) {
    if (t->state == FREE) break;
  }
  t->state = RUNNABLE;
  // YOUR CODE HERE
  t->context.ra = (uint64)func;
  t->context.sp = (uint64)t->stack+STACK_SIZE;
}
```

> sheduler函数中，找到一个RUNNABLE的线程就进行切换

```c
if (current_thread != next_thread) {         /* switch threads?  */
    next_thread->state = RUNNING;
    t = current_thread;
    current_thread = next_thread;
    /* YOUR CODE HERE
     * Invoke thread_switch to switch from t to next_thread:
     * thread_switch(??, ??);
     */
    thread_switch((uint64)(&t->context), (uint64)(&current_thread->context));
  } else
    next_thread = 0;
}
```

> thread_switch代码和swtch一样

```text
thread_switch:
	/* YOUR CODE HERE */
        sd ra, 0(a0)
        sd sp, 8(a0)
        sd s0, 16(a0)
        sd s1, 24(a0)
        sd s2, 32(a0)
        sd s3, 40(a0)
        sd s4, 48(a0)
        sd s5, 56(a0)
        sd s6, 64(a0)
        sd s7, 72(a0)
        sd s8, 80(a0)
        sd s9, 88(a0)
        sd s10, 96(a0)
        sd s11, 104(a0)

        ld ra, 0(a1)
        ld sp, 8(a1)
        ld s0, 16(a1)
        ld s1, 24(a1)
        ld s2, 32(a1)
        ld s3, 40(a1)
        ld s4, 48(a1)
        ld s5, 56(a1)
        ld s6, 64(a1)
        ld s7, 72(a1)
        ld s8, 80(a1)
        ld s9, 88(a1)
        ld s10, 96(a1)
        ld s11, 104(a1)
	
	ret    /* return to ra */
```

另外需要定义struct context并作为thread类的一个成员变量

## Using threads

这个lab展示了如何使用linux pthread线程库。程序定义了一个链式哈希表并向其中put 10000个kv数据对。创建多个线程执行put操作时，效率会有所提高但会产生miss的情况，也就是一些key并没有成功写入hash table。原因是线程对hash table的操作没有加锁。例如线程t1要对其中某个链表进行e1的头插，当执行完 e->next = n后切换到线程t2，线程t2同样要对该链表进行e2的头插，也运行到e->next = n。

这时候有两种情况：t1先执行 *p = e，这样就丢失了e2这个数据；t2先执行 *p = e;，这样会丢失e1.所以需要对共享数据加锁。最简单的方法是对整个hash table加锁，每次put前都需要acquire这个锁，但这样效率很低，所有put操作相当于都串行操作了。hints告诉我们实际上只要对hash table的每个bucket加锁就行，因为不同bucket的插入操作彼此互不影响。

代码的话比较简单，主要是熟悉pthread库的mutex锁的使用。

> lock定义了每个bucket的锁，put时加锁即可。

```c
...
pthread_mutex_t lock[NBUCKET];
...
static 
void put(int key, int value)
{
  int i = key % NBUCKET;

  // is the key already present?
  struct entry *e = 0;
  pthread_mutex_lock(&lock[i]);
  for (e = table[i]; e != 0; e = e->next) {
    if (e->key == key)
      break;i
  }
  if(e){
    // update the existing key.
    e->value = value;
  } else {
    // the new is new.
    insert(key, value, &table[i], table[i]);
  }
  pthread_mutex_unlock(&lock[i]);
}
```

## Barrier

程序开始创建了多个线程，要求每个线程都运行到barrier函数就sleep，直到所有线程都到达barrier函数才能被wakeup。这个lab也比较简单，主要是熟悉pthread的条件变量，本质上和xv6的sleep&wakeup一样。

> barrier_mutex用来保护条件变量nthread，当nthread满足条件后wakeup所有等待barrier_cond这个channel的线程。

```c
static void 
barrier()
{
  // YOUR CODE HERE
  //
  // Block until all threads have called barrier() and
  // then increment bstate.round.
  //
  pthread_mutex_lock(&bstate.barrier_mutex);
  bstate.nthread++;
  if (bstate.nthread == nthread) {
    bstate.nthread = 0;
    bstate.round++;
    pthread_cond_broadcast(&bstate.barrier_cond);
  }
  else {
    pthread_cond_wait(&bstate.barrier_cond, &bstate.barrier_mutex);
  }
  pthread_mutex_unlock(&bstate.barrier_mutex);
}
```



结果

![img](https://pic2.zhimg.com/80/v2-0e04ba414726a4cb23de4791fb4f4ca9_720w.jpg)