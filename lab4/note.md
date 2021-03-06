# 第十一讲 进程和线程
## 11.1 进程的概念
进程是一个具有一定功能的程序在一个**数据集合**中的一次**动态执行**过程。源代码到可执行文件再到加载到进程地址内存空间（堆、栈、代码段）。进程浩瀚了正在运行的一个程序的所有状态的信息，进程是由：
- 代码
- 数据
- 状态寄存器：CPU状态CR0、指令指针IP等
- 通用寄存器：AX、BX、CX...
- 进程占用系统资源：打开文件、已分配内存

特点：
- 动态性：动态创建
- 并发性：独立调度并占用处理机运行
- 独立性：不同进程相互工作不影响
- 制约性：因访问共享数据和资源或进程间同步产生制约

进程是处于运行状态程序的抽象，程序是一个静态的可执行文件，进程是执行中的程序，是程序+执行状态；同一个程序的多次执行过程对应不同进程；进程执行需要内存和CPU。  
进程是动态的，程序是动态的，程序是有序代码的集合，进程是程序的俄执行，进程有核心态和用户态；进程是暂时的，程序是永久的，进程的组成包括**程序**、**数据**和**进程控制块**。

## 11.2 进程控制块（PCB）
是操作系统控制进程运行的信息集合。操作系统用PCB来描述进程的基本情况和运行变化的过程。PCB是**进程存在的唯一标志**。
- 进程创建：生成该进程的PCB；
- 进程终止：回收PCB；
- 进程的组织管理：通过对PCB的组织管理实现。

进程控制块内容：
- 进程标识信息
- 处理机现场保存：从进程地址空间抽取PC、SP、其他寄存器保存
- 进程控制信息：调度和状态信息（调度进程和处理机使用情况）、进程间通信信息（通信相关的标识）、存储管理信息（指向进程映像存储空间数据结构）、进程所用资源（进程使用的系统资源，文件等）、有关数据结构链接信息（与PCB有关的进程队列）

进程控制块的组织：
- 链表：同一状态的进程其PCB组织成一个链表，多个状态对应不同链表；
- 索引表：同一状态的进程归入一个索引表，由索引指向PCB，多个状态对应多个不同的索引表。

## 11.3 进程状态
操作系统为了维护进程执行中的变化来维护进程的状态。进程的生命周期分为：
- 进程创建：创建PCB、拷贝数据。引起进程创建主要有：系统初始化、用户请求创建进程、正在执行的进程执行了创建进程的调用；
- 进程就绪：放入等待队列中等待运行；
- 进程执行：内核选择一个就绪进程，占用处理机并执行；
- 进程等待：进程执行的某项条件不满足，比如请求并等待系统服务、启动某种操作无法马上完成，**只有进程自身知道何时需要等待某种事件的发生**；
- 进程抢占：高优先级的进程就绪或进程执行时间片用完；
- 进程唤醒：被阻塞的进程需要的资源可满足，进程只能被别的进程或操作系统唤醒；
- 进程结束：把进程执行占用的资源释放，有几种可能：正常、错误退出、致命错误、强制退出。

N个进程交替运行，假定进程1执行sleep()，内核里调用计时器，进程1把当前进程占用寄存器的状态保存，切换进程2，如果计时器到点了，计时器产生中断，保存进程2的状态，恢复进程1的状态。

## 11.4 三状态进程模型
核心是：
- 就绪：进程获得了除了处理机之外的所有资源，得到处理机即可运行；
- 运行：进程正在处理机上执行；
- 等待：进程在等待某一事件在等待。

辅助状态两种：
- 创建：一个进程正在被创建，还未被转到就绪状态之前的状态；
- 结束：进程正在从系统中消失的状态，这是因为进程结束或其他原因所导致。

状态转换：
- 创建 -> 就绪：进程被创建并完成初始化，变成就绪状态；
- 就绪 -> 运行：处于就绪状态的进程被调度程序选中，分配到处理机上运行；
- 运行 -> 结束：进程表示它已经完成或因为出错，当前运行今晨会由操作系统作结束处理；
- 运行 -> 就绪：处于运行状态的进程在其运行过程中，由于分配给它的处理机时间片用完而让出处理机；
- 运行 -> 等待：当进程请求某资源且必须等待时；
- 等待 -> 就绪：进程等待的某事件到来时，它从阻塞状态变到就绪状态；

## 11.5 挂起进程模型
处于挂起状态的进程映像在磁盘上，目的是减少进程占用内存。

- 等待挂起：进程在外存并等待某事件的发生；
- 就绪挂起：进程在外存，但是只要进入内存即可运行；
- 挂起：把进程从内存转到外存

增加了内存的转换：
- 等待 -> 等待挂起：没有进程处于就绪状态或就绪状态要求更多内存资源；
- 就绪到就绪挂起：有高优先级等待进程（系统认为很快就绪）和低优先级就绪进程；
- 运行 -> 就绪挂起：对抢先式分时系统，当有高优先级等待挂起进程因事件出现而进入就绪挂起；

从外存转到内存的转换：激活
- 就绪挂起 -> 就绪：没有就绪进程或挂起就绪进程优先级高于就绪进程；
- 等待挂起 -> 等待：进程释放了足够内存，并有高优先级的等待挂起进程；

状态队列：有操作系统维护一组队列，表示系统所有进程的当前状态。
根据进程状态不同，进程PCB加入不同队列，进程状态切换时，加入不同队列。

## 11.6 线程的概念
- 为什么要引入线程
在进程内部增加一类实体，满足实体之间可以并发执行且实体之间可以共享相同的地址空间。线程是进程的一部分，描述指令流执行状态，它是进程中指令执行流的最小单元，是CPU调度的单位。这种剥离为并发提供了可能，描述了在进程资源环境中的指令流执行状态；进程扮演了资源分配的角色。  
原来只有一个指令指针，现在有多个堆栈和指令指针。线程=进程-共享资源。  
但是如果一个线程有异常，会导致其所属进程的所有线程都崩。  
- 比较
- 进程是资源分配单位，线程是CPU调度单位；
- 进程有一个完整的资源平台，线程只独享指令流执行的必要资源，如寄存器和栈；
- 线程具有就绪、等待和运行三种基本状态和其转移关系；
- 线程能减少并发执行的时间和空间开销：
1. 线程创建时间短；
2. 线程的终止时间比进程短；
3. 同一进程的线程切换时间比进程短；
4. 由于同一进程的各个线程共享内存和文件资源，可不通过内核进行直接通信。

## 11.7 用户线程
三种实现方式：  
- 用户线程：在用户空间实现，通过用户级的线程库函数完成线程的管理。在操作系统内核中仍然只有进程控制块来描述处理机的调度的情况，操作系统并不感知应用态有多线程的支持，多线程的支持是用户的函数库支持的。在应用程序内部通过构造相应的线程控制块
来控制一个进程内部多个线程的交替执行和同步。  
这种方法不依赖操作系统内核，用于不支持线程的多线程的操作系统。每个进程有私有的**线程控制块(TCB)**，TCB由线程库函数维护；同一进程的用户线程切换速度快，无需用户态/核心态的切换，且允许每个进程有自己的线程调度算法。   
缺点就是不支持基于线程的处理机抢占， 除非当前运行的线程主动放弃CPU，他所在进程的其他线程无法抢占CPU。  
POSIX Pthreads、Math C-threads、Solaris threads  
- 内核线程：在内核中实现，Windows、Solaris、Linux  
- 轻量级进程：在内核中实现，支持用户进程。

## 11.8 内核线程
内核通过系统调用完成的线程机制。由内核维护PCB和TCB，线程执行系统调用而阻塞不影响其他线程，以线程为单位的进程调度会更合理。  
**轻权进程**：内核支持的用户线程，一个进程有多个轻量级进程，每个轻权进程由一个单独的内核线程来支持。在内核支持线程，轻权进程来绑定用户线程。  
用户线程与内核线程的对应关系：一对一、多对一、多对多。

# 第十二讲 进程控制
## 12.1 进程切换
上下文切换，暂停当前运行的进程，从当前运行状态转变成其他状态，调度另一个进程从就绪状态变成运行状态，在此过程中实现**进程上下文的保存和快速切换**。维护进程生命周期的信息（寄存器等）。

进程控制块PCB：内核为每个进程维护了对应的PCB，将相同状态的进程的PCB放置在同一个队列中。

ucore中的进程控制块结构proc_struct：
- 进程ID、父进程ID，组ID；
- 进程状态信息、地址空间起始、页表起始、是否允许调度；
- 进程所占用的资源struct mm_struct* mm；
- 现场保护的上下文切换的context；
- 用于描述当前进程在哪个状态队列中的指针，等。

ucore的切换流程：开始调度 -> 清除调度标志 -> 查找就绪进程 -> 修改进程状态 -> 进程切换switch_to()。
switch_to用汇编写成。。。

## 12.2 进程创建
Windows进程创建API：CreateProcess  
Unix进程创建系统调用：fork/exec，fork()把一个进程复制成两个进程，exec()用新程序重写当前进程。

fork()的地址空间复制：fork调用对子进程就是在调用时间对父进程地址空间的一次复制。执行到fork时，复制一份，只有PID不同。系统调用exec()加载新程序取代当前运行的程序。加载进来后把代码换掉。

ucore中的do_fork：分配进程控制块数据结构、创建堆栈、复制内存数据结构、设置进程标识等。操作系统没有新的任务执行，则创建空闲进程，在proc_init中分配idleproc需要的资源，初始化idleproc的PCB。

fork的开销昂贵，在fork中内存复制是没用的，子进程将可能关闭打开的文件和连接，所以可以将fork和exec联系起来。产生了vfork，创建进程时不再创建一个同样的内存映像，用时再加载，一些时候称为轻量级fork。这时子进程应立即执行exec。现在使用写时复制技术。

## 12.3 进程加载
应用程序通过exec加载可执行文件，允许进程加载一个完全不同的程序，并从main开始执行。不同系统加载可执行文件的格式不同，并且允许进程加载时指定启动参数（argc,argv），exec调用成功时，它与运行exec之前是相同的进程，但是运行了不同的程序，且代码段和堆栈重写。主要是可执行文件格式的识别，有sys_exec、do_execv、load_icode函数。

ucore中第一个用户态进程是由proc_init创建的，执行了init_main创建内核线程，创建了shell程序。

## 12.4 进程等待与退出
父子进程的交互，完成子进程的资源回收。

子进程通过exit()向父进程返回一个值，父进程通过wait()接受并处理这个返回值。wait()父进程先等待，还是子进程先做exit()，这两种情况会导致它下面的处理有一些区别。

如果有子进程存活，也就是说父进程创建的子进程还有子进程，那这时候父进程进入等待状态，等待子进程的返回结果，父进程先执行wait，等到子进程执行的时候它执行exit()，这是exit ()是在wait之后执行的。这时候，子进程的exit()退出，唤醒父进程，父进程由等待状态回到就绪状态，父进程就处理子进程的返回的这个返回值，这是wait在前exit()在后的情况。

如果不是这样那就有一种情况，就是有僵尸子进程等待，就是子进程先执行exit()，这时它返回一个值，等待父进程的处理，exit()在前，如果子进程还一直处在这个等待的状态，在这里等待父进程的处理，父进程的wait就直接返回，如果有多个的话就从其中一个返回它的值。

进程的有序终止exit()，完成资源回收。
- 调用参数作为进程的结果；
- 关闭所有打开的文件等占用资源；
- 释放内存，释放进程相关的内核数据结构；
- 检查父进程是否存活，如存活则保留结果的值直到父进程需要他。
- 清理所有等待的僵尸进程。

# 第十三讲 实验四 内核线程管理
## 13.1 总体介绍
了解内核线程创建执行的管理过程。了解内核线程的切换和基本调度过程，对TCB充分了解。

## 13.2 关键数据结构
struct proc_struct：TCB
- pid和name代表了标识符。
- state、runs、need_reshed代表了状态和是否需要调度
- cr3不太需要，因为共用进程的页表
- kstack代表了堆栈
- mm_struct不太需要，在ucore的统一管理下
- context是通常说的上下文，基本都是寄存器，代表了当前线程的状态
- trap_frame代表中断产生时的信息（硬件保存）、段寄存器的信息（软件保存）
- 一些list，父进程的信息和线程控制块的链表
- 基于hash的list，查找对应的线程比较快

## 13.3 执行流程
kern_init最开始初始化，proc_init完成一系列的创建内核线程并执行。

创建第0号内核线程idleproc：
- alloc_proc创建TCB的内存块
-init idle_proc，设置pid、stat、kstack等

创建第1个内核线程：
- initproc：
- keep trapframe调用了do_fork，copy_thread等，如何跳到入口正确执行？是用户态还是内核态？
- init_proc
- init kernel stack，可以放到两个list中执行了
- 开始调度执行
- 找到线程队列中哪个是处于就绪的，切换switch kstack、页表、上下文，根据trapframe跳到内核线程的入口地址，开始执行函数。

## 13.4 实际操作
关注proc_init创建第0、1号线程。switch_to完成两个内核线程的切换。

# 第十四讲 实验五 用户进程管理
## 14.1 总体介绍
第一个用户进程如何创建、进程管理的实现机制、系统调用的框架实现。  
构造出第一个用户进程：建立用户代码/数据段 ---> 创建内核线程 ---> 创建用户进程“壳” ---> 填写用户进程 ---> 执行用户进程(特权级转换) ---> 完成系统调用 ---> 结束用户进程(资源回收)

## 14.2 进程的内存布局
内核虚拟内存布局有一部分是对实际物理空间的映射，0xC0000000到0xF8000000，映射为物理空间。一个Page Tabel，0xFAC00000到0xB0000000，一开始只是管理内核空间的映射关系，有了用户进程后，页表需要扩展。

进程虚拟内存空间：  
Ivalid Memory     
User Stack------------0xB0000000    
...........   
User Program & Heap---0x00800000    
Invalid Memory    
User STAB Data(optional，调试信息)    
Invalid Memory    

Invalid Memory一旦访问为非法，确保访问到这些是产生page fault，使之不能随意访问。

## 14.3 执行ELF格式的二进制代码-do_execve的实现
do_execve建好一个壳并把程序加载进来。本实验用到一个PCB（process control block），其实是跟上一个实验的TCB一样的。

首先，把之前的内存空间清空，只留下PCB，换成自己的程序。把cr3这个页表基址指向boot_cr3内核页表；把进程内存管理区域清空，对应页表清空，导致内存没有了；**load_icode**加载执行程序。

## 14.4 执行ELF格式的二进制代码-load_icode的实现
前边已经把内存管理清空了，先创建一个新内存管理空间mm_create和新页表setup_pgdir；填上我执行代码的内容，找到要加载的程序的代码段和数据段，根据代码段和数据段的虚拟地址通过mm_map完成对合法空间的建立；从程序的内存区域拷贝过来，建立物理地址和虚拟地址的映射关系；准备all_zero的内存；设置相应堆栈空间（用户态空间），使用mm_map建立；把页表的起始地址换成新建好的页表的起始地址。

完成trapframe的设置。trapframe保存了打断的中断状态保存，完成特权级转变，从kernel转换到user。

x86特权级：从ring 0 ---> ring 3，一个ring 0栈空间，构造一个信息使得执行iret时能回到用户态，重新设置ss和cs，从ring0到ring3。

用户进程有两个栈，用户栈和内核栈，通过系统调用转化。

## 14.5 进程复制
父进程如何构造子进程？
一个函数叫do_fork，是一个内核函数，完成用户空间的拷贝。首先，父进程创建进程控制块，初始化kernel stack，分配页空间和内核里的虚地址。copy_mm为新进程建立新虚存空间。copy_range拷贝父进程的内存到新进程。拷贝父进程的trapframe到新进程。添加新的proc_struct到proc_list并唤醒新进程。执行完do_fork后父进程得到子进程的pid，子进程得到0。

## 14.6 内存管理的copy-on-write机制
进程A通过do_fork创建进程B，二者重用一段空间，使得空间占用量大大减少，如果是只读的话没问题。一旦某进程做了写操作，因为页表设置成只读，则产生page_fault，触发copy-on-write机制，真正为子进程复制页表。进程创建的开销大大减小，且有效减少空间。

一个物理页可能被多个虚拟页引用，这个个数很重要，因为在进程运行时可能会出现换入换出，如何进行有效换入换出，有可能那个页既在内存中也在虚存中。

dup_mmap完成内存管理的复制。