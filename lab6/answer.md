# 实验六: 调度器
## 实验目的
- 理解操作系统的调度管理机制
- 熟悉 ucore 的系统调度器框架，以及缺省的Round-Robin 调度算法
- 基于调度器框架实现一个(Stride Scheduling)调度算法来替换缺省的调度算法

## 实验内容
- 实验五完成了用户进程的管理，可在用户态运行多个进程。
- 之前采用的调度策略是很简单的FIFO调度策略。
- 本次实验，主要是熟悉ucore的系统调度器框架，以及基于此框架的Round-Robin（RR） 调度算法。
- 然后参考RR调度算法的实现，完成Stride Scheduling调度算法。

## 调度框架和调度算法设计与实现
实验六中的kern/schedule/sched.c只实现了调度器框架，而不再涉及具体的调度算法实现，调度算法在单独的文件（default_sched.[ch]）中实现。

在init.c中的kern_init函数中的proc_init之前增加了对sched_init函数的调用。sched_init函数主要完成了对实现特定调度算法的调度类（sched_class，这里是default_sched_class）的绑定，使得ucore在后续的执行中，能够通过调度框架找到实现特定调度算法的调度类并完成进程调度相关工作。

### 进程状态
```
struct proc_struct {
    enum proc_state state;                      // Process state
    int pid;                                    // Process ID
    int runs;                                   // the running times of Proces
    uintptr_t kstack;                           // Process kernel stack
    volatile bool need_resched;                 // bool value: need to be rescheduled to release CPU?
    struct proc_struct *parent;                 // the parent process
    struct mm_struct *mm;                       // Process's memory management field
    struct context context;                     // Switch here to run process
    struct trapframe *tf;                       // Trap frame for current interrupt
    uintptr_t cr3;                              // CR3 register: the base addr of Page Directroy Table(PDT)
    uint32_t flags;                             // Process flag
    char name[PROC_NAME_LEN + 1];               // Process name
    list_entry_t list_link;                     // Process link list
    list_entry_t hash_link;                     // Process hash list
};
```

ucore定义的进程控制块struct proc_struct包含了成员变量state,用于描述进程的运行状态，而running和runnable共享同一个状态(state)值(PROC_RUNNABLE。不同之处在于处于running态的进程不会放在运行队列中。进程的正常生命周期如下：
- 进程首先在 cpu 初始化或者 sys_fork 的时候被创建，当为该进程分配了一个进程控制块之后，该进程进入 uninit态(在proc.c 中 alloc_proc)。
- 当进程完全完成初始化之后，该进程转为runnable态。
- 当到达调度点时，由调度器`sched_class`根据运行队列run_queue的内容来判断一个进程是否应该被运行，即把处于runnable态的进程转换成running状态，从而占用CPU执行。
- running态的进程通过wait等系统调用被阻塞，进入sleeping态。
- sleeping态的进程被wakeup变成runnable态的进程。
- running态的进程主动 exit 变成zombie态，然后由其父进程完成对其资源的最后释放，子进程的进程控制块成为unused。
- 所有从runnable态变成其他状态的进程都要出运行队列，反之，被放入某个运行队列中。

### 进程调度实现
#### 内核抢占点
对于用户进程而言，由于有中断的产生，可以随时打断用户进程的执行，转到操作系统内部，从而给了操作系统以调度控制权，让操作系统可以根据具体情况（比如用户进程时间片已经用完了）选择其他用户进程执行。这体现了用户进程的可抢占性。

ucore内核执行是不可抢占的（non-preemptive），即在执行“任意”内核代码时，CPU控制权不可被强制剥夺。这里需要注意，不是在所有情况下ucore内核执行都是不可抢占的，有以下几种“固定”情况是例外：
1. 进行同步互斥操作，比如争抢一个信号量、锁（lab7中会详细分析）；
2. 进行磁盘读写等耗时的异步操作，由于等待完成的耗时太长，ucore会调用shcedule让其他就绪进程执行。

以上两种是因为某个资源（也可称为事件）无法得到满足，无法继续执行下去，从而不得不主动放弃对CPU的控制权。在lab5中有几种情况是调用了schedule函数的。

编号|位置|原因
---|---|---
1|proc.c:do_exit|用户线程执行结束，主动放弃CPU
2|proc.c:do_wait|用户线程等待着子进程结束，主动放弃CPU
3|proc.c:init_main|Init_porc内核线程等待所有用户进程结束；所有用户进程结束后回收系统资源
4|proc.c:cpu_idle|idleproc内核线程等待处于就绪态的进程或线程，如果有选择一个并切换
5|sync.h:lock|进程无法得到锁，则主动放弃CPU
6|trap.c:trap|修改当前进程时间片，若时间片用完，则设置need_resched为1，让当前进程放弃CPU

第1、2、5处的执行位置体现了由于获取某种资源一时等不到满足、进程要退出、进程要睡眠等原因而不得不主动放弃CPU。第3、4处的执行位置比较特殊，initproc内核线程等待用户进程结束而执行schedule函数；idle内核线程在没有进程处于就绪态时才执行，一旦有了就绪态的进程，它将执行schedule函数完成进程调度。这里只有第6处的位置比较特殊：
```
if (!in_kernel) {
    ……

    if (current->need_resched) {
        schedule();
    }
}
```

只有当进程在用户态执行到“任意”某处用户代码位置时发生了中断，且当前进程控制块成员变量need_resched为1（表示需要调度了）时，才会执行shedule函数。这实际上体现了对用户进程的可抢占性。如果没有第一行的if语句，那么就可以体现对内核代码的可抢占性。但如果要把这一行if语句去掉，我们就不得不实现对ucore中的所有全局变量的互斥访问操作，以防止所谓的race-condition现象，这样ucore的实现复杂度会增加不少。

Race condition旨在描述一个系统或者进程的输出依赖于不受控制的事件出现顺序或者出现时机。此词源自于两个信号试着彼此竞争，来影响谁先输出。 举例来说，如果计算机中的两个进程同时试图修改一个共享内存的内容，在没有并发控制的情况下，最后的结果依赖于两个进程的执行顺序与时机。而且如果发生了并发访问冲突，则最后的结果是不正确的。从维基百科的定义来看，race condition不仅仅是出现在程序中。以下讨论的race conditon全是计算机中多个进程同时访问一个共享内存，共享变量的例子。

要阻止出现race condition情况的关键就是不能让多个进程同时访问那块共享内存。访问共享内存的那段代码就是critical section。所有的解决方法都是围绕这个critical section来设计的。想要成功的解决race condition问题，并且程序还可以正确运行，从理论上应该满足以下四个条件： 
1. 不会有两个及以上进程同时出现在他们的critical section。 
2. 不要做任何关于CPU速度和数量的假设。 
3. 任何进程在运行到critical section之外时都不能阻塞其他进程。 
4. 不会有进程永远等在critical section之前。

#### 进程切换过程
进程调度函数schedule选择了下一个将占用CPU执行的进程后，将调用进程切换，从而让新的进程得以执行。

两个用户进程，在二者进行进程切换的过程中，具体的步骤如下：
1. 首先在执行某进程A的用户代码时，出现了一个`trap`，这个时候就会从进程A的用户态切换到内核态(过程(1))，并且保存好进程A的trapframe；当内核态处理中断时发现需要进行进程切换时，ucore要通过schedule函数选择下一个将占用CPU执行的进程（即进程B），然后会调用proc_run函数，proc_run函数进一步调用switch_to函数，切换到进程B的内核态(过程(2))，继续进程B上一次在内核态的操作，并通过iret指令，最终将执行权转交给进程B的用户空间(过程(3))。
2. 当进程B由于某种原因发生中断之后(过程(4))，会从进程B的用户态切换到**内核态**，并且保存好进程B的trapframe；当内核态处理中断时发现需要进行进程切换时，即需要切换到进程A，ucore再次切换到进程A(过程(5))，会执行进程A上一次在内核调用schedule函数返回后的下一行代码，这行代码当然还是在进程A的上一次中断处理流程中。最后当进程A的中断处理完毕的时候，执行权又会反交给进程A的用户代码(过程(6))。这就是在只有两个进程的情况下，进程切换间的大体流程。

### 调度框架和调度算法
#### 设计思路
在操作方面，如果需要选择一个就绪进程，就可以从基于某种组织方式的就绪进程集合中选择出一个进程执行。**选择**是在集合中挑选一个“合适”的进程，**出**意味着离开就绪进程集合。

另外考虑到一个处于运行态的进程还会由于某种原因（比如时间片用完了）回到就绪态而不能继续占用CPU执行，这就会重新进入到就绪进程集合中。这两种情况就形成了调度器相关的三个基本操作：**在就绪进程集合中选择**、**进入就绪进程集合**和**离开就绪进程集合**。这三个操作属于调度器的基本操作。

在进程的执行过程中，**就绪进程的等待时间**和**执行进程的执行时间**是影响调度选择的重要因素。这些进程状态变化的情况需要及时让进程调度器知道，便于选择更合适的进程执行。所以这种进程变化的情况就形成了调度器相关的一个变化感知操作：**timer时间事件感知操作**。这样在进程运行或等待的过程中，调度器可以调整进程控制块中与进程调度相关的属性值（比如消耗的时间片、进程优先级等），并可能导致对进程组织形式的调整（比如以时间片大小的顺序来重排双向链表等），并最终可能导致调选择新的进程占用CPU运行。这个操作属于调度器的进程调度属性调整操作。

#### 数据结构
- 在 ucore 中，调度器引入 run-queue（简称rq,即运行队列）的概念，通过链表结构管理进程。
- 由于目前 ucore 设计运行在单CPU上，其内部只有一个全局的运行队列，用来管理系统内全部的进程。
- 运行队列通过链表的形式进行组织。链表的每一个节点是一个list_entry_t,每个list_entry_t 又对应到了`struct proc_struct *`，这其间的转换是通过宏`le2proc`来完成。
- 具体来说，我们知道在`struct proc_struct`中有一个叫`run_link`的`list_entry_t`，因此可以通过偏移量逆向找到对因某个`run_list`的`struct proc_struct`。即进程结构指针`proc = le2proc(链表节点指针, run_link)`。

```
// The introduction of scheduling classes is borrrowed from Linux, and makes the
// core scheduler quite extensible. These classes (the scheduler modules) encapsulate
// the scheduling policies.
struct sched_class {
    // the name of sched_class
    const char *name;
    // 初始化运行队列
    void (*init)(struct run_queue *rq);

    // put the proc into runqueue, and this function must be called with rq_lock
    // 进程放入运行队列
    void (*enqueue)(struct run_queue *rq, struct proc_struct *proc);

    // get the proc out runqueue, and this function must be called with rq_lock
    // 从队列中取出
    void (*dequeue)(struct run_queue *rq, struct proc_struct *proc);

    // choose the next runnable task
    // 选择下一个可运行的任务
    struct proc_struct *(*pick_next)(struct run_queue *rq);

    // dealer of the time-tick
    // 处理tick中断
    void (*proc_tick)(struct run_queue *rq, struct proc_struct *proc);

    /* for SMP support in the future
     *  load_balance
     *     void (*load_balance)(struct rq* rq);
     *  get some proc from this rq, used in load_balance,
     *  return value is the num of gotten proc
     *  int (*get_proc)(struct rq* rq, struct proc* procs_moved[]);
     */
};
```

proc.h 中的 struct proc_struct 中也记录了一些调度相关的信息：
```
struct proc_struct {
    enum proc_state state;                      // Process state
    int pid;                                    // Process ID
    int runs;                                   // the running times of Proces
    uintptr_t kstack;                           // Process kernel stack
    volatile bool need_resched;                 // bool value: need to be rescheduled to release CPU?
    struct proc_struct *parent;                 // the parent process
    struct mm_struct *mm;                       // Process's memory management field
    struct context context;                     // Switch here to run process
    struct trapframe *tf;                       // Trap frame for current interrupt
    uintptr_t cr3;                              // CR3 register: the base addr of Page Directroy Table(PDT)
    uint32_t flags;                             // Process flag
    char name[PROC_NAME_LEN + 1];               // Process name
    list_entry_t list_link;                     // Process link list
    list_entry_t hash_link;                     // Process hash list
    int exit_code;                              // exit code (be sent to parent proc)
    uint32_t wait_state;                        // waiting state
    struct proc_struct *cptr, *yptr, *optr;     // relations between processes
    struct run_queue *rq;                       // running queue contains Process
    list_entry_t run_link;                      // the entry linked in run queue 
    // 该进程的调度链表结构，该结构内部的连接组成了 运行队列 列表

    int time_slice;                             // time slice for occupying the CPU
    // 进程剩余的时间片
    skew_heap_entry_t lab6_run_pool;            // FOR LAB6 ONLY: the entry in the run pool
    //在优先队列中用到的

    uint32_t lab6_stride;                       // FOR LAB6 ONLY: the current stride of the process
    // 步进值

    uint32_t lab6_priority;                     // FOR LAB6 ONLY: the priority of process, set by lab6_set_priority(uint32_t)
    // 优先级

};
```

RR调度算法在`RR_sched_class`调度策略类中实现。
通过数据结构 struct run_queue 来描述完整的 run_queue（运行队列）。它的主要结构如下：
```
struct run_queue {
    //其运行队列的哨兵结构，可以看作是队列头和尾
    list_entry_t run_list;
    //优先队列形式的进程容器，只在 LAB6 中使用
    skew_heap_entry_t *lab6_run_pool;
    //表示其内部的进程总数
    unsigned int proc_num;
    //每个进程一轮占用的最多时间片
    int max_time_slice;
};
```
在 ucore 框架中，运行队列存储的是当前可以调度的进程，所以，只有状态为runnable的进程才能够进入运行队列。当前正在运行的进程并不会在运行队列中。

#### 调度点的相关关键函数
如果我们能够让`wakup_proc`、`schedule`、`run_timer_list`这三个调度相关函数的实现与具体调度算法无关，那么就可以认为ucore实现了一个与调度算法无关的调度框架。

`wakeup_proc`函数完成了把一个就绪进程放入到就绪进程队列中的工作，为此还调用了一个调度类接口函数`sched_class_enqueue`，这使得`wakeup_proc`的实现与具体调度算法无关。
```
void wakeup_proc(struct proc_struct *proc) {
    assert(proc->state != PROC_ZOMBIE);
    bool intr_flag;
    local_intr_save(intr_flag);
    {
        if (proc->state != PROC_RUNNABLE) {
            proc->state = PROC_RUNNABLE;
            proc->wait_state = 0;
            if (proc != current) {
                sched_class_enqueue(proc);
            }
        }
        else {
            warn("wakeup runnable process.\n");
        }
    }
    local_intr_restore(intr_flag);
}
```

`schedule`函数完成了与调度框架和调度算法相关三件事情:
- 把当前继续占用CPU执行的运行进程放放入到就绪进程队列中；
- 从就绪进程队列中选择一个“合适”就绪进程；
- 把这个“合适”的就绪进程从就绪进程队列中取出；
- 如果没有的话，说明现在没有合适的进程可以执行，就执行idle_proc；
- 加了一个runs，表明这个进程运行过几次了；

```
void schedule(void) {
    bool intr_flag;
    struct proc_struct *next;
    local_intr_save(intr_flag);
    {
        current->need_resched = 0;
        if (current->state == PROC_RUNNABLE) {
            sched_class_enqueue(current);
        }
        if ((next = sched_class_pick_next()) != NULL) {
            sched_class_dequeue(next);
        }
        if (next == NULL) {
            next = idleproc;
        }
        next->runs ++;
        if (next != current) {
            proc_run(next);
        }
    }
    local_intr_restore(intr_flag);
}
```

run_time_list在lab6中并没有涉及，是在lab7中的。

通过调用三个调度类接口函数`sched_class_enqueue`、`sched_class_pick_next`、`sched_class_enqueue`来使得完成这三件事情与具体的调度算法无关。`run_timer_list`函数在每次timer中断处理过程中被调用，从而可用来调用调度算法所需的timer时间事件感知操作，调整相关进程的进程调度相关的属性值。通过调用调度类接口函数`sched_class_proc_tick`使得此操作与具体调度算法无关。
这里涉及了一系列调度类接口函数：
- sched_class_enqueue
- sched_class_dequeue
- sched_class_pick_next
- sched_class_proc_tick

这4个函数的实现其实就是调用某基于sched_class数据结构的特定调度算法实现的4个指针函数。采用这样的调度类框架后，如果我们需要实现一个新的调度算法，则我们需要定义一个针对此算法的调度类的实例，一个就绪进程队列的组织结构描述就行了，其他的事情都可交给调度类框架来完成。

#### RR调度算法
RR调度算法的调度思想是让所有runnable态的进程分时轮流使用CPU时间。

RR调度器维护当前runnable进程的有序运行队列。当前进程的时间片用完之后，调度器将当前进程放置到运行队列的尾部，再从其头部取出进程进行调度。

RR调度算法的就绪队列在组织结构上也是一个双向链表，只是增加了一个成员变量，表明在此就绪进程队列中的最大执行时间片。而且在进程控制块proc_struct中增加了一个成员变量time_slice，用来记录进程当前的可运行时间片段。这是由于RR调度算法需要考虑执行进程的运行时间不能太长。在每个timer到时的时候，操作系统会递减当前执行进程的time_slice，当time_slice为0时，就意味着这个进程运行了一段时间（这个时间片段称为进程的时间片），需要把CPU让给其他进程执行，于是操作系统就需要让此进程重新回到rq的队列尾，且重置此进程的时间片为就绪队列的成员变量最大时间片max_time_slice值，然后再从rq的队列头取出一个新的进程执行。

RR_enqueue的函数实现如下表所示。即把某进程的进程控制块指针放入到rq队列末尾，且如果进程控制块的时间片为0，则需要把它重置为rq成员变量max_time_slice。这表示如果进程在当前的执行时间片已经用完，需要等到下一次有机会运行时，才能再执行一段时间。
```
static void RR_enqueue(struct run_queue *rq, struct proc_struct *proc) {
    assert(list_empty(&(proc->run_link)));
    list_add_before(&(rq->run_list), &(proc->run_link));
    if (proc->time_slice == 0 || proc->time_slice > rq->max_time_slice) {
        proc->time_slice = rq->max_time_slice;
    }
    proc->rq = rq;
    rq->proc_num ++;
}
```
RR_pick_next的函数实现如下表所示。即选取就绪进程队列rq中的队头队列元素，并把队列元素转换成进程控制块指针。
```
static struct proc_struct *
RR_pick_next(struct run_queue *rq) {
    list_entry_t *le = list_next(&(rq->run_list));
    if (le != &(rq->run_list)) {
        return le2proc(le, run_link);
    }
    return NULL;
}
```
RR_dequeue的函数实现如下表所示。即把就绪进程队列rq的进程控制块指针的队列元素删除，并把表示就绪进程个数的proc_num减一。
```
static void RR_dequeue(struct run_queue *rq, struct proc_struct *proc) {
    assert(!list_empty(&(proc->run_link)) && proc->rq == rq);
    list_del_init(&(proc->run_link));
    rq->proc_num --;
}
```
RR_proc_tick的函数实现如下表所示。每次timer到时后，trap函数将会间接调用此函数来把当前执行进程的时间片time_slice减一。如果time_slice降到零，则设置此进程成员变量need_resched标识为1，这样在下一次中断来后执行trap函数时，会由于当前进程程成员变量need_resched标识为1而执行schedule函数，从而把当前执行进程放回就绪队列末尾，而从就绪队列头取出在就绪队列上等待时间最久的那个就绪进程执行。
```
static void
RR_proc_tick(struct run_queue *rq, struct proc_struct *proc) {
    if (proc->time_slice > 0) {
        proc->time_slice --;
    }
    if (proc->time_slice == 0) {
        proc->need_resched = 1;
    }
}
```

### Stride Scheduling
#### 基本思路
1. 为每个runnable的进程设置一个当前状态stride，表示该进程当前的调度权，也可以表示这个进程执行了多久了。另外定义其对应的pass值，表示对应进程在调度后，stride 需要进行的累加值。
2. 每次需要调度时，从当前 runnable 态的进程中选择**stride最小**的进程调度。
3. 对于获得调度的进程P，将对应的stride加上其对应的步长pass（只与进程的优先权有关系）。
4. 在一段固定的时间之后，回到2步骤，重新调度当前stride最小的进程。

可以证明，如果令`P.pass =BigStride / P.priority`，其中`P.priority`表示进程的优先权（大于 1），而 BigStride 表示一个预先定义的大常数，则该调度方案为每个进程分配的时间将与其优先级成正比。

将该调度器应用到 ucore 的调度器框架中来，则需要将调度器接口实现如下：

- init:
    - 初始化调度器类的信息（如果有的话）。
    - 初始化当前的运行队列为一个空的容器结构。（比如和RR调度算法一样，初始化为一个有序列表）
- enqueue
    - 初始化刚进入运行队列的进程 proc的stride属性。
    - 将 proc插入放入运行队列中去（注意：这里并不要求放置在队列头部）。
- dequeue
    - 从运行队列中删除相应的元素。
- pick next
    - 扫描整个运行队列，返回其中stride值最小的对应进程。
    - 更新对应进程的stride值，即pass = BIG_STRIDE / P->priority; P->stride += pass。
- proc tick:
    - 检测当前进程是否已用完分配的时间片。如果时间片用完，应该正确设置进程结构的相关标记来引起进程切换。
    - 一个 process 最多可以连续运行 rq.max_time_slice个时间片。

#### 使用优先队列实现 Stride Scheduling
使用优化的优先队列数据结构实现该调度。

优先队列是这样一种数据结构：使用者可以快速的插入和删除队列中的元素，并且在预先指定的顺序下快速取得当前在队列中的最小（或者最大）值及其对应元素。可以看到，这样的数据结构非常符合 Stride 调度器的实现。

libs/skew_heap.h中是优先队列的一个实现。
```
static inline void skew_heap_init(skew_heap_entry_t *a) __attribute__((always_inline));
// 初始化一个队列节点

static inline skew_heap_entry_t *skew_heap_merge(
     skew_heap_entry_t *a, skew_heap_entry_t *b,
     compare_f comp);
// 合并两个优先队列

static inline skew_heap_entry_t *skew_heap_insert(
     skew_heap_entry_t *a, skew_heap_entry_t *b,
     compare_f comp) __attribute__((always_inline));
// 将节点 b 插入至以节点 a 为队列头的队列中去，返回插入后的队列

static inline skew_heap_entry_t *skew_heap_remove(
     skew_heap_entry_t *a, skew_heap_entry_t *b,
     compare_f comp) __attribute__((always_inline));
// 将节点 b 插入从以节点 a 为队列头的队列中去，返回删除后的队列
```
当使用优先队列作为Stride调度器的实现方式之后，运行队列结构也需要作相关改变，其中包括：
1. `struct run_queue`中的`lab6_run_pool`指针，在使用优先队列的实现中表示当前优先队列的头元素，如果优先队列为空，则其指向空指针（NULL）。
2. `struct proc_struct`中的`lab6_run_pool`结构，表示当前进程对应的优先队列节点。本次实验已经修改了系统相关部分的代码，使得其能够很好地适应LAB6新加入的数据结构和接口。而在实验中我们需要做的是用优先队列实现一个正确和高效的Stride调度器，如果用较简略的伪代码描述，则有：

- init(rq):
    - Initialize rq->run_list
    - Set rq->lab6_run_pool to NULL
    - Set rq->proc_num to 0
- enqueue(rq, proc)
    - Initialize proc->time_slice
    - Insert proc->lab6_run_pool into rq->lab6_run_pool
    - rq->proc_num ++
- dequeue(rq, proc)
    - Remove proc->lab6_run_pool from rq->lab6_run_pool
    - rq->proc_num --
- pick_next(rq)
    - If rq->lab6_run_pool == NULL, return NULL
    - Find the proc corresponding to the pointer rq->lab6_run_pool
    - proc->lab6_stride += BIG_STRIDE / proc->lab6_priority
    - Return proc
- proc_tick(rq, proc):
    - If proc->time_slice > 0, proc->time_slice --
    – If proc->time_slice == 0, set the flag proc->need_resched    

## 练习1: 使用 Round Robin 调度算法（不需要编码）
与之前相比，新增了斜堆数据结构的实现；新增了调度算法Round Robin的实现，具体为调用sched.c文件中的`sched_class`的一系列函数，主要有enqueue、dequeue、pick_next等。之后，这些函数进一步调用调度器中的相应函数，默认该调度器为Round Robin调度器，这是在`default_sched.[c|h]`中定义的；新增了set_priority，get_time等函数；

首先在init.c中调用了sched_init函数，在这里把sched_class赋值为default_sched_class，也就是RR，如下：
```
void
sched_init(void) {
    list_init(&timer_list);

    sched_class = &default_sched_class;
    rq = &__rq;
    rq->max_time_slice = MAX_TIME_SLICE;
    sched_class->init(rq);
    cprintf("sched class: %s\n", sched_class->name);
}
```

- RR_init函数：这个函数会被封装为sched_init函数，用于调度算法的初始化，它是在ucore的init.c里面被调用进行初始化，主要完成了计时器list、run_queue的run_list的初始化；
- enqueue函数：将某个进程放入调用算法中的可执行队列中，被封装成sched_class_enqueue函数，这个函数仅在wakeup_proc和schedule函数中被调用，wakeup_proc将某个不是RUNNABLE的进程改成RUNNABLE的并调用enqueue加入可执行队列，而后者是将正在执行的进程换出到可执行队列中去并取出一个可执行进程；
- dequeue函数：将某个在队列中的进程取出，sched_class_dequeue将其封装并在schedule中被调用，将调度算法选择的进程从等待的可执行进程队列中取出；
- pick_next函数：根据调度算法选择下一个要执行的进程，仅在schedule中被调用；
- proc_tick函数：在时钟中断时执行的操作，时间片减一，当时间片为0时，说明这个进程需要重新调度了。仅在进行时间中断的ISR中调用；

> 请理解并分析sched_calss中各个函数指针的用法，并接合Round Robin 调度算法描述ucore的调度执行过程：

- ucore中的调度主要通过schedule和wakeup_proc函数完成，schedule主要把当前执行的进程入队，调用sched_class_pick_next选择下一个执行的进程并将其出队，开始执行。scheduleha函数把当前的进程入队，挑选一个进程将其出队并开始执行。
- 当需要将某一个进程加入就绪进程队列中，需要调用enqueue，将其插入到使用链表组织run_queue的队尾，将这个进程的能够使用的时间片初始化为max_time_slice；
- 当需要将某一个进程从就绪队列中取出，需要调用dequeue，调用list_del_init将其直接删除即可；
- 当需要取出执行的下一个进程时，只需调用pick_next将就绪队列run_queue的队头取出即可；
- 在一个时钟中断中，调用proc_tick将当前执行的进程的剩余可执行时间减1，一旦减到了0，则这个进程的need_resched为1，设成可以被调度的，这样之后就会调用schedule函数将这个进程切换出去；

> 请在实验报告中简要说明如何设计实现”多级反馈队列调度算法“，给出概要设计，鼓励给出详细设计;

1.调度机制：

1. 进程在进入待调度的队列等待时，首先进入优先级最高的Q1等待。
2. 设置多个就绪队列。在系统中设置多个就绪队列，并为每个队列赋予不同的优先级，从第一个开始逐个降低。不同队列进程中所赋予的执行时间也不同，优先级越高，时间片越小。
3. 每个队列都采用FCFS（先来先服务）算法。轮到该进程执行时，若在该时间片内完成，便撤离操作系统，否则调度程序将其转入第二队列的末尾等待调度，.......。若进程最后被调到第N队列中时，便采用RR方式运行。
4. 按队列优先级调度。调度按照优先级最高队列中诸进程运行，仅当第一队列空闲时才调度第二队列进程执行。若低优先级队列执行中有优先级高队列进程执行，应立刻将此进程放入队列末尾，把处理机分配给新到高优先级进程。

- 设置N个多级反馈队列的入口，Q0，Q1，Q2，Q3，...，编号越靠前的队列优先级越低，优先级越低的队列上时间片的长度越大；
- 调用sched_init对调度算法初始化的时候需要同时对N个队列进行初始化；
- 在将进程加入到就绪进程集合的时候，观察这个进程的时间片有没有使用完，如果使用完了，就将所在队列的优先级调低，加入到优先级低一级的队列中去，如果没有使用完时间片，则加入到当前优先级的队列中去；
- 在同一个优先级的队列内使用时间片轮转算法；
- 在选择下一个执行的进程的时候，先考虑更高优先级的队列中是否存在任务，如果不存在在去找较低优先级的队列；
- 从就绪进程集合中删除某一个进程的话直接在对应队列中删除；

## 练习2：实现 Stride Scheduling 调度算法（需要编码）

**啊啊啊忘了在trap.c里改怪不得怎么都搞不对啊啊啊啊啊啊啊啊啊这下子总算有170了！！！**

还是先看看代码里斜堆（skew heap）的实现吧，好多地方要用到这个结构，具体可以在yuhao0102.github.io里仔细看。
在libs/skew.h中定义了skew heap。

猜测这只是一个入口，类似链表那种实现，不包括数据，只有指针。
```
struct skew_heap_entry {
     struct skew_heap_entry *parent, *left, *right;
};
```

`proc_stride_comp_f`函数是用来比较这两个进程的stride的，a比b大返回1，相等返回0，a比b小返回-1。
```
/* The compare function for two skew_heap_node_t's and the
 * corresponding procs*/
static int proc_stride_comp_f(void *a, void *b)
{
     struct proc_struct *p = le2proc(a, lab6_run_pool);
     struct proc_struct *q = le2proc(b, lab6_run_pool);
     int32_t c = p->lab6_stride - q->lab6_stride;
     if (c > 0) return 1;
     else if (c == 0) return 0;
     else return -1;
}
```

这是初始化的函数，把三个指针初始化为NULL
```
static inline void
skew_heap_init(skew_heap_entry_t *a)
{
     a->left = a->right = a->parent = NULL;
}
```

这个是把两个堆merge在一起的操作，强行内联hhh，这个是递归的！
```
static inline skew_heap_entry_t *
skew_heap_merge(skew_heap_entry_t *a, skew_heap_entry_t *b,
                compare_f comp)
{
     if (a == NULL) return b;
     else if (b == NULL) return a;
// 如果a或b有一个为空，则返回另一个

     skew_heap_entry_t *l, *r;
     if (comp(a, b) == -1)
     {
          r = a->left;
          l = skew_heap_merge(a->right, b, comp);

          a->left = l;
          a->right = r;
          if (l) l->parent = a;

          return a;
// 否则判断a和b的值哪个大，如果a比b小，则a的右子树和b合并，a作为堆顶        
     }
     else
     {
          r = b->left;
          l = skew_heap_merge(a, b->right, comp);

          b->left = l;
          b->right = r;
          if (l) 
          	l->parent = b;
	      return b;
// 另一种情况	      
     }
}
```

insert就是把一个单节点的堆跟大堆合并
```
static inline skew_heap_entry_t *
skew_heap_insert(skew_heap_entry_t *a, skew_heap_entry_t *b,
                 compare_f comp)
{
     skew_heap_init(b);
     return skew_heap_merge(a, b, comp);
}
```

删除就是把节点的左右子树进行merge，比较简单，记得删掉这个节点之后补充它的parent即可
```
static inline skew_heap_entry_t *
skew_heap_remove(skew_heap_entry_t *a, skew_heap_entry_t *b,
                 compare_f comp)
{
     skew_heap_entry_t *p   = b->parent;
     skew_heap_entry_t *rep = skew_heap_merge(b->left, b->right, comp);
     if (rep) rep->parent = p;

     if (p)
     {
          if (p->left == b)
               p->left = rep;
          else p->right = rep;
          return a;
     }
     else return rep;
}
```

首先把default_sched.c中设置RR调度器为默认调度器的部分注释掉，然后把default_sched_stride_c改成default_sched_stride.c，这里对默认调度器进行了重新定义。
```
struct sched_class default_sched_class = {
     .name = "stride_scheduler",
     .init = stride_init,
     .enqueue = stride_enqueue,
     .dequeue = stride_dequeue,
     .pick_next = stride_pick_next,
     .proc_tick = stride_proc_tick,
};
```
针对PCB的初始化，代码如下，综合了几个实验的初始化代码，也是一个总结：
```
//LAB4:EXERCISE1 YOUR CODE
/*
 * below fields in proc_struct need to be initialized
 *       enum proc_state state;                      // Process state
 *       int pid;                                    // Process ID
 *       int runs;                                   // the running times of Proces
 *       uintptr_t kstack;                           // Process kernel stack
 *       volatile bool need_resched;                 // bool value: need to be rescheduled to release CPU?
 *       struct proc_struct *parent;                 // the parent process
 *       struct mm_struct *mm;                       // Process's memory management field
 *       struct context context;                     // Switch here to run process
 *       struct trapframe *tf;                       // Trap frame for current interrupt
 *       uintptr_t cr3;                              // CR3 register: the base addr of Page Directroy Table(PDT)
 *       uint32_t flags;                             // Process flag
 *       char name[PROC_NAME_LEN + 1];               // Process name
 */
        proc->state = PROC_UNINIT;
        proc->pid = -1;
        proc->cr3 = boot_cr3;

        proc->runs = 0;
        proc->kstack = 0;
        proc->need_resched = 0;
        proc->parent = NULL;
        proc->mm = NULL;
        memset(&proc->context, 0, sizeof(struct context));
        proc->tf = NULL;
        proc->flags = 0;
        memset(proc->name, 0, PROC_NAME_LEN);

//LAB5 YOUR CODE : (update LAB4 steps)
/*
 * below fields(add in LAB5) in proc_struct need to be initialized
 *       uint32_t wait_state;                        // waiting state
 *       struct proc_struct *cptr, *yptr, *optr;     // relations between processes
 */
        proc->wait_state = 0;
        proc->cptr = proc->optr = proc->yptr = NULL;
//LAB6 YOUR CODE : (update LAB5 steps)
/*
 * below fields(add in LAB6) in proc_struct need to be initialized
 *     struct run_queue *rq;                       // running queue contains Process
 *     list_entry_t run_link;                      // the entry linked in run queue
 *     int time_slice;                             // time slice for occupying the CPU
 *     skew_heap_entry_t lab6_run_pool;            // FOR LAB6 ONLY: the entry in the run pool
 *     uint32_t lab6_stride;                       // FOR LAB6 ONLY: the current stride of the process
 *     uint32_t lab6_priority;                     // FOR LAB6 ONLY: the priority of process, set by lab6_set_priority(uint32_t)
 */
        proc->rq = NULL;
        memset(&proc->run_link, 0, sizeof(list_entry_t));
        proc->time_slice = 0;
        memset(&proc->lab6_run_pool,0,sizeof(skew_heap_entry_t));
        proc->lab6_stride=0;
        proc->lab6_priority=1;
```
主要就是在`vim kern/schedule/default_sched_stride.c`里的修改。
```
#define BIG_STRIDE ((uint32_t)(1<<31)-3)
```
BIG_STRIDE应该设置成小于2^32-1的一个常数。

这个函数用来对run_queue进行初始化等操作
```
/*
 * stride_init initializes the run-queue rq with correct assignment for
 * member variables, including:
 *
 *   - run_list: should be a empty list after initialization.
 *   - lab6_run_pool: NULL
 *   - proc_num: 0
 *   - max_time_slice: no need here, the variable would be assigned by the caller.
 *
 * hint: see libs/list.h for routines of the list structures.
 */
static void
stride_init(struct run_queue *rq) {
     /* LAB6: YOUR CODE
      * (1) init the ready process list: rq->run_list
      * (2) init the run pool: rq->lab6_run_pool
      * (3) set number of process: rq->proc_num to 0
      */
      list_init(&rq->run_list);
      rq->lab6_run_pool = NULL;
      rq->proc_num = 0;
}
```


```

/*
 * stride_enqueue inserts the process ``proc'' into the run-queue
 * ``rq''. The procedure should verify/initialize the relevant members
 * of ``proc'', and then put the ``lab6_run_pool'' node into the
 * queue(since we use priority queue here). The procedure should also
 * update the meta date in ``rq'' structure.
 *
 * proc->time_slice denotes the time slices allocation for the
 * process, which should set to rq->max_time_slice.
 *
 * hint: see libs/skew_heap.h for routines of the priority
 * queue structures.
 */
static void
stride_enqueue(struct run_queue *rq, struct proc_struct *proc) {
     /* LAB6: YOUR CODE
      * (1) insert the proc into rq correctly
      * NOTICE: you can use skew_heap or list. Important functions
      *         skew_heap_insert: insert a entry into skew_heap
      *         list_add_before: insert  a entry into the last of list
      * (2) recalculate proc->time_slice
      * (3) set proc->rq pointer to rq
      * (4) increase rq->proc_num
      */
      rq->lab6_run_pool = skew_heap_insert(rq->lab6_run_pool, &proc->lab6_run_pool, proc_stride_comp_f);
      // 做插入操作，把这个进程插到run_pool里。
      if(proc->time_slice == 0 || proc->time_slice > rq->max_time_slice) {
          proc->time_slice = rq->max_time_slice;
      }
      // 如果这个进程的时间片不符合要求，就把它初始化成最大值。
      proc->rq = rq;
      rq->proc_num ++;
      //run_queue里的进程数++
}
```

做删除操作，把这个进程从run_pool里删除，并且将run_queue里的进程数减一。
```
/*
 * stride_dequeue removes the process ``proc'' from the run-queue
 * ``rq'', the operation would be finished by the skew_heap_remove
 * operations. Remember to update the ``rq'' structure.
 *
 * hint: see libs/skew_heap.h for routines of the priority
 * queue structures.
 */
static void
stride_dequeue(struct run_queue *rq, struct proc_struct *proc) {
     /* LAB6: YOUR CODE
      * (1) remove the proc from rq correctly
      * NOTICE: you can use skew_heap or list. Important functions
      *         skew_heap_remove: remove a entry from skew_heap
      *         list_del_init: remove a entry from the  list
      */
      rq->lab6_run_pool = skew_heap_remove(rq->lab6_run_pool, &proc->lab6_run_pool, proc_stride_comp_f);
      rq->proc_num --;
}
```

pick_next从run_queue中选择stride值最小的进程，即斜堆的根节点对应的进程，并且返回这个proc，同时更新这个proc的stride
```
/*
 * stride_pick_next pick the element from the ``run-queue'', with the
 * minimum value of stride, and returns the corresponding process
 * pointer. The process pointer would be calculated by macro le2proc,
 * see kern/process/proc.h for definition. Return NULL if
 * there is no process in the queue.
 *
 * When one proc structure is selected, remember to update the stride
 * property of the proc. (stride += BIG_STRIDE / priority)
 *
 * hint: see libs/skew_heap.h for routines of the priority
 * queue structures.
 */
static struct proc_struct *
stride_pick_next(struct run_queue *rq) {
     /* LAB6: YOUR CODE
      * (1) get a  proc_struct pointer p  with the minimum value of stride
             (1.1) If using skew_heap, we can use le2proc get the p from rq->lab6_run_poll
             (1.2) If using list, we have to search list to find the p with minimum stride value
      * (2) update p;s stride value: p->lab6_stride
      * (3) return p
      */
      if (rq->lab6_run_pool == NULL)
          return NULL;
      struct proc_struct *p = le2proc(rq->lab6_run_pool, lab6_run_pool);
      p->lab6_stride += BIG_STRIDE/p->lab6_priority;
      return p;
}
```

要在trap的时候调用！！！！如果这个proc的时间片还有的话，就减一，如果这个时间片为0了，就把它设成可调度的，参与调度。
```
/*
 * stride_proc_tick works with the tick event of current process. You
 * should check whether the time slices for current process is
 * exhausted and update the proc struct ``proc''. proc->time_slice
 * denotes the time slices left for current
 * process. proc->need_resched is the flag variable for process
 * switching.
 */
static void
stride_proc_tick(struct run_queue *rq, struct proc_struct *proc) {
     /* LAB6: YOUR CODE */
        if (proc->time_slice > 0) {
                proc->time_slice --;
        }
        if (proc->time_slice == 0) {
                proc->need_resched = 1;
        }
}
```