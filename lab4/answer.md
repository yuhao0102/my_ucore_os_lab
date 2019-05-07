# Lab4 内核线程管理 实验报告
## 实验目的
了解内核线程创建/执行的管理过程  
了解内核线程的切换和基本调度过程

## 实验内容
当一个程序加载到内存中运行时，首先通过ucore OS的内存管理子系统分配合适的空间，然后就需要考虑如何分时使用CPU来“并发”执行多个程序，让每个运行的程序（这里用线程或进程表示）“感到”它们各自拥有“自己”的CPU。

内核线程是一种特殊的进程，内核线程与用户进程的区别有两个：
- 内核线程只运行在内核态
- 用户进程会在在用户态和内核态交替运行
- 所有内核线程共用ucore内核内存空间，不需为每个内核线程维护单独的内存空间
- 用户进程需要维护各自的用户内存空间

## 预备知识
### 内核线程管理
本实验实现了让ucore实现分时共享CPU，实现多条控制流能够并发执行。**内核线程**是一种特殊的进程，内核线程与用户进程的区别有两个：
- **内核线程只运行在内核态而用户进程会在在用户态和内核态交替运行**；
- **所有内核线程直接使用共同的ucore内核内存空间**，不需为每个内核线程维护单独的内存空间而用户进程需要维护各自的用户内存空间。

设计管理线程的数据结构，即进程控制块(PCB)。创建内核线程对应的进程控制块，把这些进程控制块通过链表连在一起，便于随时进行插入，删除和查找操作。通过调度器（scheduler）来让不同的内核线程在不同的时间段占用CPU执行，实现对CPU的分时共享。

kern/init/init.c中的kern_init函数中，当完成虚拟内存的初始化工作vmm_init()后，就调用了proc_init函数。
```
void
proc_init(void) {
    int i;

    list_init(&proc_list);
    // initialize the process double linked list
    for (i = 0; i < HASH_LIST_SIZE; i ++) {
        list_init(hash_list + i);
    }

    if ((idleproc = alloc_proc()) == NULL) {
        panic("cannot alloc idleproc.\n");
    }

    idleproc->pid = 0;
    idleproc->state = PROC_RUNNABLE;
    idleproc->kstack = (uintptr_t)bootstack;
    idleproc->need_resched = 1;
    // 完成了idleproc内核线程创建   
    set_proc_name(idleproc, "idle");
    nr_process ++;

    current = idleproc;

    int pid = kernel_thread(init_main, "Hello world!!", 0);
    if (pid <= 0) {
        panic("create init_main failed.\n");
    }

    initproc = find_proc(pid);
    // initproc内核线程的创建
    set_proc_name(initproc, "init");

    assert(idleproc != NULL && idleproc->pid == 0);
    assert(initproc != NULL && initproc->pid == 1);
}
```
idleproc内核线程的工作就是不停地查询，看是否有其他内核线程可以执行了，如果有，马上让调度器选择那个内核线程执行（请参考cpu_idle函数的实现）。所以**idleproc内核线程是在ucore操作系统没有其他内核线程可执行的情况下才会被调用**。

接着就是调用kernel_thread函数来创建initproc内核线程。initproc内核线程的工作就是显示“Hello World”，表明自己存在且能正常工作了。
调度器会在特定的调度点上执行调度，完成进程切换。

在lab4中，这个调度点就一处，即在cpu_idle函数中，此函数如果发现当前进程（也就是idleproc）的need_resched置为1（在初始化idleproc的进程控制块时就置为1了），则调用schedule函数，完成进程调度和进程切换。进程调度的过程其实比较简单，就是在进程控制块链表中查找到一个“合适”的内核线程，所谓“合适”就是指内核线程处于“PROC_RUNNABLE”状态。

在接下来的switch_to函数(在后续有详细分析，有一定难度，需深入了解一下)完成具体的进程切换过程。一旦切换成功，那么initproc内核线程就可以通过显示字符串来表明本次实验成功。

进程管理信息用struct proc_struct表示，在kern/process/proc.h中定义如下：
```
struct proc_struct {
    enum proc_state state;        // Process state
    int pid;                      // Process ID
    int runs;                     // the running times of Proces
    uintptr_t kstack;             // Process kernel stack
    volatile bool need_resched;   // need to be rescheduled to release CPU?
    struct proc_struct *parent;   // the parent process
    struct mm_struct *mm;         // Process's memory management field
    struct context context;       // Switch here to run process
    struct trapframe *tf;         // Trap frame for current interrupt
    uintptr_t cr3;                // the base addr of Page Directroy Table(PDT)
    uint32_t flags;               // Process flag
    char name[PROC_NAME_LEN + 1]; // Process name
    list_entry_t list_link;       // Process link list
    list_entry_t hash_link;       // Process hash list
};
```

- mm：内存管理的信息。在lab3中有涉及，主要包括内存映射列表、页表指针等。**在实际OS中，内核线程常驻内存，不需要考虑swap page问题**，在用户进程中考虑进程用户内存空间的swap_page问题时mm才会发挥作用。所以在lab4中mm对于内核线程就没有用了，这样内核线程的proc_struct的成员变量*mm=0是合理的。mm里有个很重要的项pgdir，记录的是该进程使用的一级页表的物理地址。由于*mm=NULL，所以在proc_struct数据结构中需要有一个代替pgdir项来记录页表起始地址，这就是proc_struct数据结构中的**cr3**成员变量。
- state：进程所处的状态。
```
enum proc_state {
    PROC_UNINIT = 0,  // uninitialized
    PROC_SLEEPING,    // sleeping
    PROC_RUNNABLE,    // runnable(maybe running)
    PROC_ZOMBIE,      // almost dead, and wait parent proc to reclaim his resource
};
```
- parent：用户进程的父进程（创建它的进程）。在所有进程中，只有一个进程没有父进程，就是内核创建的第一个内核线程idleproc。内核根据这个父子关系建立一个树形结构，用于维护一些特殊的操作，例如确定某个进程是否可以对另外一个进程进行某种操作等等。
- context：进程的上下文，用于进程切换（参见switch.S）。在uCore中，所有的进程在内核中也是相对独立的（例如独立的内核堆栈以及上下文等等）。使用context保存寄存器的目的就在于在内核态中能够进行上下文之间的切换。实际利用context进行上下文切换的函数是在kern/process/switch.S中定义switch_to。
```
// 在上下文切换时保存寄存器信息，其中有些寄存器貌似不被保存，为了省事
// The 这个结构体的布局要跟switch.S中的switch_to操作对应。
struct context {
    uint32_t eip;
    uint32_t esp;
    uint32_t ebx;
    uint32_t ecx;
    uint32_t edx;
    uint32_t esi;
    uint32_t edi;
    uint32_t ebp;
};
```
- tf：中断帧的指针，总是指向内核栈的某个位置：当进程从用户空间跳到内核空间时，**中断帧记录了进程在被中断前的状态**。当内核需要跳回用户空间时，需要调整中断帧以**恢复让进程继续执行的各寄存器值**。除此之外，uCore内核允许嵌套中断。因此为了保证嵌套中断发生时tf总是能够指向当前的trapframe，uCore在内核栈上维护了tf的链。
- cr3: cr3 保存页表的物理地址，目的就是进程切换的时候方便直接使用lcr3实现页表切换，避免每次都根据 mm 来计算 cr3。mm数据结构是用来实现用户空间的虚存管理的，但是内核线程没有用户空间，它执行的只是内核中的一小段代码（通常是一小段函数），所以它没有mm结构，也就是NULL。当某个进程是一个普通用户态进程的时候，PCB中的cr3就是mm中页表（pgdir）的物理地址；而当它是内核线程的时候，cr3等于boot_cr3。而boot_cr3指向了uCore启动时建立好的内核虚拟空间的页目录表首地址。
- kstack: 每个线程都有一个**内核栈**，并且位于内核地址空间的不同位置。对于内核线程，该栈就是运行时的程序使用的栈；而**对于普通进程，该栈是发生特权级改变的时候使保存被打断的硬件信息用的栈**。uCore在创建进程时分配了 2 个连续的物理页（参见memlayout.h中KSTACKSIZE的定义）作为内核栈的空间。这个栈很小，所以内核中的代码应该尽可能的紧凑，并且避免在栈上分配大的数据结构，以免栈溢出，导致系统崩溃。kstack记录了分配给该进程/线程的内核栈的位置。主要作用有以下几点。

> 首先，当内核准备从一个进程切换到另一个的时候，需要根据kstack 的值正确的设置好tss，以便在进程切换以后再发生中断时能够使用正确的栈。

> 其次，内核栈位于内核地址空间，并且是不共享的（每个线程都拥有自己的内核栈），因此不受到 mm 的管理，当进程退出的时候，内核能够根据 kstack 的值快速定位栈的位置并进行回收。uCore 的这种内核栈的设计借鉴的是 linux 的方法（但由于内存管理实现的差异，它实现的远不如 linux 的灵活），它使得每个线程的内核栈在不同的位置，这样从某种程度上方便调试，但同时也使得内核对栈溢出变得十分不敏感，因为一旦发生溢出，它极可能污染内核中其它的数据使得内核崩溃。如果能够通过页表，将所有进程的内核栈映射到固定的地址上去，能够避免这种问题，但又会使得进程切换过程中对栈的修改变得相当繁琐。


为了管理系统中所有的进程控制块，uCore维护了如下全局变量（位于kern/process/proc.c）：
- static struct proc \*current：当前占用CPU且处于“运行”状态进程控制块指针。通常这个变量是只读的，只有在进程切换的时候才进行修改，并且整个切换和修改过程需要保证操作的原子性，目前至少需要屏蔽中断。可以参考 switch_to 的实现。
- static struct proc \*initproc：本实验中，指向一个内核线程。本实验以后，此指针将指向第一个用户态进程。
- static list_entry_t hash_list[HASH_LIST_SIZE]：所有进程控制块的哈希表，proc_struct中的成员变量hash_link将基于pid链接入这个哈希表中。
- list_entry_t proc_list：所有进程控制块的双向线性列表，proc_struct中的成员变量list_link将链接入这个链表中。

### 创建并执行内核线程
ucore实现了一个简单的进程/线程机制，进程包含独立的地址空间，至少一个线程、内核数据、进程状态、文件等。ucore需要高效地管理所有细节。在ucore，一个线程看成一个特殊的进程（process）。

进程状态|意义|原因
|PROC_UNINIT|uninitialized|alloc_proc|
|----|----|----|
|PROC_SLEEPING|sleeping|try_free_pages, do_wait, do_sleep|
|PROC_RUNNABLE|runnable(maybe running)|proc_init, wakeup_proc,|
|PROC_ZOMBIE|almost dead|do_exit|

进程之间的关系：
parent:           proc->parent  (proc is children)
children:         proc->cptr    (proc is parent)
older sibling:    proc->optr    (proc is younger sibling)
younger sibling:  proc->yptr    (proc is older sibling)

建立进程控制块（proc.c中的alloc_proc函数）。首先，考虑最简单的内核线程，它通常只是内核中的一小段代码或者函数，没有自己的“专属”空间。这是由于在uCore OS启动后，已经对整个内核内存空间进行了管理，通过设置页表建立了内核虚拟空间（即boot_cr3指向的二级页表描述的空间）。所以uCore OS内核中的所有线程都不需要再建立各自的页表，只需共享这个内核虚拟空间就可以访问整个物理内存了。从这个角度看，内核线程被uCore OS内核这个大“内核进程”所管理。

### 创建第 0 个内核线程 idleproc
在init.c中的kern_init函数调用了proc.c中的proc_init函数。proc_init函数启动了创建内核线程的步骤。

首先当前的执行上下文（从kern_init启动至今）就可以看成是uCore内核（也可看做是内核进程）中的一个内核线程的上下文。为此，uCore通过给当前执行的上下文分配一个进程控制块以及对它进行相应初始化，将其打造成第0个内核线程——idleproc。具体步骤如下：
- 首先调用alloc_proc函数来通过kmalloc函数获得proc_struct结构的一块内存块，作为第0个进程控制块，并把proc进行初步初始化（即把proc_struct中的各个成员变量清零）。但有些成员变量设置了特殊的值，比如：
```
proc->state = PROC_UNINIT;  设置进程为“初始”态
proc->pid = -1;             设置进程pid的未初始化值
proc->cr3 = boot_cr3;       由于该内核线程在内核中运行，故采用为uCore内核已经建立的页表，
							即设置为在uCore内核页表的起始地址boot_cr3，使用内核页目录表的基址
```
内核线程共用一个映射内核空间的页表，这表示内核空间对所有内核线程都是“可见”的，所以更精确地说，这些内核线程都应该是从属于同一个唯一的“大内核进程”—uCore内核。

- proc_init函数对idleproc内核线程进行进一步初始化：
```
idleproc->pid = 0;
idleproc->state = PROC_RUNNABLE;
idleproc->kstack = (uintptr_t)bootstack;
idleproc->need_resched = 1;
set_proc_name(idleproc, "idle");
```
第一条将pid赋值为0，表明idleproc是第0个内核线程。

第二条语句改变了idleproc的状态，使其变为“准备工作”，现在只要uCore调度便可执行。

第三条语句设置了idleproc所使用的内核栈的起始地址。**需要注意以后的其他线程的内核栈都需要通过分配获得，因为uCore启动时设置的内核栈就直接分配给idleproc使用了所以这里不用分配**。

第四条把idleproc->need_resched设置为“1”，在cpu_idle函数中指明如果进程的need_resched为1那么就可以调度执行其他的了，如果当前idleproc在执行，则只要此标志为1，马上就调用schedule函数要求调度器切换其他进程执行。

### 创建第 1 个内核线程 initproc
第0个内核线程主要工作是完成内核中各个子系统的初始化。uCore接下来还需创建其他进程来完成各种工作，通过调用kernel_thread函数创建了一个内核线程init_main。
```
// init_main - the second kernel thread used to create user_main kernel threads
static int
init_main(void *arg) {
    cprintf("this initproc, pid = %d, name = \"%s\"\n", current->pid, get_proc_name(current));
    cprintf("To U: \"%s\".\n", (const char *)arg);
    cprintf("To U: \"en.., Bye, Bye. :)\"\n");
    return 0;
}
```
下面我们来分析一下创建内核线程的函数kernel_thread。kernel_thread函数采用了局部变量tf来放置保存内核线程的临时中断帧，并把中断帧的指针传递给do_fork函数，而do_fork函数会调用copy_thread函数来在新创建的进程内核栈上专门给进程的中断帧分配一块空间。给中断帧分配完空间后，就需要构造新进程的中断帧，具体过程是：
```
kernel_thread(int (*fn)(void *), void *arg, uint32_t clone_flags)
{
    struct trapframe tf;
    memset(&tf, 0, sizeof(struct trapframe));
    // 给tf进行清零初始化
    tf.tf_cs = KERNEL_CS;
    tf.tf_ds = tf_struct.tf_es = tf_struct.tf_ss = KERNEL_DS;
    // 设置中断帧的代码段（tf.tf_cs）和数据段(tf.tf_ds/tf_es/tf_ss)为内核空间的段（KERNEL_CS/KERNEL_DS）
    tf.tf_regs.reg_ebx = (uint32_t)fn;
    // fn是函数主体
    tf.tf_regs.reg_edx = (uint32_t)arg;
    // arg是fn函数的参数
    tf.tf_eip = (uint32_t)kernel_thread_entry;
    // tf.tf_eip的指出了initproc内核线程从kernel_thread_entry开始执行
    return do_fork(clone_flags | CLONE_VM, 0, &tf);
}
```
kernel_thread_entry是entry.S中实现的汇编函数，它做的事情很简单：
```
kernel_thread_entry: # void kernel_thread(void)
pushl %edx # push arg
call *%ebx # call fn
pushl %eax # save the return value of fn(arg)
call do_exit # call do_exit to terminate current thread
```
从上可以看出，kernel_thread_entry函数主要为内核线程的主体fn函数做了一个准备开始和结束运行的“壳”：
- 把函数fn的参数arg（保存在edx寄存器中）压栈；
- 调用fn函数
- 把函数返回值eax寄存器内容压栈
- 调用do_exit函数退出线程执行。

do_fork是创建线程的主要函数。kernel_thread函数通过调用do_fork函数最终完成了内核线程的创建工作。do_fork函数主要做了以下6件事情：
- 分配并初始化进程控制块（alloc_proc函数）；
- 分配并初始化内核栈（setup_stack函数）；
- 根据clone_flag标志复制或共享进程内存管理结构（copy_mm函数）；
- 设置进程在内核（将来也包括用户态）正常运行和调度所需的中断帧和执行上下文（copy_thread函数）；
- 把设置好的进程控制块放入hash_list和proc_list两个全局进程链表中；
- 进程已经准备好执行了，把进程状态设置为“就绪”态；设置返回码为子进程的id号。

copy_thread函数代码如下：
```
static void
copy_thread(struct proc_struct *proc, uintptr_t esp, struct trapframe *tf) {
    proc->tf = (struct trapframe *)(proc->kstack + KSTACKSIZE) - 1;
    // 在内核堆栈的顶部设置中断帧大小的一块栈空间
    *(proc->tf) = *tf; 
    // 拷贝在kernel_thread函数建立的临时中断帧的初始值
    proc->tf->tf_regs.reg_eax = 0;
    // 设置子进程/线程执行完do_fork后的返回值
    proc->tf->tf_esp = esp; 
    // 设置中断帧中的栈指针esp
    proc->tf->tf_eflags |= FL_IF; 
    // 使能中断
    // 以上两句设置中断帧中的栈指针esp和标志寄存器eflags，特别是eflags设置了FL_IF标志，
    // 这表示此内核线程在执行过程中，能响应中断，打断当前的执行。
    proc->context.eip = (uintptr_t)forkret;
    proc->context.esp = (uintptr_t)(proc->tf);
}
```
对于initproc而言，它的中断帧如下所示：
```
//所在地址位置
initproc->tf= (proc->kstack+KSTACKSIZE) – sizeof (struct trapframe);
//具体内容
initproc->tf.tf_cs = KERNEL_CS;
initproc->tf.tf_ds = initproc->tf.tf_es = initproc->tf.tf_ss = KERNEL_DS;
initproc->tf.tf_regs.reg_ebx = (uint32_t)init_main;
initproc->tf.tf_regs.reg_edx = (uint32_t) ADDRESS of "Helloworld!!";
initproc->tf.tf_eip = (uint32_t)kernel_thread_entry;
initproc->tf.tf_regs.reg_eax = 0;
initproc->tf.tf_esp = esp;
initproc->tf.tf_eflags |= FL_IF;
```
设置好中断帧后，最后就是设置initproc的进程上下文。uCore调度器选择了initproc执行，需要根据initproc->context中保存的执行现场来恢复initproc的执行。这里设置了initproc的执行现场中主要的两个信息：
- 上次停止执行时的下一条指令地址context.eip
- 上次停止执行时的堆栈地址context.esp。

可以看出，由于initproc的中断帧占用了实际给initproc分配的栈空间的顶部，所以initproc就只能把栈顶指针context.esp设置在initproc的中断帧的起始位置。根据context.eip的赋值，可以知道initproc实际开始执行的地方在forkret函数（主要完成do_fork函数返回的处理工作）处。至此，initproc内核线程已经做好准备执行了。

### 调度并执行内核线程 initproc
在uCore执行完proc_init函数后，就创建好了两个内核线程：`idleproc`和`initproc`，这时uCore当前的执行现场就是idleproc，等到执行到init函数的最后一个函数cpu_idle之前，uCore的所有初始化工作就结束了，idleproc将通过执行cpu_idle函数让出CPU，给其它内核线程执行，具体过程如下：
```
void
cpu_idle(void) {
    while (1) {
        if (current->need_resched) {
            schedule();
        }
    }
}
```       
首先，判断当前内核线程idleproc的need_resched是否不为0，idleproc中的need_resched本就置为1，所以会马上调用schedule函数找其他处于“就绪”态的进程执行。uCore的调度器为FIFO调度器，其核心就是schedule函数。它的执行逻辑很简单：
- 设置当前内核线程current->need_resched为0；
- 在proc_list队列中查找下一个处于“就绪”态的线程或进程；
- 找到这样的进程后，就调用proc_run函数，保存当前进程current的上下文，恢复新进程的执行现场，完成进程切换。

uCore通过proc_run和进一步的switch_to函数完成两个执行现场的切换，具体流程如下：
- 让current指向next内核线程initproc；
- 设置任务状态段ts中特权态0下的栈顶指针esp0为next内核线程initproc的内核栈的栈顶，即next->kstack + KSTACKSIZE；
- 设置CR3寄存器的值为next内核线程initproc的页目录表起始地址next->cr3，这实际上是完成进程间的页表切换；
- 由switch_to函数完成具体的两个线程的执行现场切换，即切换各个寄存器，当switch_to函数执行完“ret”指令后，就切换到initproc执行了。

注意，在第二步设置任务状态段ts中特权态0下的栈顶指针esp0的目的是**建立好内核线程**或**将来用户线程在执行特权态切换（从特权态0<-->特权态3，或从特权态3<-->特权态0）时能够正确定位处于特权态0时进程的内核栈的栈顶**，而这个栈顶其实放了一个trapframe结构的内存空间。如果是在特权态3发生了中断/异常/系统调用，则CPU会从特权态3-->特权态0，且CPU从此栈顶（当前被打断进程的内核栈顶）开始压栈来保存被中断/异常/系统调用打断的用户态执行现场；如果是在特权态0发生了中断/异常/系统调用，则CPU会从从当前内核栈指针esp所指的位置开始压栈保存被中断/异常/系统调用打断的内核态执行现场。反之，当执行完对中断/异常/系统调用打断的处理后，最后会执行一个“iret”指令。在执行此指令之前，CPU的当前栈指针esp一定指向上次产生中断/异常/系统调用时CPU保存的被打断的指令地址CS和EIP，“iret”指令会根据ESP所指的保存的址CS和EIP恢复到上次被打断的地方继续执行。

第四步proc_run函数调用switch_to函数，参数是前一个进程和后一个进程的执行现场。

switch.S中的switch_to函数的执行流程：
```
.globl switch_to
switch_to: # switch_to(from, to)
### save from's registers ###
movl 4(%esp), %eax # eax points to from
popl 0(%eax)
# esp--> return address, so save return addr in FROM’s context
保存前一个进程的执行现场，前两条汇编指令保存了进程在返回switch_to函数后的指令地址到context.eip中

movl %esp, 4(%eax)
……
movl %ebp, 28(%eax)
 7条汇编指令完成了保存前一个进程的其他7个寄存器到context中的相应成员变量中

### restore to's registers ###
 恢复下一个进程的执行现场，这其实就是上述保存过程的逆执行过程
movl 4(%esp), %eax # not 8(%esp): popped return address already
# eax now points to to

movl 28(%eax), %ebp
……
movl 4(%eax), %esp
 从context的高地址的成员变量ebp开始，逐一把相关成员变量的值赋值给对应的寄存器

pushl 0(%eax) 
# push TO’s context’s eip, so return addr = TO’s eip
 把context中保存的下一个进程要执行的指令地址context.eip放到了堆栈顶

ret 
 after ret, eip= TO’s eip
 把栈顶的内容赋值给EIP寄存器，这样就切换到下一个进程执行了，即当前进程已经是下一个进程了
```
uCore会执行进程切换，让initproc执行。在对initproc进行初始化时，设置了initproc->context.eip = (uintptr_t)forkret，这样，当执行switch_to函数并返回后，initproc将执行其实际上的执行入口地址forkret。而forkret会调用位于kern/trap/trapentry.S中的forkrets函数执行，具体代码如下：
```
.globl __trapret
 __trapret:
 # restore registers from stack
 popal
 # restore %ds and %es
 popl %es
 popl %ds
 # get rid of the trap number and error code
 addl $0x8, %esp
 iret

 .globl forkrets
 forkrets:
 # set stack to this new process trapframe
 movl 4(%esp), %esp 
 把esp指向当前进程的中断帧，esp指向了current->tf.tf_eip
 
 jmp __trapret
```
如果此时执行的是initproc，则current->tf.tf_eip=kernel_thread_entry，initproc->tf.tf_cs = KERNEL_CS，所以当执行完iret后，就开始在内核中执行kernel_thread_entry函数了。

而initproc->tf.tf_regs.reg_ebx = init_main，所以在kernl_thread_entry中执行“call %ebx”后，就开始执行initproc的主体了。Initprocde的主体函数很简单就是输出一段字符串，然后就返回到kernel_tread_entry函数，并进一步调用do_exit执行退出操作了。

## 练习1：分配并初始化一个进程控制块
alloc_proc函数（位于kern/process/proc.c中）负责分配并返回一个新的struct proc_struct结 构，用于存储新建立的内核线程的管理信息。比较简单，state、pid和cr3需要考虑，其他的无脑赋0和memset一波带走就行
```
static struct proc_struct *
alloc_proc(void) {
    struct proc_struct *proc = kmalloc(sizeof(struct proc_struct));
    if (proc != NULL) {
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
    }
    return proc;
}
```
```
请说明proc_struct中struct context context和struct trapframe *tf成员变量含义和在本实验中的作用是啥？
（提示通过看代码和编程调试可以判断出来）：
结构体中存储了除eax之外的所有通用寄存器以及eip的数值，保存了线程运行的上下文信息；

struct context {
    uint32_t eip;
    uint32_t esp;
    uint32_t ebx;
    uint32_t ecx;
    uint32_t edx;
    uint32_t esi;
    uint32_t edi;
    uint32_t ebp;
};
context用于内核线程之间切换时，保存原先线程运行的上下文

struct trapframe *tf的作用：

在copy_thread函数中对tf进行了设置。在这个函数中，把context变量的esp设置成tf变量的地址，把eip设置成forkret函数指针。
forkret函数调用了__trapret进行中断返回，tf变量用于构造出新线程时，正确地将控制权转交给新的线程。
```

## 练习2：为新创建的内核线程分配资源
创建一个内核线程需要分配和设置好很多资源。kernel_thread函数通过调用do_fork函数完成具体内核线程的创建工作。do_fork函数会调用alloc_proc函数来分配并初始化一个进程控制块，但alloc_proc只是找到了一小块内存用以记录进程的必要信息，并没有实际分配这些资源。

ucore一般通过do_fork实际创建新的内核线程。do_fork的作用是：
> 创建当前内核线程的一个副本，它们的执行上下文、代码、数据都一样，但是存储位置不同。在这个过程中，需要给新内核线程分配资源，并且复制原进程的状态。为内核线程创建新的线程控制块，并且对控制块中的每个成员变量进行正确的设置，使得之后可以正确切换到对应的线程中执行。练习2完成了在kern/process/proc.c中的do_fork函数中的处理过程。它的大致执行步骤包括：

- 调用alloc_proc，首先获得一块用户信息块。
- 为进程分配一个内核栈。
- 复制原进程的内存管理信息到新进程（但内核线程不必做此事）
- 复制原进程上下文到新进程
- 将新进程添加到进程列表
- 唤醒新进程
- 返回新进程号

```
/* do_fork -     parent process for a new child process
 * @clone_flags: used to guide how to clone the child process
 * @stack:       the parent's user stack pointer. if stack==0, It means to fork a kernel thread.
 * @tf:          the trapframe info, which will be copied to child process's proc->tf
 */
int
do_fork(uint32_t clone_flags, uintptr_t stack, struct trapframe *tf) {
    int ret = -E_NO_FREE_PROC;
    struct proc_struct *proc;
    if (nr_process >= MAX_PROCESS) {
        goto fork_out;
    }
    ret = -E_NO_MEM;
    //LAB4:EXERCISE2 YOUR CODE
    /*
     * Some Useful MACROs, Functions and DEFINEs, you can use them in below implementation.
     * MACROs or Functions:
     *   alloc_proc:   create a proc struct and init fields (lab4:exercise1)
     *                 创建进程并初始化
     *   setup_kstack: alloc pages with size KSTACKPAGE as process kernel stack
     *                 创建页，大小为KSTACKPAGE，并作为进程的内核栈
     *   copy_mm:      process "proc" duplicate OR share process "current"'s mm according clone_flags
     *                 if clone_flags & CLONE_VM, then "share" ; else "duplicate"
     *                 进程复制memory manager，根据clone_flag不同决定不同操作
     *   copy_thread:  setup the trapframe on the  process's kernel stack top and
     *                 setup the kernel entry point and stack of process
     *                 在进程内核栈顶建立trapframe
     *   hash_proc:    add proc into proc hash_list
     *                 添加进程到hash_list中
     *   get_pid:      alloc a unique pid for process
     *                 为进程分配一个独特的pid
     *   wakeup_proc:  set proc->state = PROC_RUNNABLE
     * VARIABLES:
     *   proc_list:    the process set's list
     *   nr_process:   the number of process set
     */

    // 1. call alloc_proc to allocate a proc_struct
    // 为要创建的新的线程分配线程控制块的空间
    proc = alloc_proc();
    if(proc == NULL)
        goto fork_out;
    // 判断是否分配到内存空间
    // 2. call setup_kstack to allocate a kernel stack for child process
    // 为新的线程设置栈，在本实验中，每个线程的栈的大小初始均为2个Page, 即8KB
    int status = setup_kstack(proc);
    if(status != 0)
        goto fork_out;
    // 3. call copy_mm to dup OR share mm according clone_flag
    // 对虚拟内存空间进行拷贝，由于在本实验中，内核线程之间共享一个虚拟内存空间，因此实际上该函数不需要进行任何操作
    status = copy_mm(clone_flags, proc);
    if(status != 0)
        goto fork_out;
    // 4. call copy_thread to setup tf & context in proc_struct
    // 在新创建的内核线程的栈上面设置伪造好的中端帧，便于后文中利用iret命令将控制权转移给新的线程
    copy_thread(proc, stack, tf);
    // 5. insert proc_struct into hash_list && proc_list
    // 为新的线程创建pid
    proc->pid = get_pid();
    hash_proc(proc);
    // 将线程放入使用hash组织的链表中，便于加速以后对某个指定的线程的查找
    nr_process ++;
    // 将全局线程的数目加1
    list_add(&proc_list, &proc->list_link);
    // 将线程加入到所有线程的链表中，便于进行调度
    // 6. call wakeup_proc to make the new child process RUNNABLE
    // 唤醒该线程，即将该线程的状态设置为可以运行
    wakeup_proc(proc);
    // 7. set ret vaule using child proc's pid
    // 返回新线程的pid
    ret = proc->pid;
fork_out:     
```
请说明ucore是否做到给每个新fork的线程一个唯一的id？请说明你的分析和理由。

可以。ucore中为fork的线程分配pid的函数为get_pid：
```
// get_pid - alloc a unique pid for process
static int get_pid(void) {
    static_assert(MAX_PID > MAX_PROCESS);
    struct proc_struct *proc;
    list_entry_t *list = &proc_list, *le;
    static int next_safe = MAX_PID, last_pid = MAX_PID;
    if (++ last_pid >= MAX_PID) {
        last_pid = 1;
        goto inside;
    }
    if (last_pid >= next_safe) {
    inside:
        next_safe = MAX_PID;
    repeat:
        le = list;
        while ((le = list_next(le)) != list) {
            proc = le2proc(le, list_link);
            if (proc->pid == last_pid) {
                if (++ last_pid >= next_safe) {
                    if (last_pid >= MAX_PID) {
                        last_pid = 1;
                    }
                    next_safe = MAX_PID;
                    goto repeat;
                }
            }
            else if (proc->pid > last_pid && next_safe > proc->pid) {
                next_safe = proc->pid;
            }
        }
    }
    return last_pid;
}            	
```
如果有严格的next_safe > last_pid + 1，那么可以直接取last_pid + 1作为新的pid（需要last_pid没有超出MAX_PID从而变成1），
如果在进入函数的时候，这两个变量之后没有合法的取值，也就是说next_safe > last_pid + 1不成立，那么进入循环，在循环之中首先通过if(proc->pid == last_pid)这一分支确保了不存在任何进程的pid与last_pid重合，然后再通过if (proc->pid > last_pid && next_safe > proc->pid)这一判断语句保证了不存在任何已经存在的pid满足：last_pid< pid < next_safe，这样就确保了最后能够找到这么一个满足条件的区间，获得合法的pid；

## 练习3：阅读代码，理解 proc_run 函数和它调用的函数如何完成进程切换的。
唯一调用到这个函数是在线程调度器的schedule函数中，proc_run将proc加载到CPU
```
// proc_run - make process "proc" running on cpu
// NOTE: before call switch_to, should load  base addr of "proc"'s new PDT
void proc_run(struct proc_struct *proc) {
    // 判断需要运行的线程是否是正在运行的线程
    if (proc != current) { 
        bool intr_flag;
        struct proc_struct *prev = current, *next = proc;
        //如果不是的话，获取到切换前后的两个线程
        local_intr_save(intr_flag);
        // 关闭中断
        {
            current = proc;
            load_esp0(next->kstack + KSTACKSIZE);
            lcr3(next->cr3);
            // 设置了TSS和cr3，相当于是切换了页表和栈
            switch_to(&(prev->context), &(next->context));
            // switch_to恢复要运行的线程的上下文，然后由于恢复的上下文中已经将返回地址（copy_thread函数中完成）修改成了forkret函数的地址(如果这个线程是第一运行的话，否则就是切换到这个线程被切换出来的地址)，也就是会跳转到这个函数，最后进一步跳转到了__trapsret函数，调用iret最终将控制权切换到新的线程；
        }
        local_intr_restore(intr_flag);
        // 使能中断
    }
}
```
forkret函数：
```
// forkret -- the first kernel entry point of a new thread/process
// NOTE: the addr of forkret is setted in copy_thread function
//       after switch_to, the current proc will execute here.
static void
forkret(void) {
    forkrets(current->tf);
}
```


在本实验的执行过程中，创建且运行了几个内核线程？

总共创建了两个内核线程，分别为：

- idleproc: 最初的内核线程，在完成新的内核线程的创建以及各种初始化工作之后，进入死循环，用于调度其他线程；
- initproc: 被创建用于打印"Hello World"的线程；

语句 local_intr_save(intr_flag);....local_intr_restore(intr_flag);说明理由在这里有何作用? 请说明理由。

- 关闭中断，使得在这个语句块内的内容不会被中断打断，是一个原子操作；
- 在proc_run函数中，将current指向了要切换到的线程，但是此时还没有真正将控制权转移过去，如果在这个时候出现中断打断这些操作，就会出现current中保存的并不是正在运行的线程的中断控制块，从而出现错误。
