
# 实验五：用户进程管理
## 实验目的
了解第一个用户进程创建过程  
了解系统调用框架的实现机制  
了解ucore如何实现系统调用sys_fork/sys_exec/sys_exit/sys_wait来进行进程管理  

## 实验内容
实验4的线程运行都在内核态。实验5创建了用户进程，让用户进程在用户态执行，且在需要ucore支持时，可通过系统调用来让ucore提供服务。为此需要构造出第一个用户进程，并通过系统调用sys_fork/sys_exec/sys_exit/sys_wait来支持运行不同的应用程序，完成对用户进程的执行过程的基本管理。

## 预备知识
### 实验执行流程概述
提供各种操作系统功能的**内核线程只能在CPU核心态运行是操作系统自身的要求**，操作系统就要呆在核心态，才能管理整个计算机系统。ucore提供了用户态进程的创建和执行机制，给应用程序执行提供一个用户态运行环境。显然，由于进程的执行空间扩展到了用户态空间，且出现了创建子进程执行应用程序等与lab4有较大不同的地方，所以具体实现的不同主要集中在**进程管理**和**内存管理部分**。

首先，我们从ucore的初始化部分来看，kern_init中调用的物理内存初始化，进程管理初始化等都有一定的变化。在内存管理部分，与lab4最大的区别就是**增加用户态虚拟内存的管理**。
- 首先为了管理用户态的虚拟内存，需要对页表的内容进行扩展，能够**把部分物理内存映射为用户态虚拟内存**。如果某进程执行过程中，CPU在用户态下执行（在CS段寄存器最低两位包含有一个2位的优先级域，如果为0，表示CPU运行在特权态；如果为3，表示CPU运行在用户态。），则可以访问本进程页表描述的用户态虚拟内存，但由于权限不够，不能访问内核态虚拟内存。
- 另一方面，在用户态内存空间和内核态内核空间之间需要拷贝数据，让**CPU处在内核态才能完成对用户空间的读或写**，为此需要设计专门的拷贝函数（copy_from_user和copy_to_user）完成。但反之则会导致违反CPU的权限管理，导致内存访问异常。
- 在进程管理方面，主要涉及到的是进程控制块中与内存管理相关的部分，包括建立进程的页表和维护进程可访问空间（可能还没有建立虚实映射关系）的信息；
- 加载一个ELF格式的程序到进程控制块管理的内存中的方法；
- 在进程复制（fork）过程中，把父进程的内存空间拷贝到子进程内存空间的技术；
- 另外一部分与用户态进程生命周期管理相关，包括让进程放弃CPU而睡眠等待某事件、让父进程等待子进程结束、一个进程杀死另一个进程、给进程发消息、建立进程的血缘关系链表。

在用户进程管理中，首先，构造出第一个进程idle_proc，作为所有后续进程的祖先；然后，在proc_init函数中，对idle_proc进行进一步初始化，通过alloc把当前ucore的执行环境转变成idle内核线程的执行现场；然后调用kernl_thread来创建第二个内核线程init_main，而init_main内核线程有创建了user_main内核线程。到此，内核线程创建完毕。
```
// proc_init - set up the first kernel thread idleproc "idle" by itself and
//           - create the second kernel thread init_main
void
proc_init(void) {
    int i;

    list_init(&proc_list);
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
    set_proc_name(idleproc, "idle");
    nr_process ++;

    current = idleproc;

    int pid = kernel_thread(init_main, NULL, 0);
    if (pid <= 0) {
        panic("create init_main failed.\n");
    }

    initproc = find_proc(pid);
    set_proc_name(initproc, "init");

    assert(idleproc != NULL && idleproc->pid == 0);
    assert(initproc != NULL && initproc->pid == 1);
}        
```

接下来是用户进程的创建过程。第一步实际上是通过user_main函数调用kernel_tread创建子进程，通过kernel_execve调用来把某一具体程序的执行内容放入内存。

具体的放置方式是根据**ld在此文件上的地址分配**为基本原则，把程序的不同部分放到某进程的用户空间中，从而通过此进程来完成程序描述的任务。一旦执行了这一程序对应的进程，就会从内核态切换到用户态继续执行。

以此类推：
> **CPU在用户空间执行的用户进程，其地址空间不会被其他用户的进程影响，但由于系统调用（用户进程直接获得操作系统服务的唯一通道）、外设中断和异常中断的会随时产生，从而间接推动了用户进程实现用户态到到内核态的切换工作。当进程执行结束后，需回收进程占用和没消耗完毕的设备整个过程，且为新的创建进程请求提供服务。**

### 创建用户进程
#### 应用程序的组成和编译
lab5中新增了一个文件夹user，其中是用于本实验的用户程序。如hello.c
```
#include <stdio.h>
#include <ulib.h>

int main(void) {
    cprintf("Hello world!!.\n");
    cprintf("I am process %d.\n", getpid());
    cprintf("hello pass.\n");
    return 0;
}
```
按照手册，注释掉Makefile的第六行，编译，（部分）输出如下：
```
gcc -Iuser/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  
-fno-stack-protector -Ilibs/ -Iuser/include/ -Iuser/libs/ -c user/pgdir.c -o obj/user/pgdir.o

ld -m    elf_i386 -nostdlib -T tools/user.ld -o obj/__user_pgdir.out  
  obj/user/libs/panic.o obj/user/libs/syscall.o obj/user/libs/ulib.o 
  obj/user/libs/initcode.o obj/user/libs/stdio.o obj/user/libs/umain.o  
  obj/libs/string.o obj/libs/printfmt.o obj/libs/hash.o obj/libs/rand.o obj/user/pgdir.o

+ ld bin/kernel
ld -m elf_i386 -nostdlib -T tools/kernel.ld -o bin/kernel  
  obj/kern/init/entry.o obj/kern/init/init.o obj/kern/libs/stdio.o 
  obj/kern/libs/readline.o obj/kern/debug/panic.o obj/kern/debug/kdebug.o
  obj/kern/debug/kmonitor.o obj/kern/driver/ide.o obj/kern/driver/clock.o 
  obj/kern/driver/console.o obj/kern/driver/picirq.o obj/kern/driver/intr.o 
  obj/kern/trap/trap.o obj/kern/trap/vectors.o obj/kern/trap/trapentry.o 
  obj/kern/mm/pmm.o obj/kern/mm/swap_fifo.o obj/kern/mm/vmm.o obj/kern/mm/kmalloc.o 
  obj/kern/mm/swap.o obj/kern/mm/default_pmm.o obj/kern/fs/swapfs.o obj/kern/process/entry.o 
  obj/kern/process/switch.o obj/kern/process/proc.o obj/kern/schedule/sched.o 
  obj/kern/syscall/syscall.o  obj/libs/string.o obj/libs/printfmt.o obj/libs/hash.o obj/libs/rand.o 
  -b binary  obj/__user_badarg.out obj/__user_forktree.out obj/__user_faultread.out obj/__user_divzero.out 
  obj/__user_exit.out obj/__user_hello.out obj/__user_waitkill.out obj/__user_softint.out obj/__user_spin.out
  obj/__user_yield.out obj/__user_badsegment.out obj/__user_testbss.out obj/__user_faultreadkernel.out 
  obj/__user_forktest.out obj/__user_pgdir.out
```
从中可以看出，hello应用程序不仅仅是hello.c，还包含了支持hello应用程序的用户态库：
- user/libs/initcode.S：所有应用程序的起始用户态执行地址“_start”，调整了EBP和ESP后，调用umain函数。
- user/libs/umain.c：实现了umain函数，这是所有应用程序执行的第一个C函数，它将调用应用程序的main函数，并在main函数结束后调用exit函数，而exit函数最终将调用sys_exit系统调用，让操作系统回收进程资源。
- user/libs/ulib.[ch]：实现了最小的C函数库，除了一些与系统调用无关的函数，其他函数是对访问系统调用的包装。
- user/libs/syscall.[ch]：用户层发出系统调用的具体实现。
- user/libs/stdio.c：实现cprintf函数，通过系统调用sys_putc来完成字符输出。
- user/libs/panic.c：实现\_\_panic/\_\_warn函数，通过系统调用sys_exit完成用户进程退出。

在make的最后一步执行了一个ld命令，把hello应用程序的执行码obj/\_\_user_hello.out连接在了ucore kernel的末尾。且ld命令会在kernel中会把\_\_user_hello.out的位置和大小记录在全局变量**\_binary_obj\_\_\_user_hello_out_start**和**\_binary_obj\_\_\_user_hello_out_size**中，这样这个hello用户程序就能够和ucore内核一起被 bootloader 加载到内存里中，并且通过这两个全局变量定位hello用户程序执行码的起始位置和大小。

#### 用户进程的虚拟地址空间
在tools/user.ld描述了用户程序的用户虚拟空间的执行入口虚拟地址：
```
SECTIONS {
    /* Load programs at this address: "." means the current address */
    . = 0x800020;
```

在tools/kernel.ld描述了操作系统的内核虚拟空间的起始入口虚拟地址：
```
SECTIONS {
    /* Load the kernel at this address: "." means the current address */
    . = 0xC0100000;
```
这样ucore把用户进程的虚拟地址空间分了两块:
- 一块与内核线程一样，是所有用户进程都共享的内核虚拟地址空间，映射到同样的物理内存空间中，这样在物理内存中只需放置一份内核代码，使得用户进程从用户态进入核心态时，内核代码可以统一应对不同的内核程序；
- 另外一块是用户虚拟地址空间，虽然虚拟地址范围一样，但映射到不同且没有交集的物理内存空间中。这样当ucore把用户进程的执行代码（即应用程序的执行代码）和数据（即应用程序的全局变量等）放到用户虚拟地址空间中时，确保了各个进程不会“非法”访问到其他进程的物理内存空间。

这样ucore给一个用户进程具体设定的虚拟内存空间（kern/mm/memlayout.h）如下所示：
```
 Virtual memory map:                                          Permissions
                                                              kernel/user

     4G ------------------> +---------------------------------+
                            |                                 |
                            |         Empty Memory (*)        |
                            |                                 |
                            +---------------------------------+ 0xFB000000
                            |   Cur. Page Table (Kern, RW)    | RW/-- PTSIZE
     VPT -----------------> +---------------------------------+ 0xFAC00000
                            |        Invalid Memory (*)       | --/--
     KERNTOP -------------> +---------------------------------+ 0xF8000000
                            |                                 |
                            |    Remapped Physical Memory     | RW/-- KMEMSIZE
                            |                                 |
     KERNBASE ------------> +---------------------------------+ 0xC0000000
                            |        Invalid Memory (*)       | --/--
     USERTOP -------------> +---------------------------------+ 0xB0000000
                            |           User stack            |
                            +---------------------------------+
                            |                                 |
                            :                                 :
                            |         ~~~~~~~~~~~~~~~~        |
                            |         ~~~~~~~~~~~~~~~~        |
                            :                                 :
                            |                                 |
                            ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
                            |       User Program & Heap       |
     UTEXT ---------------> +---------------------------------+ 0x00800000
                            |        Invalid Memory (*)       | --/--
                            |  - - - - - - - - - - - - - - -  |
                            |    User STAB Data (optional)    |
     USERBASE, USTAB------> +---------------------------------+ 0x00200000
                            |        Invalid Memory (*)       | --/--
     0 -------------------> +---------------------------------+ 0x00000000
(*) Note: The kernel ensures that "Invalid Memory" is *never* mapped.
    "Empty Memory" is normally unmapped, but user programs may map pages
    there if desired.

*/                            
```

#### 创建并执行用户进程
在确定了用户进程的执行代码和数据，以及用户进程的虚拟空间布局后，我们可以来创建用户进程了。在本实验中第一个用户进程是由第二个内核线程initproc通过把hello应用程序执行码覆盖到initproc的用户虚拟内存空间来创建的，相关代码如下所示：
```
 // kernel_execve - do SYS_exec syscall to exec a user program called by user_main kernel_thread
    static int
    kernel_execve(const char *name, unsigned char *binary, size_t size) {
    int ret, len = strlen(name);
    asm volatile (
        "int %1;"
        : "=a" (ret)
        : "i" (T_SYSCALL), "0" (SYS_exec), "d" (name), "c" (len), "b" (binary), "D" (size)
        : "memory");
    return ret;
   }

    #define __KERNEL_EXECVE(name, binary, size) ({                          \
            cprintf("kernel_execve: pid = %d, name = \"%s\".\n",        \
                    current->pid, name);                                \
            kernel_execve(name, binary, (size_t)(size));                \
        })

    #define KERNEL_EXECVE(x) ({                                             \
            extern unsigned char _binary_obj___user_##x##_out_start[],  \
                _binary_obj___user_##x##_out_size[];                    \
            __KERNEL_EXECVE(#x, _binary_obj___user_##x##_out_start,     \
                            _binary_obj___user_##x##_out_size);         \
        })
……
// init_main - the second kernel thread used to create kswapd_main & user_main kernel threads
static int init_main(void *arg) {
    #ifdef TEST
    KERNEL_EXECVE2(TEST, TESTSTART, TESTSIZE);
    #else
    KERNEL_EXECVE(hello);
    #endif
    panic("kernel_execve failed.\n");
    return 0;
}
```
**##的作用是参数的连接，把“exit”这个字符串连接到这个宏中的x对应位置**  
**#的作用是使一个东西字符串化**

Initproc的执行主体是init_main函数，这个函数在缺省情况下是执行宏KERNEL_EXECVE(hello)，而这个宏最终是调用kernel_execve函数来调用SYS_exec系统调用，由于ld在链接hello应用程序执行码时定义了两全局变量：
```
_binary_obj___user_hello_out_start：hello执行码的起始位置
_binary_obj___user_hello_out_size中：hello执行码的大小
```

kernel_execve把这两个变量作为SYS_exec系统调用的参数，让ucore来创建此用户进程。当ucore收到此系统调用后，将依次调用如下函数：
```
vector128(vectors.S) -->
__alltraps(trapentry.S) --> trap(trap.c) --> trap_dispatch(trap.c) --> syscall(syscall.c) --> sys_exec（syscall.c）--> do_execve(proc.c)
```
最终通过do_execve函数来完成用户进程的创建工作。此函数的主要工作流程如下：
- 为加载新的执行码做好**用户态内存空间清空**准备。如果mm不为NULL，则设置页表为内核空间页表，且进一步判断mm的引用计数减1后是否为0，如果为0，则表明没有进程再需要此进程所占用的内存空间，为此将根据mm中的记录，释放进程所占用户空间内存和进程页表本身所占空间。最后把当前进程的mm内存管理指针为空。由于此处的initproc是内核线程，所以mm为NULL，整个处理都不会做。
- **加载应用程序执行码到当前进程的新创建的用户态虚拟空间中**。这里涉及到读ELF格式的文件，申请内存空间，建立用户态虚存空间，加载应用程序执行码等。load_icode函数完成了整个复杂的工作。
- load_icode函数的主要工作就是给用户进程建立一个能够让用户进程正常运行的用户环境。此函数有一百多行，完成了如下重要工作：
1. 调用mm_create函数来申请进程的内存管理数据结构mm所需内存空间，并对mm进行初始化；
2. 调用setup_pgdir来申请一个页目录表所需的一个页大小的内存空间，并把描述ucore内核虚空间映射的内核页表（boot_pgdir所指）的内容拷贝到此新目录表中，最后让mm->pgdir指向此页目录表，这就是进程新的页目录表了，且能够正确映射内核虚空间；
3. 根据应用程序执行码的起始位置来解析此ELF格式的执行程序，并调用mm_map函数根据ELF格式的执行程序说明的各个段（代码段、数据段、BSS段等）的起始位置和大小建立对应的vma结构，并把vma插入到mm结构中，从而表明了用户进程的合法用户态虚拟地址空间；
4. 调用根据执行程序各个段的大小分配物理内存空间，并根据执行程序各个段的起始位置确定虚拟地址，并在页表中建立好物理地址和虚拟地址的映射关系，然后把执行程序各个段的内容拷贝到相应的内核虚拟地址中，至此应用程序执行码和数据已经根据编译时设定地址放置到虚拟内存中了；
- 需要**给用户进程设置用户栈**，为此调用mm_mmap函数建立用户栈的vma结构，明确用户栈的位置在用户虚空间的顶端，大小为256个页，即1MB，并分配一定数量的物理内存且建立好栈的虚地址<-->物理地址映射关系；
- 至此,进程内的内存管理vma和mm数据结构已经建立完成，于是把**mm->pgdir赋值到cr3寄存器中**，即**更新了用户进程的虚拟内存空间**，此时的initproc已经被hello的代码和数据覆盖，成为了第一个用户进程，但此时这个用户进程的执行现场还没建立好；
- 先清空进程的中断帧，再**重新设置进程的中断帧**，使得在执行中断返回指令“iret”后，能够让CPU转到用户态特权级，并回到用户态内存空间，使用用户态的代码段、数据段和堆栈，且能够跳转到用户进程的第一条指令执行，并确保在用户态能够响应中断；
- 至此，用户进程的用户环境已经搭建完毕。此时initproc将按产生系统调用的函数调用路径原路返回，执行中断返回指令“iret”（位于trapentry.S的最后一句）后，将切换到用户进程hello的第一条语句位置_start处（位于user/libs/initcode.S的第三句）开始执行。

#### 进程退出和等待进程
ucore分了两步来完成进程退出工作，首先，进程本身完成大部分资源的占用内存回收工作，然后父进程完成剩余资源占用内存的回收工作。为何不让进程本身完成所有的资源回收工作呢？这是因为进程要执行回收操作，就表明此进程还存在，还在执行指令，这就需要内核栈的空间不能释放，且表示进程存在的进程控制块不能释放。所以需要父进程来帮忙释放子进程无法完成的这两个资源回收工作。

为此在用户态的函数库中提供了exit函数，此函数最终访问sys_exit系统调用接口让操作系统来帮助当前进程执行退出过程中的部分资源回收。

首先，exit函数会把一个退出码error_code传递给ucore，ucore通过执行内核函数do_exit来完成对当前进程的退出处理，主要工作是回收当前进程所占的大部分内存资源，并通知父进程完成最后的回收工作，具体流程如下：

1. 如果current->mm != NULL，表示是用户进程，则开始回收此用户进程所占用的用户态虚拟内存空间；
- 首先执行“lcr3(boot_cr3)”，切换到内核态的页表上，这样当前用户进程目前只能在内核虚拟地址空间执行了，这是为了确保后续释放用户态内存和进程页表的工作能够正常执行；
- 如果当前进程控制块的成员变量mm的成员变量mm_count减1后为0（表明这个mm没有再被其他进程共享，可以彻底释放进程所占的用户虚拟空间了。），则开始回收用户进程所占的内存资源：
- 调用exit_mmap函数释放current->mm->vma链表中每个vma描述的进程合法空间中实际分配的内存，然后把对应的页表项内容清空，最后还把页表所占用的空间释放并把对应的页目录表项清空；
- 调用put_pgdir函数释放当前进程的页目录所占的内存；
- 调用mm_destroy函数释放mm中的vma所占内存，最后释放mm所占内存；
- 此时设置current->mm为NULL，表示与当前进程相关的用户虚拟内存空间和对应的内存管理成员变量所占的内核虚拟内存空间已经回收完毕；
2. 这时，设置当前进程的执行状态`current->state=PROC_ZOMBIE`，当前进程的退出码current->exit_code=error_code。此时当前进程已经不能被调度了，需要此进程的父进程来做最后的回收工作（即回收描述此进程的内核栈和进程控制块）；
3. 如果当前进程的父进程current->parent处于等待子进程状态： 
`current->parent->wait_state==WT_CHILD`，
则唤醒父进程（即执行“`wakup_proc(current->parent)`”），让父进程帮助自己完成最后的资源回收；
4. 如果当前进程还有子进程，则需要把这些子进程的父进程指针设置为内核线程initproc，且各个子进程指针需要插入到initproc的子进程链表中。如果某个子进程的执行状态是PROC_ZOMBIE，则需要唤醒initproc来完成对此子进程的最后回收工作。
5. 执行schedule()函数，选择新的进程执行。

那么父进程如何完成对子进程的最后回收工作呢？这要求父进程要执行wait用户函数或wait_pid用户函数，这两个函数的区别是，**wait函数等待任意子进程的结束通知**，而**wait_pid函数等待进程id号为pid的子进程结束通知**。这两个函数最终访问sys_wait系统调用接口让ucore来完成对子进程的最后回收工作，即回收子进程的内核栈和进程控制块所占内存空间，具体流程如下：
1. 如果pid!=0，表示只找一个进程id号为pid的退出状态的子进程，否则找任意一个处于退出状态的子进程；
2. 如果此子进程的执行状态不为PROC_ZOMBIE，表明此子进程还没有退出，则当前进程只好设置自己的执行状态为PROC_SLEEPING，睡眠原因为WT_CHILD（即等待子进程退出），调用schedule()函数选择新的进程执行，自己睡眠等待，如果被唤醒，则重复跳回步骤1处执行；
3. 如果此子进程的执行状态为PROC_ZOMBIE，表明此子进程处于退出状态，需要当前进程（即子进程的父进程）完成对子进程的最终回收工作，即首先把子进程控制块从两个进程队列proc_list和hash_list中删除，并释放子进程的内核堆栈和进程控制块。自此，子进程才彻底地结束了它的执行过程，消除了它所占用的所有资源。

#### 系统调用实现
用户进程只能在操作系统给它圈定好的“用户环境”中执行，但“用户环境”限制了用户进程能够执行的指令，即用户进程只能执行一般的指令，无法执行特权指令。如果用户进程想执行一些需要特权指令的任务，比如通过网卡发网络包等，只能让操作系统来代劳了。于是就需要一种机制来确保用户进程不能执行特权指令，但能够请操作系统“帮忙”完成需要特权指令的任务，这种机制就是系统调用。

采用系统调用机制为用户进程提供一个获得操作系统服务的统一接口层：
- 一来可简化用户进程的实现，把一些共性的、繁琐的、与硬件相关、与特权指令相关的任务放到操作系统层来实现，但提供一个简洁的接口给用户进程调用；
- 二来这层接口事先可规定好，且严格检查用户进程传递进来的参数和操作系统要返回的数据，使得让操作系统给用户进程服务的同时，保护操作系统不会被用户进程破坏。

从硬件层面上看，需要硬件能够支持在用户态的用户进程通过某种机制切换到内核态。

##### 初始化系统调用对应的中断描述符
在ucore初始化函数kern_init中调用了idt_init函数来初始化中断描述符表，并设置一个特定中断号的中断门，专门用于用户进程访问系统调用。此事由ide_init函数完成：
```
void
idt_init(void) {
    extern uintptr_t __vectors[];
    int i;
    for (i = 0; i < sizeof(idt) / sizeof(struct gatedesc); i ++) {
        SETGATE(idt[i], 0, GD_KTEXT, __vectors[i], DPL_KERNEL);
    }
    SETGATE(idt[T_SYSCALL], 1, GD_KTEXT, __vectors[T_SYSCALL], DPL_USER);
    lidt(&idt_pd);
}
```

在上述代码中，可以看到在执行加载中断描述符表lidt指令前，专门设置了一个特殊的中断描述符idt[T_SYSCALL]，它的特权级设置为DPL_USER，中断向量处理地址在__vectors[T_SYSCALL]处。这样建立好这个中断描述符后，一旦用户进程执行“INT T_SYSCALL”后，由于此中断允许用户态进程产生（注意它的特权级设置为DPL_USER），所以CPU就会从用户态切换到内核态，保存相关寄存器，并跳转到__vectors[T_SYSCALL]处开始执行，形成如下执行路径：
```
vector128(vectors.S) --> 
__alltraps(trapentry.S) --> trap(trap.c) --> trap_dispatch(trap.c) --> syscall(syscall.c)
```

##### 建立系统调用的用户库准备
在操作系统中初始化好系统调用相关的中断描述符、中断处理起始地址等后，还需在用户态的应用程序中初始化好相关工作，简化应用程序访问系统调用的复杂性。为此在用户态建立了一个中间层，即简化的libc实现，在user/libs/ulib.[ch]和user/libs/syscall.[ch]中完成了对访问系统调用的封装。用户态最终的访问系统调用函数是syscall，实现如下：
```
static inline int
syscall(int num, ...) {
    va_list ap;
    va_start(ap, num);
    uint32_t a[MAX_ARGS];
    int i, ret;
    for (i = 0; i < MAX_ARGS; i ++) {
        a[i] = va_arg(ap, uint32_t);
    }
    va_end(ap);

    asm volatile (
        "int %1;"
        : "=a" (ret)
        : "i" (T_SYSCALL),
          "a" (num),
          "d" (a[0]),
          "c" (a[1]),
          "b" (a[2]),
          "D" (a[3]),
          "S" (a[4])
        : "cc", "memory");
    return ret;
}
```

从中可以看出，应用程序调用的exit/fork/wait/getpid等库函数最终都会调用syscall函数，只是调用的参数不同而已，如果看最终的汇编代码会更清楚：
```
……
  34:    8b 55 d4               mov    -0x2c(%ebp),%edx
  37:    8b 4d d8               mov    -0x28(%ebp),%ecx
  3a:    8b 5d dc               mov    -0x24(%ebp),%ebx
  3d:    8b 7d e0               mov    -0x20(%ebp),%edi
  40:    8b 75 e4               mov    -0x1c(%ebp),%esi
  43:    8b 45 08               mov    0x8(%ebp),%eax
  46:    cd 80                  int    $0x80
  48:    89 45 f0               mov    %eax,-0x10(%ebp)
……
```

可以看到其实是把系统调用号放到EAX，其他5个参数a[0]\~a[4]分别保存到EDX/ECX/EBX/EDI/ESI五个寄存器中，及最多用6个寄存器来传递系统调用的参数，且系统调用的返回结果是EAX。比如对于getpid库函数而言，系统调用号（SYS_getpid=18）是保存在EAX中，返回值（调用此库函数的的当前进程号pid）也在EAX中。

##### 与用户进程相关的系统调用
在本实验中，与进程相关的各个系统调用属性如下所示：
|系统调用名  | 含义 | 具体完成服务的函数 |
|----|----|----|
|SYS_exit   | process exit  |  do_exit |
|SYS_fork   | create child process, dup mm  |  do_fork-->wakeup_proc |
|SYS_wait   | wait child process | do_wait |
|SYS_exec   | after fork, process execute a new program  | load a program and refresh the mm |
|SYS_yield  | process flag itself need resecheduling | proc->need_sched=1, then scheduler will rescheule this process |
|SYS_kill   | kill process  |  do_kill-->proc->flags \|= PF_EXITING, -->wakeup_proc-->do_wait-->do_exit |
|SYS_getpid | get the process's pid |  |

<br>
##### 系统调用的执行过程
与用户态的函数库调用执行过程相比，系统调用执行过程的有四点主要的不同：
- 不是通过“CALL”指令而是通过“INT”指令发起调用；
- 不是通过“RET”指令，而是通过“IRET”指令完成调用返回；
- 当到达内核态后，操作系统需要严格检查系统调用传递的参数，确保不破坏整个系统的安全性；
- 执行系统调用可导致进程等待某事件发生，从而可引起进程切换；

下面我们以getpid系统调用的执行过程大致看看操作系统是如何完成整个执行过程的。当用户进程调用getpid函数，最终执行到`INT T_SYSCALL`指令后，CPU根据操作系统建立的系统调用中断描述符，转入内核态，并跳转到vector128处（kern/trap/vectors.S），开始了操作系统的系统调用执行过程，函数调用和返回操作的关系如下所示：
```
vector128(vectors.S) --> 
__alltraps(trapentry.S) --> trap(trap.c) --> trap_dispatch(trap.c) --> syscall(syscall.c) --> sys_getpid(syscall.c) --> …… --> __trapret(trapentry.S)
```
在执行trap函数前，软件还需进一步保存执行系统调用前的执行现场，即把与用户进程继续执行所需的相关寄存器等当前内容保存到当前进程的中断帧trapframe中（注意，在创建进程是，把进程的trapframe放在给进程的内核栈分配的空间的顶部）。软件做的工作在vector128和__alltraps的起始部分：
```
vectors.S::vector128起始处:
  pushl $0
  pushl $128
......
trapentry.S::__alltraps起始处:
pushl %ds
  pushl %es
  pushal
……
```
自此，用于保存用户态的用户进程执行现场的trapframe的内容填写完毕，操作系统可开始完成具体的系统调用服务。在sys_getpid函数中，简单地把当前进程的pid成员变量做为函数返回值就是一个具体的系统调用服务。完成服务后，操作系统按调用关系的路径原路返回到__alltraps中。然后操作系统开始根据当前进程的中断帧内容做恢复执行现场操作。其实就是把trapframe的一部分内容保存到寄存器内容。恢复寄存器内容结束后，调整内核堆栈指针到中断帧的tf_eip处，这是内核栈的结构如下：
```
/* below here defined by x86 hardware */
    uintptr_t tf_eip;
    uint16_t tf_cs;
    uint16_t tf_padding3;
    uint32_t tf_eflags;
/* below here only when crossing rings */
    uintptr_t tf_esp;
    uint16_t tf_ss;
    uint16_t tf_padding4;
```
这时执行`IRET`指令后，CPU根据内核栈的情况回复到用户态，并把EIP指向tf_eip的值，即`INT T_SYSCALL`后的那条指令。这样整个系统调用就执行完毕了。

## 读load_icode有感
```
/* load_icode - load the content of binary program(ELF format) as the new content of current process
 * @binary:  the memory addr of the content of binary program
 * @size:  the size of the content of binary program
 * 读取一个二进制elf文件并为其设置执行场景，并执行
 */
static int
load_icode(unsigned char *binary, size_t size) {
    if (current->mm != NULL) {
        panic("load_icode: current->mm must be empty.\n");
    }

    int ret = -E_NO_MEM;
    struct mm_struct *mm;
    //(1) create a new mm for current process
    if ((mm = mm_create()) == NULL) {
        goto bad_mm;
    }
    //(2) create a new PDT, and mm->pgdir= kernel virtual addr of PDT
    if (setup_pgdir(mm) != 0) {
        goto bad_pgdir_cleanup_mm;
    }
    //(3) copy TEXT/DATA section, build BSS parts in binary to memory space of process
    struct Page *page;
    //(3.1) get the file header of the bianry program (ELF format)
    // 将二进制串转成描述elf的结构体
    struct elfhdr *elf = (struct elfhdr *)binary;
    //(3.2) get the entry of the program section headers of the bianry program (ELF format)
    // 获取elf头的起始地址
    struct proghdr *ph = (struct proghdr *)(binary + elf->e_phoff);
    // 代码段的头
    //(3.3) This program is valid?
    // 第一个实验中说了elf的这个域是ELF_MAGIC
    if (elf->e_magic != ELF_MAGIC) {
        ret = -E_INVAL_ELF;
        goto bad_elf_cleanup_pgdir;
    }

    uint32_t vm_flags, perm;
    struct proghdr *ph_end = ph + elf->e_phnum;
    for (; ph < ph_end; ph ++) {
    //(3.4) find every program section headers
    // 每一个程序段
        if (ph->p_type != ELF_PT_LOAD) {
          //程序段头里的这个程序段的类型，如可加载的代码、数据、动态链接信息等
            continue ;
        }
        if (ph->p_filesz > ph->p_memsz) {
            ret = -E_INVAL_ELF;
            goto bad_cleanup_mmap;
        }
        if (ph->p_filesz == 0) {
        // 这个段的大小
            continue ;
        }
    //(3.5) call mm_map fun to setup the new vma ( ph->p_va, ph->p_memsz)
        vm_flags = 0, perm = PTE_U;
        if (ph->p_flags & ELF_PF_X) vm_flags |= VM_EXEC;
        if (ph->p_flags & ELF_PF_W) vm_flags |= VM_WRITE;
        if (ph->p_flags & ELF_PF_R) vm_flags |= VM_READ;
        // 可读、可写、可执行？

        if (vm_flags & VM_WRITE) perm |= PTE_W;
        if ((ret = mm_map(mm, ph->p_va, ph->p_memsz, vm_flags, NULL)) != 0) {
            goto bad_cleanup_mmap;
        // 创建一个vma，并把这个vma加入到mm的list中
        }
        unsigned char *from = binary + ph->p_offset;
        size_t off, size;
        uintptr_t start = ph->p_va, end, la = ROUNDDOWN(start, PGSIZE);
        // 
        ret = -E_NO_MEM;

     //(3.6) alloc memory, and  copy the contents of every program section (from, from+end) to process's memory
(la, la+end)
        end = ph->p_va + ph->p_filesz;
     //(3.6.1) copy TEXT/DATA section of bianry program
     // 分配页
        while (start < end) {
            if ((page = pgdir_alloc_page(mm->pgdir, la, perm)) == NULL) {
                goto bad_cleanup_mmap;
            }
            off = start - la, size = PGSIZE - off, la += PGSIZE;
            if (end < la) {
                size -= la - end;
            }
            memcpy(page2kva(page) + off, from, size);
            start += size, from += size;
        }

      //(3.6.2) build BSS section of binary program
        end =  ph->p_va + ph->p_memsz;
        if (start < la) {
            /* ph->p_memsz == ph->p_filesz */
            if (start == end) {
                continue ;
            }
            off = start + PGSIZE - la, size = PGSIZE - off;
            if (end < la) {
                size -= la - end;
            }
            memset(page2kva(page) + off, 0, size);
            start += size;
            assert((end < la && start == end) || (end >= la && start == la));
        }
        while (start < end) {
            if ((page = pgdir_alloc_page(mm->pgdir, la, perm)) == NULL) {
                goto bad_cleanup_mmap;
            }
            off = start - la, size = PGSIZE - off, la += PGSIZE;
            if (end < la) {
                size -= la - end;
            }
            memset(page2kva(page) + off, 0, size);
            start += size;
        }
    }
    //(4) build user stack memory
    vm_flags = VM_READ | VM_WRITE | VM_STACK;
    if ((ret = mm_map(mm, USTACKTOP - USTACKSIZE, USTACKSIZE, vm_flags, NULL)) != 0) {
        goto bad_cleanup_mmap;
    }
    assert(pgdir_alloc_page(mm->pgdir, USTACKTOP-PGSIZE , PTE_USER) != NULL);
    assert(pgdir_alloc_page(mm->pgdir, USTACKTOP-2*PGSIZE , PTE_USER) != NULL);
    assert(pgdir_alloc_page(mm->pgdir, USTACKTOP-3*PGSIZE , PTE_USER) != NULL);
    assert(pgdir_alloc_page(mm->pgdir, USTACKTOP-4*PGSIZE , PTE_USER) != NULL);

    //(5) set current process's mm, sr3, and set CR3 reg = physical addr of Page Directory
    mm_count_inc(mm); // mm的count加1，计算有多少进程同时使用这个mm
    current->mm = mm; // 当前进程的mm是这个mm
    current->cr3 = PADDR(mm->pgdir); // 虚拟地址转换成物理地址
    lcr3(PADDR(mm->pgdir));

    //(6) setup trapframe for user environment
    struct trapframe *tf = current->tf;
    memset(tf, 0, sizeof(struct trapframe));
    /* LAB5:EXERCISE1 YOUR CODE
     * should set tf_cs,tf_ds,tf_es,tf_ss,tf_esp,tf_eip,tf_eflags
     * NOTICE: If we set trapframe correctly, then the user level process can return to USER MODE from kernel. S
o
     * tf_cs should be USER_CS segment (see memlayout.h)
     * tf_ds=tf_es=tf_ss should be USER_DS segment        
     * tf_esp should be the top addr of user stack (USTACKTOP)
     * tf_eip should be the entry point of this binary program (elf->e_entry)
     * tf_eflags should be set to enable computer to produce Interrupt
     */
    tf->tf_cs = USER_CS;
    tf->tf_ds = tf->tf_es = tf->tf_ss = USER_DS;
    tf->tf_esp = USTACKTOP;
    tf->tf_eip = elf->e_entry;
    tf->tf_eflags = 0x00000002 | FL_IF; // to enable interrupt
    //网上这里有的是这么写的，不知道为啥，我觉得应该只要FL_IF就够了，可能是我考虑不周
/*
    #define FL_IF           0x00000200  // Interrupt Flag
    tf->tf_eflags = FL_IF;
*/
    ret = 0;
out:
    return ret;
bad_cleanup_mmap:
    exit_mmap(mm);
bad_elf_cleanup_pgdir:
    put_pgdir(mm);
bad_pgdir_cleanup_mm:
    mm_destroy(mm);
bad_mm:
    goto out;
}
```

## 练习1：加载应用程序并执行
do_execv函数调用了load_icode函数（位于kern/process/proc.c中）来加载并解析一个处于内存中的ELF执行文件格式的应用程序，并建立了相应的用户内存空间来存放应用程序的代码段、数据段 等，且要设置好proc_struct结构中的成员变量trapframe中的内容，确保在执行此进程后，能够从应用程序设定的起始执行地址开始执行。

load_icode函数是由do_execve函数调用的，而该函数是exec系统调用的最终处理的函数，功能为将某一个指定的ELF可执行二进制文件加载到当前内存中来，然后当前进程执行这个可执行文件（先前执行的内容全部清空），而load_icode函数的功能则在于为执行新的程序初始化好内存空间，在调用该函数之前，do_execve中已经退出了当前进程的内存空间，改使用了内核的内存空间，这样使得对原先用户态的内存空间的操作成为可能；

由于最终是在用户态下运行的，所以需要将段寄存器初始化为用户态的代码段、数据段、堆栈段；
esp应当指向先前的步骤中创建的用户栈的栈顶；
eip应当指向ELF可执行文件加载到内存之后的入口处；
eflags中应当初始化为中断使能，注意eflags的第1位是恒为1的；
设置ret为0，表示正常返回；
见上边的函数代码。

首先在初始化IDT的时候，设置系统调用对应的中断描述符，使其能够在用户态下被调用，并且设置为trap类型。设置系统调用中断是用户态的。
```
    extern uintptr_t __vectors[];
    int i;
    for (i = 0; i < sizeof(idt) / sizeof(struct gatedesc); i ++) {
        SETGATE(idt[i], 0, GD_KTEXT, __vectors[i], DPL_KERNEL);
    }
    SETGATE(idt[T_SYSCALL], 1, GD_KTEXT, __vectors[T_SYSCALL], DPL_USER);
    lidt(&idt_pd);

/* *
 * Set up a normal interrupt/trap gate descriptor
 *   - istrap: 1 for a trap (= exception) gate, 0 for an interrupt gate
 *   - sel: Code segment selector for interrupt/trap handler
 *   - off: Offset in code segment for interrupt/trap handler
 *   - dpl: Descriptor Privilege Level - the privilege level required
 *          for software to invoke this interrupt/trap gate explicitly
 *          using an int instruction.
 * */
#define SETGATE(gate, istrap, sel, off, dpl) {               \
        (gate).gd_off_15_0 = (uint32_t)(off) & 0xffff;      \
        (gate).gd_ss = (sel);                                \
        (gate).gd_args = 0;                                 \
        (gate).gd_rsv1 = 0;                                 \
        (gate).gd_type = (istrap) ? STS_TG32 : STS_IG32;    \
        (gate).gd_s = 0;                                    \
        (gate).gd_dpl = (dpl);                              \
        (gate).gd_p = 1;                                    \
        (gate).gd_off_31_16 = (uint32_t)(off) >> 16;        \
    }
```
同样是在trap.c里，设置当计时器到点之后，也就是100个时钟周期之后，这个进程就是可以被重新调度的了，实现多线程的并发执行。
```
    case IRQ_OFFSET + IRQ_TIMER:
        ticks++;
        if(ticks>=TICK_NUM){
            assert(current != NULL);
            current->need_resched = 1;
            //print_ticks();
            ticks=0;
        }
        /* LAB5 YOUR CODE */
        /* you should upate you lab1 code (just add ONE or TWO lines of code):
         *    Every TICK_NUM cycle, you should set current process's current->need_resched = 1
         */
-
```
在proc_alloc函数中，额外对进程控制块中新增加的wait_state, cptr, yptr, optr成员变量进行初始化；
在alloc_proc(void)函数中，对新增的几个变量初始化
```
     //LAB5 YOUR CODE : (update LAB4 steps)
    /*
     * below fields(add in LAB5) in proc_struct need to be initialized
     *       uint32_t wait_state;                        // waiting state
     *       struct proc_struct *cptr, *yptr, *optr;     // relations between processes
         */
        proc->wait_state = 0;
        proc->cptr = proc->optr = proc->yptr = NULL;
```
在do_fork函数中，使用set_links函数来完成将fork的线程添加到线程链表中的过程，值得注意的是，该函数中就包括了将其加入list和对进程总数加1这一操作，因此需要将原先的这个操作给删除掉；
```
// set_links - set the relation links of process
static void
set_links(struct proc_struct *proc) {
    list_add(&proc_list, &(proc->list_link));
    proc->yptr = NULL;
    if ((proc->optr = proc->parent->cptr) != NULL) {
        proc->optr->yptr = proc;
    }
    proc->parent->cptr = proc;
    nr_process ++;
}

//LAB5 YOUR CODE : (update LAB4 steps)
/* Some Functions
 *    set_links:  set the relation links of process.  ALSO SEE: remove_links:  lean the relation links of process
 *    -------------------
 *    update step 1: set child proc's parent to current process, make sure current process's wait_state is 0
 *    update step 5: insert proc_struct into hash_list && proc_list, set the relation links of process
 */
// 1. call alloc_proc to allocate a proc_struct
    proc = alloc_proc();
    if(proc == NULL)
        goto fork_out;
// 2. call setup_kstack to allocate a kernel stack for child process
    proc->parent = current;
    assert(current->wait_state == 0);

    int status = setup_kstack(proc);
    if(status != 0)
        goto bad_fork_cleanup_kstack;
// 3. call copy_mm to dup OR share mm according clone_flag
    status = copy_mm(clone_flags, proc);
    if(status != 0)
        goto bad_fork_cleanup_proc;
// 4. call copy_thread to setup tf & context in proc_struct
    copy_thread(proc, stack, tf);
// 5. insert proc_struct into hash_list && proc_list
    proc->pid = get_pid();
    hash_proc(proc);
    set_links(proc);

// delete thses two lines !!!
    //nr_process ++;
    //list_add(&proc_list, &proc->list_link);
// delete thses two lines !!!

// 6. call wakeup_proc to make the new child process RUNNABLE
    wakeup_proc(proc);
// 7. set ret vaule using child proc's pid
    ret = proc->pid;
```

请在实验报告中描述当创建一个用户态进程并加载了应用程序后，CPU是如何让这个应用程序最终在用户态执行起来的。即这个用户态进程被ucore选择占用CPU执行（RUNNING态） 到具体执行应用程序第一条指令的整个经过。

- 在经过调度器占用了CPU的资源之后，用户态进程调用了exec系统调用，从而转入到了系统调用的处理例程；
- 调用中断处理例程之后，最终控制权转移到了syscall.c中的syscall函数，然后根据系统调用号转移给了sys_exec函数，在该函数中调用了上文中提及的do_execve函数来完成指定应用程序的加载；
- 在do_execve中进行了若干设置，包括退出当前进程的页表，换用kernel的PDT之后，使用load_icode函数，完成了对整个用户线程内存空间的初始化，包括堆栈的设置以及将ELF可执行文件的加载，之后通过current->tf指针修改了当前系统调用的trapframe，使得最终中断返回的时候能够切换到用户态，并且同时可以正确地将控制权转移到应用程序的入口处；
- 在完成了do_exec函数之后，进行正常的中断返回的流程，由于中断处理例程的栈上面的eip已经被修改成了应用程序的入口处，而cs上的CPL是用户态，因此iret进行中断返回的时候会将堆栈切换到用户的栈，并且完成特权级的切换，并且跳转到要求的应用程序的入口处；
- 接下来开始具体执行应用程序的第一条指令；

> 本问题参考：https://www.jianshu.com/p/8c852af5b403

## 练习2：父进程复制自己的内存空间给子进程
创建子进程的函数do_fork在执行中将拷贝当前进程（即父进程）的用户内存地址空间中的合法内容到新进程中（子进程），完成内存资源的复制。具体是通过copy_range函数（位于 kern/mm/pmm.c中）实现的，请补充copy_range的实现，确保能够正确执行。

- 父进程调用fork()，进入中断处理机制，最终交由syscall函数进行处理；
- 在syscall，根据系统调用号，交由sys_fork函数处理；
- 进一步调用do_fork函数，这个函数创建了子进程、并且将父进程的内存空间复制给子进程；
- 在do_fork函数中，调用copy_mm进行内存空间的复制，在该函数中，进一步调用了dup_mmap。dup_mmap中遍历父进程的所有合法虚拟内存空间，并且将这些空间的内容复制到子进程的内存空间中去；
- 在copy_range函数中，对需要复制的内存空间按照页为单位从父进程的内存空间复制到子进程的内存空间中去；

遍历父进程指定的某一段内存空间中的每一个虚拟页，如果这个虚拟页存在，为子进程对应的同一个地址（但是页目录表是不一样的，因此不是一个内存空间）也申请分配一个物理页，然后将前者中的所有内容复制到后者中去，然后为子进程的这个物理页和对应的虚拟地址（事实上是线性地址）建立映射关系；而在本练习中需要完成的内容就是内存的复制和映射的建立，具体流程如下：

- 找到父进程指定的某一物理页对应的内核虚拟地址；
- 找到需要拷贝过去的子进程的对应物理页对应的内核虚拟地址；
- 将前者的内容拷贝到后者中去；
- 为子进程当前分配这一物理页映射上对应的在子进程虚拟地址空间里的一个虚拟页；

```
/* copy_range - copy content of memory (start, end) of one process A to another process B
 * @to:    the addr of process B's Page Directory
 * @from:  the addr of process A's Page Directory
 * @share: flags to indicate to dup OR share. We just use dup method, so it didn't be used.
 *
 * CALL GRAPH: copy_mm-->dup_mmap-->copy_range
 */
int
copy_range(pde_t *to, pde_t *from, uintptr_t start, uintptr_t end, bool share) {
    assert(start % PGSIZE == 0 && end % PGSIZE == 0);
    assert(USER_ACCESS(start, end));
    // copy content by page unit.
    do {
        //call get_pte to find process A's pte according to the addr start
        pte_t *ptep = get_pte(from, start, 0), *nptep;
        if (ptep == NULL) {
            start = ROUNDDOWN(start + PTSIZE, PTSIZE);
            continue ;
        }
        //call get_pte to find process B's pte according to the addr start. If pte is NULL, just alloc a PT
        if (*ptep & PTE_P) {
            if ((nptep = get_pte(to, start, 1)) == NULL) {
                return -E_NO_MEM;
            }
        uint32_t perm = (*ptep & PTE_USER);
        //get page from ptep
        struct Page *page = pte2page(*ptep);
        // alloc a page for process B
        struct Page *npage=alloc_page();
        assert(page!=NULL);
        assert(npage!=NULL);
        int ret=0;
        /* LAB5:EXERCISE2 YOUR CODE
         * replicate content of page to npage, build the map of phy addr of nage with the linear addr start
         *
         * Some Useful MACROs and DEFINEs, you can use them in below implementation.
         * MACROs or Functions:
         *    page2kva(struct Page *page): return the kernel vritual addr of memory which page managed (SEE pmm.
h)
         *    page_insert: build the map of phy addr of an Page with the linear addr la
         *    memcpy: typical memory copy function
         *
         * (1) find src_kvaddr: the kernel virtual address of page
         * (2) find dst_kvaddr: the kernel virtual address of npage
         * (3) memory copy from src_kvaddr to dst_kvaddr, size is PGSIZE
         * (4) build the map of phy addr of  nage with the linear addr start
         */
         char *src_kvaddr = page2kva(page); 
         //找到父进程需要复制的物理页在内核地址空间中的虚拟地址，这是由于这个函数执行的时候使用的时内核的地址空间
         char *dst_kvaddr = page2kva(npage); 
         // 找到子进程需要被填充的物理页的内核虚拟地址
        memcpy(dst_kvaddr, src_kvaddr, PGSIZE); 
        // 将父进程的物理页的内容复制到子进程中去
        page_insert(to, npage, start, perm); 
        // 建立子进程的物理页与虚拟页的映射关系
        assert(ret == 0);
        }
        start += PGSIZE;
    } while (start != 0 && start < end);
    return 0;
}            
```

## 练习3：阅读分析源代码，理解进程执行 fork/exec/wait/exit 的实现，以及系统调用的实现（不需要编码）

1. fork：在执行了fork系统调用之后，会执行正常的中断处理流程，到中断向量表里查系统调用入口，最终将控制权转移给syscall，之后根据系统调用号执行sys_fork函数，进一步执行了上文中的do_fork函数，新进程的进程控制块进行初始化、设置、以及调用copy_mm将父进程内存中的内容到子进程的内存的复制工作，然后调用wakeup_proc将新创建的进程放入可执行队列（runnable），之后由调度器对子进程进行调度。

2. exec：在执行了exec系统调用之后，会执行正常的中断处理流程，到中断向量表里查系统调用入口，最终将控制权转移给syscall，之后根据系统调用号执行sys_exec函数，进一步执行了上文中的do_execve函数。在该函数中，会对内存空间进行清空，然后调用load_icode将将要执行的程序加载到内存中，然后调用lcr3(boot_cr4)设置好中断帧，使得最终中断返回之后可以跳转到指定的应用程序的入口处，就可以正确执行了。

3. wait：在执行了wait系统调用之后，会执行正常的中断处理流程，到中断向量表里查系统调用入口，最终将控制权转移给syscall，之后根据系统调用号执行sys_wait函数，进一步执行了的do_wait函数，在这个函数中，找一个当前进程的处于ZOMBIE状态的子进程，如果有的话直接将其占用的资源释放掉即可；如果找不到，则将我这个进程的状态改成SLEEPING态，并且标记为等待ZOMBIE态的子进程，然后调用schedule函数将其当前线程从CPU占用中切换出去，直到有对应的子进程结束来唤醒这个进程为止。

4. exit：在执行了exit系统调用之后，会执行正常的中断处理流程，到中断向量表里查系统调用入口，最终将控制权转移给syscall，之后根据系统调用号执行sys_exit函数，进一步执行了的do_exit函数，首先将释放当前进程的大多数资源，然后将其标记为ZOMBIE态，然后调用wakeup_proc函数将其父进程唤醒（如果父进程执行了wait进入SLEEPING态的话），然后调用schedule函数，让出CPU资源，等待父进程进一步完成其所有资源的回收；

问题回答

请分析fork/exec/wait/exit在实现中是如何影响进程的执行状态的？

fork不会影响当前进程的执行状态，但是会将子进程的状态标记为RUNNALB，使得可以在后续的调度中运行起来；
exec不会影响当前进程的执行状态，但是会修改当前进程中执行的程序；
wait系统调用取决于是否存在可以释放资源（ZOMBIE）的子进程，如果有的话不会发生状态的改变，如果没有的话会将当前进程置为SLEEPING态，等待执行了exit的子进程将其唤醒；
exit会将当前进程的状态修改为ZOMBIE态，并且会将父进程唤醒（修改为RUNNABLE），然后主动让出CPU使用权；
