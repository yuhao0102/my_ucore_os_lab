# 第十七讲 同步互斥
## 背景
独立进程：不和其他进程共享资源或状态，具有确定性（输入决定结果）；可重现（能够重现起始条件）；调度顺序不重要。

并发进程：多个进程之间有资源共享；不确定性；不可重现。某些情况下调度的不一致会造成结果的不一致，也可能出现不可重现性。程序错误也可能是间歇性发生的。

进程需要与计算机中的其他进程和设备合作。有几个好处：
1. 共享资源。多个用户使用同一个计算机；
2. 提高速度。IO和计算可以重叠；程序可划分为多个模块放在多个处理器上并行执行；
3. 模块化。将大程序分解成小程序。

并发创建新进程时的标识分配：程序调用fork()创建进程，操作系统需要分配一个新的且唯一的进程ID，在内核中，这个系统调用会执行`new_pid = next_pid++`。

原子操作是一次不存在任何中断或失败的操作。要么成功要么不执行，不会出现部分执行的情况。操作系统需要利用同步机制在并发执行的同时，保证一些操作是原子操作。

## 现实生活中的同步问题
利用原子操作实现一个锁。
- Lock.Acquire()
    - 在锁被释放前一直等待，然后获得锁；
    - 如果两个线程都在等待同一个锁，那如果锁被释放了，只有一个进程能得到锁
- Lock.Release()
    - 解锁并唤醒任何等待中的进程。
- 过程：
    - 进入临界区
    - 操作
    - 退出临界区

进程之间的交互关系：相互感知程度。
- 相互不感知（完全不了解其他进程）：独立
- 间接感知（双方与第三方交互）：通过共享合作
- 直接感知（直接交互，如通信）：通过通信合作

可能会出现如下几种：
- 互斥：一个进程占用，则其他进程不能使用
- 死锁：多个进程各自占用部分资源，形成循环等待
- 饥饿：其他进程轮流占用资源，一个进程一直得不到资源

## 临界区和禁用硬件中断同步方法
临界区是互斥执行的代码，进入区检查进程进入临界区的条件是否成立，进入之前设置相应“正在访问临界区”的标志；退出区清除“正在访问临界区”标志。

临界区访问规则：
- 空闲则入：没有进程在临界区时任何进程可以进入；
- 忙则等待：有进程在临界区，则其他进程均不能进入临界区；
- 有限等待：等待进入临界区的进程不能无线等待；
- 让权等待：不能进入临界区的进程，需要及时释放CPU；

实现方法：
- 禁用硬件中断：没有中断和上下文切换，因此没有并发，硬件将中断处理延迟到中断被启用之后，现在计算机体系结构都提供指令来实现禁用中断，进入临界区时禁止所有中断，退出临界区时使能所有中断。这种办法有局限性，关中断之后进程无法停止，也可能导致其他进程处于饥饿状态；临界区可能很长，无法确定相应中断所需的时间。
- 软件方法：两个线程，T0和T1，线程可以通过共享一些共有变量来同步行为。
- 借用操作系统的支持采用更高级的抽象方法




## 基于软件的同步方法
## 高级抽象的同步方法