# 实验七：同步互斥
## 实验目的
- 理解操作系统的同步互斥的设计实现；
- 理解底层支撑技术：禁用中断、定时器、等待队列；
- 在ucore中理解信号量（semaphore）机制的具体实现；
- 理解管程机制，在ucore内核中增加基于管程（monitor）的条件变量（condition variable）的支持；
- 了解经典进程同步问题，并能使用同步机制解决进程同步问题。

## 实验内容
lab6已经可以调度运行多个进程，如果多个进程需要协同操作或访问共享资源，则存在如何同步和有序竞争的问题。本次实验，主要是熟悉ucore的进程同步机制—信号量（semaphore）机制，以及基于信号量的哲学家就餐问题解决方案。然后掌握管程的概念和原理，并参考信号量机制，实现基于管程的条件变量机制和基于条件变量来解决哲学家就餐问题。

在本次实验中，在kern/sync/check_sync.c中提供了一个基于信号量的哲学家就餐问题解法。同时还需完成练习，即实现基于管程（主要是灵活运用条件变量和互斥信号量）的哲学家就餐问题解法。

哲学家就餐问题描述如下：有五个哲学家，他们的生活方式是交替地进行思考和进餐。哲学家们公用一张圆桌，周围放有五把椅子，每人坐一把。在圆桌上有五个碗和五根筷子，当一个哲学家思考时，他不与其他人交谈，饥饿时便试图取用其左、右最靠近他的筷子，但他可能一根都拿不到。只有在他拿到两根筷子时，方能进餐，进餐完后，放下筷子又继续思考。

## 同步互斥的设计与实现
### 实验执行流程概述
**互斥**是指某一资源同时只允许一个进程对其进行访问，具有**唯一性**和**排它性**，但**互斥不用限制进程对资源的访问顺序**，即访问可以是无序的。**同步**是指在进程间的执行必须严格**按照规定的某种先后次序来运行**，即访问是有序的，这种先后次序取决于要系统完成的任务需求。在进程写资源情况下，进程间要求满足互斥条件。在进程读资源情况下，可允许多个进程同时访问资源。

实验七设计实现了多种同步互斥手段，包括时钟中断管理、等待队列、信号量、管程机制（包含条件变量设计）等，并基于信号量实现了哲学家问题的执行过程。而本次实验的练习是要求用管程机制实现哲学家问题的执行过程。在实现信号量机制和管程机制时，需要让无法进入临界区的进程睡眠，为此在ucore中设计了等待队列wait_queue。当进程无法进入临界区（即无法获得信号量）时，可让进程进入等待队列，这时的进程处于等待状态（也可称为阻塞状态），从而会让实验六中的调度器选择一个处于就绪状态（即RUNNABLE_STATE）的进程，进行进程切换，让新进程有机会占用CPU执行，从而让整个系统的运行更加高效。

lab7/kern/sync/check_sync.c中的check_sync函数可以理解为是实验七的起始执行点，是实验七的总控函数。进一步分析此函数，可以看到这个函数主要分为了两个部分，第一部分是实现基于信号量的哲学家问题，第二部分是实现基于管程的哲学家问题。
- 对于check_sync函数的第一部分，首先实现初始化了一个互斥信号量，然后创建了对应5个哲学家行为的5个信号量，并创建5个内核线程代表5个哲学家，每个内核线程完成了基于信号量的哲学家吃饭睡觉思考行为实现。
- 对于check_sync函数的第二部分，首先初始化了管程，然后又创建了5个内核线程代表5个哲学家，每个内核线程要完成基于管程的哲学家吃饭、睡觉、思考的行为实现。

### 同步互斥的底层支撑
由于调度的存在，且进程在访问某类资源暂时无法满足的情况下会进入等待状态，导致了多进程执行时序的不确定性和潜在执行结果的不确定性。为了确保执行结果的正确性，本试验需要设计更加完善的进程等待和互斥的底层支撑机制，确保能正确提供基于信号量和条件变量的同步互斥机制。

由于有定时器、屏蔽/使能中断、等待队列wait_queue支持test_and_set_bit等原子操作机器指令（在本次实验中没有用到）的存在，使得我们在实现进程等待、同步互斥上得到了极大的简化。下面将对定时器、屏蔽/使能中断和等待队列进行进一步讲解。

#### 定时器
在传统的操作系统中，定时器提供了基于时间事件的调度机制。在ucore中，两次时间中断之间的时间间隔为一个时间片，timer splice。

基于此时间单位，操作系统得以向上提供基于时间点的事件，并实现基于时间长度的睡眠等待和唤醒机制。在每个时钟中断发生时，操作系统产生对应的时间事件。

sched.h, sched.c定义了有关timer的各种相关接口来使用 timer 服务，其中主要包括:
- `typedef struct {……} timer_t`：定义了 timer_t 的基本结构，其可以用 sched.h 中的timer_init函数对其进行初始化。
- `void timer_init(timer t *timer, struct proc_struct *proc, int expires)`: 对某定时器进行初始化，让它在expires时间片之后唤醒proc进程。
- `void add_timer(timer t *timer)`：向系统添加某个初始化过的timer_t，该定时器在指定时间后被激活，并将对应的进程唤醒至runnable（如果当前进程处在等待状态）。
- `void del_timer(timer_t *time)`：向系统删除（或者说取消）某一个定时器。该定时器在取消后不会被系统激活并唤醒进程。
- `void run_timer_list(void)`：更新当前系统时间点，遍历当前所有处在系统管理内的定时器，找出所有应该激活的计数器，并激活它们。该过程在且只在每次定时器中断时被调用。在ucore中，其还会调用调度器事件处理程序。

一个 timer_t 在系统中的存活周期可以被描述如下：
- timer_t在某个位置被创建和初始化，并通过add_timer加入系统管理列表中；
- 系统时间被不断累加，直到 run_timer_list 发现该 timer_t到期；
- run_timer_list更改对应的进程状态，并从系统管理列表中移除该timer_t；

#### 屏蔽与使能中断
之前用过，这里简单看看。

在ucore中提供的底层机制包括中断屏蔽/使能控制等。`kern/sync.c`有开关中断的控制函数`local_intr_save(x)`和`local_intr_restore(x)`，它们是基于`kern/driver`文件下的`intr_enable()`、`intr_disable()`函数实现的。具体调用关系为：
> 关中断：`local_intr_save` --> `__intr_save` --> `intr_disable` --> `cli`
> 开中断：`local_intr_restore` --> `__intr_restore` --> `intr_enable` --> `sti`

最终的cli和sti是x86的机器指令，最终实现了关（屏蔽）中断和开（使能）中断，即设置了eflags寄存器中与中断相关的位。通过关闭中断，可以防止对当前执行的控制流被其他中断事件处理所打断。既然不能中断，那也就意味着在内核运行的当前进程无法被打断或被重新调度，即实现了对临界区的互斥操作。所以在单处理器情况下，可以通过开关中断实现对临界区的互斥保护，需要互斥的临界区代码的一般写法为：
```
local_intr_save(intr_flag);
{
  临界区代码
}
local_intr_restore(intr_flag);
```
但是，在多处理器情况下，这种方法是无法实现互斥的，因为屏蔽了一个CPU的中断，只能阻止本地CPU上的进程不会被中断或调度，并不意味着其他CPU上执行的进程不能执行临界区的代码。所以，开关中断只对单处理器下的互斥操作起作用。

#### 等待队列
在课程中提到用户进程或内核线程可以转入等待状态以等待某个特定事件（比如睡眠,等待子进程结束,等待信号量等），当该事件发生时这些进程能够被再次唤醒。内核实现这一功能的一个底层支撑机制就是**等待队列wait_queue**，等待队列和每一个事件（睡眠结束、时钟到达、任务完成、资源可用等）联系起来。需要等待事件的进程在转入休眠状态后插入到等待队列中。当事件发生之后，内核遍历相应等待队列，唤醒休眠的用户进程或内核线程，并设置其状态为就绪状态（PROC_RUNNABLE），并将该进程从等待队列中清除。

ucore在`kern/sync/{ wait.h, wait.c }`中实现了等待项wait结构和等待队列wait queue结构以及相关函数），这是实现ucore中的信号量机制和条件变量机制的基础，进入wait queue的进程会被设为等待状态（PROC_SLEEPING），直到他们被唤醒。

数据结构定义
```
typedef  struct {
    struct proc_struct *proc;     //等待进程的指针
    uint32_t wakeup_flags;        //进程被放入等待队列的原因标记
    wait_queue_t *wait_queue;     //指向此wait结构所属于的wait_queue
    list_entry_t wait_link;       //用来组织wait_queue中wait节点的连接
} wait_t;

typedef struct {
    list_entry_t wait_head;       //wait_queue的队头
} wait_queue_t;

le2wait(le, member)               //实现wait_t中成员的指针向wait_t 指针的转化
```
相关函数说明
与wait和wait queue相关的函数主要分为两层，底层函数是对wait queue的初始化、插入、删除和查找操作，相关函数如下：

`wait_init`：初始化wait结构，将放入等待队列的原因标记设置为WT_INTERRUPTED，意为可以被打断等待状态
```
void
wait_init(wait_t *wait, struct proc_struct *proc) {
    wait->proc = proc;
    wait->wakeup_flags = WT_INTERRUPTED;
    list_init(&(wait->wait_link));
}
```

`wait_in_queue`：wait是否在wait queue中
```
bool
wait_in_queue(wait_t *wait) {
    return !list_empty(&(wait->wait_link));
}
```

`wait_queue_init`：初始化wait_queue结构
```
void
wait_queue_init(wait_queue_t *queue) {
    list_init(&(queue->wait_head));
}
```

`wait_queue_add`：设置当前等待项wait的等待队列，并把wait前插到wait queue中
```
void
wait_queue_add(wait_queue_t *queue, wait_t *wait) {
    assert(list_empty(&(wait->wait_link)) && wait->proc != NULL);
    wait->wait_queue = queue;
    list_add_before(&(queue->wait_head), &(wait->wait_link));
}
```

`wait_queue_del`：从wait queue中删除wait
```
void
wait_queue_del(wait_queue_t *queue, wait_t *wait) {
    assert(!list_empty(&(wait->wait_link)) && wait->wait_queue == queue);
    list_del_init(&(wait->wait_link));
}
```

`wait_queue_next`：取得wait_queue中wait等待项的后一个链接指针
```
wait_t *
wait_queue_next(wait_queue_t *queue, wait_t *wait) {
    assert(!list_empty(&(wait->wait_link)) && wait->wait_queue == queue);
    list_entry_t *le = list_next(&(wait->wait_link));
    if (le != &(queue->wait_head)) {
        return le2wait(le, wait_link);
    }
    return NULL;
}
```

`wait_queue_prev`：取得wait_queue中wait等待项的前一个链接指针
```
wait_t *
wait_queue_prev(wait_queue_t *queue, wait_t *wait) {
    assert(!list_empty(&(wait->wait_link)) && wait->wait_queue == queue);
    list_entry_t *le = list_prev(&(wait->wait_link));
    if (le != &(queue->wait_head)) {
        return le2wait(le, wait_link);
    }
    return NULL;
}
```

`wait_queue_first`：取得wait queue的第一个wait
```
wait_t *
wait_queue_first(wait_queue_t *queue) {
    list_entry_t *le = list_next(&(queue->wait_head));
    if (le != &(queue->wait_head)) {
        return le2wait(le, wait_link);
    }
    return NULL;
}
```

`wait_queue_last`：取得wait queue的最后一个wait
```
wait_t *
wait_queue_last(wait_queue_t *queue) {
    list_entry_t *le = list_prev(&(queue->wait_head));
    if (le != &(queue->wait_head)) {
        return le2wait(le, wait_link);
    }
    return NULL;
}
```

`bool wait_queue_empty`：wait queue是否为空
```
bool
wait_queue_empty(wait_queue_t *queue) {
    return list_empty(&(queue->wait_head));
}
```

高层函数基于底层函数实现了让进程进入等待队列--wait_current_set，以及从等待队列中唤醒进程--wakeup_wait，相关函数如下：

`wait_current_set`：进程进入等待队列，当前进程的状态设置成睡眠
```
void
wait_current_set(wait_queue_t *queue, wait_t *wait, uint32_t wait_state) {
    assert(current != NULL);
    wait_init(wait, current);
    current->state = PROC_SLEEPING;
    current->wait_state = wait_state;
    wait_queue_add(queue, wait);
}
```

`wait_current_del`：把与当前进程关联的wait从等待队列queue中删除
```
#define wait_current_del(queue, wait)                                       \
    do {                                                                    \
        if (wait_in_queue(wait)) {                                          \
            wait_queue_del(queue, wait);                                    \
        }                                                                   \
    } while (0)
```

`wakeup_wait`：唤醒等待队列上的wait所关联的进程
```
void
wakeup_wait(wait_queue_t *queue, wait_t *wait, uint32_t wakeup_flags, bool del) {
    if (del) {
        wait_queue_del(queue, wait);
    }
    wait->wakeup_flags = wakeup_flags;
    wakeup_proc(wait->proc);
}
```

`void wakeup_first`：唤醒等待队列上第一个的等待的进程
```
void
wakeup_first(wait_queue_t *queue, uint32_t wakeup_flags, bool del) {
    wait_t *wait;
    if ((wait = wait_queue_first(queue)) != NULL) {
        wakeup_wait(queue, wait, wakeup_flags, del);
    }
}
```

`wakeup_queue`：唤醒等待队列上的所有等待进程
```
void
wakeup_queue(wait_queue_t *queue, uint32_t wakeup_flags, bool del) {
    wait_t *wait;
    if ((wait = wait_queue_first(queue)) != NULL) {
        if (del) {
            do {
                wakeup_wait(queue, wait, wakeup_flags, 1);
            } while ((wait = wait_queue_first(queue)) != NULL);
        }
        else {
            do {
                wakeup_wait(queue, wait, wakeup_flags, 0);
            } while ((wait = wait_queue_next(queue, wait)) != NULL);
        }
    }
}
```

### 信号量
信号量是一种同步互斥机制的实现，普遍存在于现在的各种操作系统内核里。相对于spinlock 的应用对象，信号量的应用对象是在临界区中运行的时间较长的进程。等待信号量的进程需要睡眠来减少占用 CPU 的开销。
```
struct semaphore {
	int count;
	queueType queue;
};
void semWait(semaphore s)
{
	s.count--;
	if (s.count < 0) {
		/* place this process in s.queue */;
		/* block this process */;
	}
}
void semSignal(semaphore s)
{
	s.count++;
	if (s.count<= 0) {
		/* remove a process P from s.queue */;
		/* place process P on ready list */;
	}
}
```
基于上诉信号量实现可以认为，当多个（>1）进程可以进行互斥或同步合作时，一个进程会由于无法满足信号量设置的某条件而在某一位置停止，直到它接收到一个特定的信号（表明条件满足了）。为了发信号，需要使用一个称作信号量的特殊变量。为通过信号量s传送信号，信号量的V操作采用进程可执行原语semSignal(s)；为通过信号量s接收信号，信号量的P操作采用进程可执行原语semWait(s)；如果相应的信号仍然没有发送，则进程被阻塞或睡眠，直到发送完为止。
ucore中信号量参照上述原理描述，建立在开关中断机制和wait_queue的基础上进行了具体实现。信号量的数据结构定义如下：
```
typedef struct {
    int value;                   //信号量的当前值
    wait_queue_t wait_queue;     //信号量对应的等待队列
} semaphore_t;
```
`semaphore_t`是最基本的记录型信号量（record semaphore)结构，包含了用于计数的整数值value，和一个进程等待队列wait_queue，一个等待的进程会挂在此等待队列上。

在ucore中最重要的信号量操作是P操作函数`down(semaphore_t *sem)`和V操作函数`up(semaphore_t *sem)`。但这两个函数的具体实现是`__down(semaphore_t *sem, uint32_t wait_state)`函数和`__up(semaphore_t *sem, uint32_t wait_state)`函数，二者的具体实现描述如下：

`__down(semaphore_t *sem, uint32_t wait_state, timer_t *timer)`：具体实现信号量的P操作，首先关掉中断，然后判断当前信号量的value是否大于0。如果是>0，则表明可以获得信号量，故让value减一，并打开中断返回即可；如果不是>0，则表明无法获得信号量，故需要将当前的进程加入到等待队列中，并打开中断，然后运行调度器选择另外一个进程执行。如果被V操作唤醒，则把自身关联的wait从等待队列中删除（此过程需要先关中断，完成后开中断）。

`__up(semaphore_t *sem, uint32_t wait_state)`：具体实现信号量的V操作，首先关中断，如果信号量对应的wait queue中没有进程在等待，直接把信号量的value加一，然后开中断返回；如果有进程在等待且进程等待的原因是semophore设置的，则调用wakeup_wait函数将waitqueue中等待的第一个wait删除，且把此wait关联的进程唤醒，最后开中断返回。

对照信号量的原理性描述和具体实现，可以发现二者在流程上基本一致，只是具体实现采用了关中断的方式保证了对共享资源的互斥访问，通过等待队列让无法获得信号量的进程睡眠等待。另外，我们可以看出信号量的计数器value具有有如下性质：
- value>0，表示共享资源的空闲数
- vlaue<0，表示该信号量的等待队列里的进程数
- value=0，表示等待队列为空

### 管程和条件变量
#### 原理回顾
引入了管程是为了将对共享资源的所有访问及其所需要的同步操作集中并封装起来。Hansan为管程所下的定义：“一个管程定义了一个数据结构和能为并发进程所执行（在该数据结构上）的一组操作，这组操作能同步进程和改变管程中的数据”。有上述定义可知，管程由四部分组成：
- 管程内部的共享变量；
- 管程内部的条件变量；
- 管程内部并发执行的进程；
- 对局部于管程内部的共享数据设置初始值的语句。

局限在管程中的数据结构，只能被局限在管程的操作过程所访问，任何管程之外的操作过程都不能访问它；另一方面，局限在管程中的操作过程也主要访问管程内的数据结构。由此可见，管程相当于一个**隔离区**，它把共享变量和对它进行操作的若干个过程围了起来，所有进程要访问临界资源时，都必须经过管程才能进入，而管程每次只允许一个进程进入管程，从而需要确保进程之间互斥。

但在管程中仅仅有互斥操作是不够用的。进程可能需要等待某个条件Cond为真才能继续执行。如果采用忙等(busy waiting)方式：
```
while not( Cond ) do {}
```
在单处理器情况下，将会导致所有其它进程都无法进入临界区使得该条件Cond为真，该管程的执行将会发生死锁。为此，可引入条件变量（Condition Variables，简称CV）。一个条件变量CV可理解为一个进程的等待队列，队列中的进程正等待某个条件Cond变为真。每个条件变量关联着一个条件，如果条件Cond不为真，则进程需要等待，如果条件Cond为真，则进程可以进一步在管程中执行。需要注意当一个进程等待一个条件变量CV（即等待Cond为真），该进程需要退出管程，这样才能让其它进程可以进入该管程执行，并进行相关操作，比如设置条件Cond为真，改变条件变量的状态，并唤醒等待在此条件变量CV上的进程。因此对条件变量CV有两种主要操作：
- wait_cv： 被一个进程调用，以等待断言Pc被满足后该进程可恢复执行. 进程挂在该条件变量上等待时，不被认为是占用了管程。
- signal_cv：被一个进程调用，以指出断言Pc现在为真，从而可以唤醒等待断言Pc被满足的进程继续执行。

"哲学家就餐"实例
有了互斥和信号量支持的管程就可用用了解决各种同步互斥问题。“用管程解决哲学家就餐问题”如下：
```
monitor dp
{
    enum {THINKING, HUNGRY, EATING} state[5];
    condition self[5];

    void pickup(int i) {
        state[i] = HUNGRY;
        test(i);
        if (state[i] != EATING)
            self[i].wait_cv();
    }

    void putdown(int i) {
        state[i] = THINKING;
        test((i + 4) % 5);
        test((i + 1) % 5);
    }

    void test(int i) {
        if ((state[(i + 4) % 5] != EATING) &&
           (state[i] == HUNGRY) &&
           (state[(i + 1) % 5] != EATING)) {
              state[i] = EATING;
              self[i].signal_cv();
        }
    }

    initialization code() {
        for (int i = 0; i < 5; i++)
        state[i] = THINKING;
        }
}
```

#### 关键数据结构
虽然大部分教科书上说明管程适合在语言级实现比如java等高级语言，没有提及在采用C语言的OS中如何实现。下面我们将要尝试在ucore中用C语言实现采用基于互斥和条件变量机制的管程基本原理。
ucore中的管程机制是基于信号量和条件变量来实现的。ucore中的管程的数据结构monitor_t定义如下：
```
typedef struct monitor{
    semaphore_t mutex;      // the mutex lock for going into the routines in monitor, should be initialized to 1
    // the next semaphore is used to 
    //    (1) procs which call cond_signal funciton should DOWN next sema after UP cv.sema
    // OR (2) procs which call cond_wait funciton should UP next sema before DOWN cv.sema
    semaphore_t next;        
    int next_count;         // the number of of sleeped procs which cond_signal funciton
    condvar_t *cv;          // the condvars in monitor
} monitor_t;
```

管程中的成员变量mutex是一个二值信号量，是实现每次只允许一个进程进入管程的关键元素，确保了互斥访问性质。管程中的条件变量cv通过执行wait_cv，会使得等待某个条件Cond为真的进程能够离开管程并睡眠，且让其他进程进入管程继续执行；而进入管程的某进程设置条件Cond为真并执行signal_cv时，能够让等待某个条件Cond为真的睡眠进程被唤醒，从而继续进入管程中执行。

注意：管程中的成员变量信号量next和整型变量next_count是配合进程对条件变量cv的操作而设置的，这是由于发出signal_cv的进程A会唤醒由于wait_cv而睡眠的进程B，由于管程中只允许一个进程运行，所以进程B执行会导致唤醒进程B的进程A睡眠，直到进程B离开管程，进程A才能继续执行，这个同步过程是通过信号量next完成的；而next_count表示了由于发出singal_cv而睡眠的进程个数。
管程中的条件变量的数据结构condvar_t定义如下：
```
typedef struct condvar{
    semaphore_t sem;     // the sem semaphore is used to down the waiting proc, and the signaling proc should up the waiting proc
    int count;       　    // the number of waiters on condvar
    monitor_t * owner;     // the owner(monitor) of this condvar
} condvar_t;
```

条件变量的定义中也包含了一系列的成员变量，信号量sem用于让发出wait_cv操作的等待某个条件Cond为真的进程睡眠，而让发出signal_cv操作的进程通过这个sem来唤醒睡眠的进程。count表示等在这个条件变量上的睡眠进程的个数。owner表示此条件变量的宿主是哪个管程。

#### 条件变量的signal和wait的设计
理解了数据结构的含义后，我们就可以开始管程的设计实现了。ucore设计实现了条件变量`wait_cv`操作和`signal_cv`操作对应的具体函数，即`cond_wait`函数和`cond_signal`函数，此外还有`cond_init`初始化函数。

首先来看wait_cv的原理实现：
```
cv.count++;
if(monitor.next_count > 0)
   sem_signal(monitor.next);
else
   sem_signal(monitor.mutex);
sem_wait(cv.sem);
cv.count -- ;
```
对照着可分析出cond_wait函数的具体执行过程。可以看出如果进程A执行了cond_wait函数，表示此进程等待某个条件Cond不为真，需要睡眠。因此表示等待此条件的睡眠进程个数cv.count要加一。接下来会出现两种情况。
情况一：如果monitor.next_count如果大于0，表示有大于等于1个进程执行cond_signal函数且睡了，就睡在了monitor.next信号量上（假定这些进程挂在monitor.next信号量相关的等待队列Ｓ上），因此需要唤醒等待队列Ｓ中的一个进程B；然后进程A睡在cv.sem上。如果进程A醒了，则让cv.count减一，表示等待此条件变量的睡眠进程个数少了一个，可继续执行了！

这里隐含这一个现象，即某进程A在时间顺序上先执行了cond_signal，而另一个进程B后执行了cond_wait，这会导致进程A没有起到唤醒进程B的作用。
问题: 在cond_wait有sem_signal(mutex)，但没有看到哪里有sem_wait(mutex)，这好像没有成对出现，是否是错误的？ 答案：其实在管程中的每一个函数的入口处会有wait(mutex)，这样二者就配好对了。
情况二：如果monitor.next_count如果小于等于0，表示目前没有进程执行cond_signal函数且睡着了，那需要唤醒的是由于互斥条件限制而无法进入管程的进程，所以要唤醒睡在monitor.mutex上的进程。然后进程A睡在cv.sem上，如果睡醒了，则让cv.count减一，表示等待此条件的睡眠进程个数少了一个，可继续执行了！
然后来看signal_cv的原理实现：
```
if( cv.count > 0) {
   monitor.next_count ++;
   sem_signal(cv.sem);
   sem_wait(monitor.next);
   monitor.next_count -- ;
}
```
对照着可分析出cond_signal函数的具体执行过程。首先进程B判断cv.count，如果不大于0，则表示当前没有执行cond_wait而睡眠的进程，因此就没有被唤醒的对象了，直接函数返回即可；如果大于0，这表示当前有执行cond_wait而睡眠的进程A，因此需要唤醒等待在cv.sem上睡眠的进程A。由于只允许一个进程在管程中执行，所以一旦进程B唤醒了别人（进程A），那么自己就需要睡眠。故让monitor.next_count加一，且让自己（进程B）睡在信号量monitor.next上。如果睡醒了，这让monitor.next_count减一。

#### 管程中函数的入口出口设计
为了让整个管程正常运行，还需在管程中的每个函数的入口和出口增加相关操作，即：
```
function_in_monitor （…）
{
  sem.wait(monitor.mutex);
//-----------------------------
  the real body of function;
//-----------------------------
  if(monitor.next_count > 0)
     sem_signal(monitor.next);
  else
     sem_signal(monitor.mutex);
}
```
这样带来的作用有两个，（1）只有一个进程在执行管程中的函数。（2）避免由于执行了cond_signal函数而睡眠的进程无法被唤醒。对于第二点，如果进程A由于执行了cond_signal函数而睡眠（这会让monitor.next_count大于0，且执行sem_wait(monitor.next)），则其他进程在执行管程中的函数的出口，会判断monitor.next_count是否大于0，如果大于0，则执行sem_signal(monitor.next)，从而执行了cond_signal函数而睡眠的进程被唤醒。上诉措施将使得管程正常执行。

## 练习1：理解内核级信号量的实现和基于内核级信号量的哲学家就餐问题
首先把trap.c中处理时钟中断的时候调用的sched_class_proc_tick函数替换为run_timer_list函数（后者中已经包括了前者），用于支持定时器机制；

在sem.c定义了内核级信号量机制的函数，先来学习这个文件。sem.h中是定义，这个semphore_t结构体就是信号量的定义了。里边有一个value和一个队列。
```
#ifndef __KERN_SYNC_SEM_H__
#define __KERN_SYNC_SEM_H__

#include <defs.h>
#include <atomic.h>
#include <wait.h>

typedef struct {
    int value;
    wait_queue_t wait_queue;
} semaphore_t;

void sem_init(semaphore_t *sem, int value);
void up(semaphore_t *sem);
void down(semaphore_t *sem);
bool try_down(semaphore_t *sem);

#endif /* !__KERN_SYNC_SEM_H__ */
```
sem_init对信号量进行初始化，信号量包括了一个整型数值变量和一个等待队列，该函数将该变量设置为指定的初始值（有几个资源），并且将等待队列初始化即可；wait_queue_init是把这个队列初始化。
```
void
sem_init(semaphore_t *sem, int value) {
    sem->value = value;
    wait_queue_init(&(sem->wait_queue));
}

void
wait_queue_init(wait_queue_t *queue) {
    list_init(&(queue->wait_head));
}
```

`__up`: 这个函数是**释放**一个该信号量对应的资源，如果它的等待队列中没有等待的请求，则直接把资源数加一，返回即可；如果在等待队列上有等在这个信号量上的进程，则调用`wakeup_wait`将其唤醒执行；在函数中禁用了中断，保证了操作的原子性，函数中操作的具体流程为：
```
static __noinline void __up(semaphore_t *sem, uint32_t wait_state) {
    bool intr_flag;
    local_intr_save(intr_flag);
    {
        wait_t *wait;
        //查询等待队列是否为空
        if ((wait = wait_queue_first(&(sem->wait_queue))) == NULL) {
            sem->value ++;
            //如果是空的话，没有等待的线程，给整型变量加1；
        }
        else {
        	//如果等待队列非空，有等待的线程，取出其中的一个进程唤醒；
            assert(wait->proc->wait_state == wait_state);
            wakeup_wait(&(sem->wait_queue), wait, wait_state, 1);
            //这个函数找到等待的线程并唤醒
        }
    }
    local_intr_restore(intr_flag);
}
```

`__down`: 是原理课中的P操作，表示请求一个该信号量对应的资源，同样禁用中断，保证原子性。首先查询整型变量看是否大于0，如果大于0则表示存在可分配的资源，整型变量减1，直接返回；如果整型变量小于等于0，表示没有可用的资源，那么当前进程的需求得不到满足，因此在`wait_current_set`中将其状态改为SLEEPING态，然后调用`wait_queue_add`将其挂到对应信号量的等待队列中，调用schedule函数进行调度，让出CPU，在资源得到满足，重新被唤醒之后，将自身从等待队列上删除掉；
```
static __noinline uint32_t __down(semaphore_t *sem, uint32_t wait_state) {
    bool intr_flag;
    local_intr_save(intr_flag);
    if (sem->value > 0) {
        sem->value --;
        local_intr_restore(intr_flag);
        return 0;
    }
    wait_t __wait, *wait = &__wait;
    wait_current_set(&(sem->wait_queue), wait, wait_state);
    // 挂起这个等待线程并加入等待队列
    local_intr_restore(intr_flag);

    schedule();

    local_intr_save(intr_flag);
    wait_current_del(&(sem->wait_queue), wait);
    local_intr_restore(intr_flag);
// 有可能当前线程被唤醒的原因跟之前等待的原因不一致
// 要把原因返回，由高层判断是否是合理状态。
    if (wait->wakeup_flags != wait_state) {
        return wait->wakeup_flags;
    }
    return 0;
}

void
wait_current_set(wait_queue_t *queue, wait_t *wait, uint32_t wait_state) {
    assert(current != NULL);
    wait_init(wait, current);
    current->state = PROC_SLEEPING;
    current->wait_state = wait_state;
    wait_queue_add(queue, wait);
}
```
`try_down`: 简化版的P操作，如果资源数大于0则分配，资源数小于0也不进入等待队列，即使获取资源失败也不会堵塞当前进程；
```
bool try_down(semaphore_t *sem) {
    bool intr_flag, ret = 0;
    local_intr_save(intr_flag);
    if (sem->value > 0) {
        sem->value --, ret = 1;
    }
    local_intr_restore(intr_flag);
    return ret;
}
```

请在实验报告中给出给用户态进程/线程提供信号量机制的设计方案，并比较说明给内核级提供信号量机制的异同。
> 用于保证操作原子性的禁用中断机制、以及CPU提供的Test and Set指令机制都只能在用户态下运行，为了方便起见，可以将信号量机制的实现放在OS中来提供，然后使用系统调用的方法统一提供出若干个管理信号量的系统调用，分别如下所示：
- 申请创建一个信号量的系统调用，可以指定初始值，返回一个信号量描述符(类似文件描述符)；
- 将指定信号量执行P操作；
- 将指定信号量执行V操作；
- 将指定信号量释放掉；

给内核级线程提供信号量机制和给用户态进程/线程提供信号量机制的异同点在于：
> 相同点：
提供信号量机制的代码实现逻辑是相同的；  
> 不同点：
由于实现原子操作的中断禁用、Test and Set指令等均需要在内核态下运行，因此提供给用户态进程的信号量机制是通过系统调用来实现的，而内核级线程只需要直接调用相应的函数就可以了；

## 练习2: 完成内核级条件变量和基于内核级条件变量的哲学家就餐问题
首先掌握管程机制，然后基于信号量实现完成条件变量实现，然后用管程机制实现哲学家就餐问题的解决方案（基于条件变量）。

In [OS CONCEPT] 7.7 section, the accurate define and approximate implementation of MONITOR was introduced.
INTRODUCTION:
通常，管程是一种语言结构，编译器通常会强制执行互斥。 将其与信号量进行比较，信号量通常是OS构造。
DEFNIE & CHARACTERISTIC:
管程是组合在一起的过程、变量和数据结构的集合。
进程可以调用监视程序但无法访问内部数据结构。
管程中一次只能有一个进程处于活动状态。
条件变量允许阻塞和解除阻塞。
  cv.wait() 阻塞一个进程
     该过程等待条件变量cv。
  cv.signal() (也视为 cv.notify) 解除一个等待条件变量cv的进程的阻塞状态。
     发生这种情况时，我们仍然需要在管程中只有一个进程处于活动状态。 这可以通过以下几种方式完成：
         在某些系统上，旧进程（执行信号的进程）离开管程，新进程进入
         在某些系统上，信号必须是管程内执行的最后一个语句。
         在某些系统上，旧进程将阻塞，直到管程再次可用。
         在某些系统上，新进程（未被信号阻止的进程）将保持阻塞状态，直到管程再次可用。
如果在没有人等待的情况下发出条件变量信号，则信号丢失。 将此与信号量进行比较，其中信号将允许将来执行等待的进程无阻塞。
不应该将条件变量视为传统意义上的变量。
它没有价值。
将其视为OOP意义上的对象。
它有两种方法，wait和signal来操纵调用过程。
定义如下，mutex保证对操作的互斥访问，这些访问主要是对共享变量的访问，所以需要互斥；cv是条件变量。
```
monitor mt {
    ----------------variable------------------
    semaphore mutex;
    semaphore next;
    int next_count;
    condvar {int count, sempahore sem}  cv[N];
    other variables in mt;
}
```
实现如下：
```
typedef struct condvar{
    semaphore_t sem;        // the sem semaphore is used to down the waiting proc, 
                            // and the signaling proc should up the waiting proc
    int count;              // the number of waiters on condvar
    monitor_t * owner;      // the owner(monitor) of this condvar
} condvar_t;

typedef struct monitor{
    semaphore_t mutex;      // the mutex lock for going into the routines in monitor, should be initialized to 1    semaphore_t next;       // the next semaphore is used to down the signaling proc itself, 
                            // and the other OR wakeuped waiting proc should wake up the sleeped signaling proc.
    int next_count;         // the number of of sleeped signaling proc
    condvar_t *cv;          // the condvars in monitor
} monitor_t;
```
这是一个管程里的操作，首先在操作开始和结束有wait和signal，保证对中间的访问是互斥的，条件不满足则执行wait执行等待。特殊信号量next和后边的if-else是有对应关系的。
```
    --------routines in monitor---------------
    routineA_in_mt () {
       wait(mt.mutex);
       ...
       real body of routineA
       ...
       if(next_count>0)
           signal(mt.next);
       else
           signal(mt.mutex);
    }
```

条件变量是管程的重要组成部分。
cond_wait: 一个条件得不到满足，则睡眠，如果这个条件得到满足，则另一个进程调用signal唤醒这个进程。该函数的功能为将当前进程等待在指定信号量上。等待队列的计数加1，然后释放管程的锁或者唤醒一个next上的进程来释放锁（否则会造成管程被锁死无法继续访问，同时这个操作不能和前面的等待队列计数加1的操作互换顺序，要不不能保证共享变量访问的互斥性），然后把自己等在条件变量的等待队列上，直到有signal信号将其唤醒，正常退出函数；
```
    --------condvar wait/signal---------------
    cond_wait (cv) {
        cv.count ++;
        if(mt.next_count>0)
           signal(mt.next)
        else
           signal(mt.mutex);
        wait(cv.sem);//由于条件不满足，则wait，这里时cv的sem
        cv.count --;
     }
```
实现：
```
// Suspend calling thread on a condition variable waiting for condition Atomically unlocks
// mutex and suspends calling thread on conditional variable after waking up locks mutex. Notice: mp is mutex s
emaphore for monitor's procedures
void
cond_wait (condvar_t *cvp) {
    //LAB7 EXERCISE1: YOUR CODE
    cprintf("cond_wait begin:  cvp %x, cvp->count %d, cvp->owner->next_count %d\n", cvp, cvp->count, cvp->owner
->next_count);
    cvp->count ++;
    if (cvp->owner->next_count > 0) {
        up(&cvp->owner->next);
    } else {
        up(&cvp->owner->mutex);
    }
    down(&cvp->sem);
    cvp->count --;
    cprintf("cond_wait end:  cvp %x, cvp->count %d, cvp->owner->next_count %d\n", cvp, cvp->count, cvp->owner->
next_count);
}
```
cond_signal: 将指定条件变量上等待队列中的一个线程进行唤醒，并且将控制权转交给这个进程。判断当前的条件变量的等待队列是否大于0，即队列上是否有正在等待的进程，如果没有则不需要进行任何操作；如果有正在等待的进程，则将其中的一个唤醒，这里的等待队列是使用了一个信号量来进行实现的，由于信号量中已经包括了对等待队列的操作，因此要进行唤醒只需要对信号量执行up操作即可；接下来当前进程为了将控制权转交给被唤醒的进程，将自己等待到了这个条件变量所述的管程的next信号量上，这样的话就可以切换到被唤醒的进程。

有线程处于等待时，它的cv.count大于0，会有进一步的操作，唤醒其他进程，自身处于睡眠状态。上边的wait如果A进程中monitor.next_count大于0，那么可以唤醒monitor.next，正好与这里的wait对应。
```
如果cv.count大于0，有线程正在等待，把线程A从等待队列中移除，并唤醒线程A。在A的real_body之后的那个signal是唤醒B的实际函数。这里的next_count是发出条件变量signal的线程的个数。当B发出了条件变量signal操作，且把自身置成睡眠状态，使得被唤醒的A有机会在它自己退出的时候唤醒B。这是因为A和B都是在管程中执行的函数，都会涉及到对共享变量的访问，但是只允许一个进程对共享变量访问，保证互斥！
     cond_signal(cv) {
         if(cv.count>0) {
            mt.next_count ++;
            signal(cv.sem);
            wait(mt.next);
            mt.next_count--;
         }
      }
```

实现：
```
// Unlock one of threads waiting on the condition variable.
void
cond_signal (condvar_t *cvp) {
   //LAB7 EXERCISE1: YOUR CODE
   cprintf("cond_signal begin: cvp %x, cvp->count %d, cvp->owner->next_count %d\n", cvp, cvp->count, cvp->owner
->next_count);
   if(cvp->count>0) {
       cvp->owner->next_count ++;
       up(&cvp->sem);
       down(&cvp->owner->next);
       cvp->owner->next_count --;
   }
   cprintf("cond_signal end: cvp %x, cvp->count %d, cvp->owner->next_count %d\n", cvp, cvp->count, cvp->owner->
next_count);
}
```

哲学家就餐问题：
phi_take_forks_condvar表示指定的哲学家尝试获得自己所需要进餐的两把叉子，如果不能获得则阻塞。首先给管程上锁，将哲学家的状态修改为HUNGER，判断当前哲学家是否可以获得足够的资源进行就餐，即判断与之相邻的哲学家是否正在进餐；如果能够进餐，将自己的状态修改成EATING，然后释放锁，离开管程即可；如果不能进餐，等待在自己对应的条件变量上，等待相邻的哲学家释放资源的时候将自己唤醒；
最终具体的代码实现如下：
```

void phi_take_forks_condvar(int i) {
//--------into routine in monitor--------------
     // LAB7 EXERCISE1: YOUR CODE
     // I am hungry
     // try to get fork
//--------leave routine in monitor--------------
      down(&(mtp->mutex));
      state_condvar[i]=HUNGRY;
      if(state_condvar[(i+4)%5]!=EATING && state_condvar[(i+1)%5]!=EATING){
          state_condvar[i]=EATING;
      }
      else
      {
          cprintf("phi_take_forks_condvar: %d didn’t get fork and will wait\n", i);
          cond_wait(mtp->cv + i);
      }

      if(mtp->next_count>0)
         up(&(mtp->next));
      else
         up(&(mtp->mutex));
}
```
phi_put_forks_condvar函数则是释放当前哲学家占用的叉子，并且唤醒相邻的因为得不到资源而进入等待的哲学家。首先获取管程的锁，将自己的状态修改成THINKING，检查相邻的哲学家是否在自己释放了叉子的占用之后满足了进餐的条件，如果满足，将其从等待中唤醒（使用cond_signal）；释放锁，离开管程；
```
void phi_put_forks_condvar(int i) {
//--------into routine in monitor--------------
     // LAB7 EXERCISE1: YOUR CODE
     // I ate over
     // test left and right neighbors
//--------leave routine in monitor--------------
    down(&(mtp->mutex));
    state_condvar[i] = THINKING;
    cprintf("phi_put_forks_condvar: %d finished eating\n", i);
    phi_test_condvar((i + N - 1) % N);
    phi_test_condvar((i + 1) % N);
    if(mtp->next_count>0)
       up(&(mtp->next));
    else
       up(&(mtp->mutex));
}
```
phi_test_sema检查了第i个哲学家左右两边的人是不是处于EATING状态，如果都不是的话，而且第i个人又是HUNGRY的，则唤醒第i个。
```
#define LEFT (i-1+N)%N /* i的左邻号码 */
#define RIGHT (i+1)%N /* i的右邻号码 */
void phi_test_sema(i) /* i：哲学家号码从0到N-1 */
{
    if(state_sema[i]==HUNGRY&&state_sema[LEFT]!=EATING
            &&state_sema[RIGHT]!=EATING)
    {
        state_sema[i]=EATING;
        up(&s[i]);
    }
}

```
请在实验报告中给出给用户态进程/线程提供条件变量机制的设计方案，并比较说明给内核级 提供条件变量机制的异同。

本实验中管程的实现中互斥访问的保证是完全基于信号量的，如果根据上文中的说明使用系统调用实现用户态的信号量的实现机制，那么就可以按照相同的逻辑在用户态实现管程机制和条件变量机制；
