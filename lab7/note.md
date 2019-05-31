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

## 基于软件的同步方法
- 软件方法：两个线程，T0和T1，线程可以通过共享一些共有变量来同步行为。
    - 采用共享变量，设置一个共享变量表示允许进入临界区的线程；
    - 设置一个共享变量数组，描述每个变量是否在临界区中，先判断另一个线程的flag是否是1，如果可以进入了，设置自己的flag；可能会同时等待或同时进入；
    - Peterson算法：turn表示该哪个进程进入临界区，flag[]表示进程是否准备好进入临界区。在进入区进程i要设置flag[i]=true，且turn=j，判断（flag[i] && turn==j），如果j没有申请进入，则i直接进去没问题。如果j也申请了，看谁先向trun里写数据，谁先写谁进入，由总线仲裁决定先后顺序！
    - N线程时，采用Eisenberg和McGuire算法，采用一个处理循环。
    - 基于软件的方法很复杂，是一个忙等待

## 高级抽象的同步方法
- 借用操作系统的支持采用更高级的抽象方法，例如，锁、信号量等，用硬件原语来实现
- 锁：一个二进制变量（锁定，解锁），Acquire和Release，使用锁控制临界区访问。
- 原子操作指令：CPU体系结构中一类特殊的指令，把若干操作合成一个原子操作，不会出现部分执行的情况
    - 测试和置位（TS），从内存中读取，测试值是否为1并返回T/F，内存单元置为1。
    - 交换指令：交换内存中的两个值。

使用TS指令实现自旋锁：
```
class Lock {
	int value = 0;
}
Lock::Acquire() {
	while(test_and_set(value))
		; // spin
}
Lock::Release() {
	value = 0;
}
```
用TS指令把value读出来，向里边写入1。

- 如果锁被释放，那么TS指令读取0并将值设置为1
    - 锁被设置为忙并且需要等待完成
- 如果锁处于忙状态，那么TS指令读取1并将指令设置为1
    - 不改变锁的状态并且需要循环

无忙等待锁：
```
class Lock {
	int value = 0;
	WaitQueue q;
}
Lock::Acquire() {
	while(test_and_set(value)){
		add this TCP to wait queue
		schedule();
	}
}
Lock::Release() {
	value = 0;
	remove one thread t from q
	wakeup(t)
}
```

原子操作指令锁的特征：
- 优点：
    - 适用于单处理器或共享内存的多处理器中任意数量的进程
    - 支持多临界区
- 缺点：
    - 忙等待的话占用了CPU时间
    - 可能导致饥饿，进程离开临界区时有多个等待进程的话？
    - 可能**死锁**，低优先级的进程占用了临界区，但是请求访问临界区的高优先级进程获得了处理器并等待临界区。


# 第十八讲 信号量与管程
## 信号量
多线程的引入导致了资源的竞争，同步是协调多线程对共享数据的访问，在任何时候只能有一个线程执行临界区代码。

信号量是操作系统提供的协调共享资源访问的方法，软件同步是平等线程间的一种同步协商机制。信号量是由OS负责管理的，OS作为管理者，地位高于进程。用信号量表示一类资源，信号量的大小表示资源的可用量。

信号量是一种抽象数据类型，由一个整型变量（共享资源数目）和两个原子操作组成。
- P()（荷兰语尝试减少）
    - sem减一
    - 如sem<0，进入等待，否则继续
- V()（荷兰语增加）
    - sem加一
    - 如sem<=0，唤醒一个等待进程    

信号量是被保护的整型变量，初始化完成后只能通过PV操作修改，是由操作系统保证PV操作是原子操作的。

P操作可能阻塞，V操作不会阻塞。P操作中sem可以等于0，但是如果小于0的话，说明我没有资源了，把这个进程放入等待队列，并且阻塞。退出时执行V操作，如果sem++后还小于0，则说明还有等着的，就把一个进程唤醒开始执行。
```
class Semaphore{
	int sem;
	WaitQueue q;
}
Semaphore::P(){
	sem --;
	if(sem<0){
		Add this thread t to q;
		block(p)
	}
}
Semaphore::V(){
	sem++;
	if(sem<=0){
		remove a thread t from q;
		wakeup(t)
	}
}
```
它的原子性是操作系统保证的，执行不会被打断。

## 信号量使用
两种：二进制信号量，资源数目是0或1；资源信号量，资源数目为任意非负值。

一种是临界区的互斥访问。每类资源设置一个信号量，对应一个临界区，信号量初值为1，
```
mutex = new Semaphore(1)

mutex->P();
Critical Section
mutex->V()
```
第一个进程进来之后，mutex是0了，第二个进程再执行到P操作时，mutex变成-1，则会等待。第一个进程执行结束后，执行V操作，-1变成0，这时候唤醒第二个进程。

必须成对使用P()和V()操作。P()保证互斥访问，V()操作保证使用后及时释放。

一种是条件同步，初值设置为0。事件出现时设置为1。这个事件就相当于是一种资源。
```
condition = new Semaphore(0)
```

生产者-消费者：一个或多个生产者在生成数据后放在缓冲区总，单个消费者从缓冲区中取出数据，任何时刻只能有一个生产者或消费者可访问缓冲区（互斥关系），也就是缓冲区是一个临界区。缓冲区空时必须等待生产者（条件同步），缓冲区满时生产者必须等待消费者（条件同步）。

三个信号量：二进制信号量mutex描述互斥关系；资源信号量fullBuffer和emptyBuffer代表了条件同步关系。

刚开始时缓冲区都是空的，所以fullBuffers为0，emptyBuffers为n
```
class BounderBuffer{
	mutex = new Semaphore(1);
	fullBuffers = new Semphore(0);
	emptyBuffers = new Semphore(n);
}
```

mutex实现了对缓冲区的互斥访问，但是只是这样是不够的，先检查是否有空缓冲区，有的话则检查是否有另外的消费者占用缓冲区。
```
BounderBuffer::Deposit(c){
	emptyBuffers->P();
	mutex->P();
	Add c to the buffer
	mutex->V();
	fullBuffers->V();//生产者写了之后就释放一个资源
}
BounderBuffer::Remove(c){
	fullBuffers->P();
	mutex->P();
	Remove c from buffer
	mutex->V();
	emptyBuffers->V();//消费者用了一个之后释放一个
}


```

## 管程
在管程内部使用了条件变量，管程是一种用于多线程互斥访问共享资源的程序结构，采用了面向对象的方法，简化了线程间的同步控制，在任意时刻最多只有一个线程执行管程代码。正在管程中的线程可临时放弃管程的互斥访问，等待事件出现时恢复。

收集现在要同步的进程之间共享的数据，放到一起处理。在入口加一个互斥访问的锁，任何一个线程到临界区后排队，挨个进入。管理共享数据的并发访问。需要共享资源时对应相应的条件变量，使用管程中的程序。

条件变量是管程内的等待机制，进入管程的线程因资源占用而进入等待，每个条件变量表示一种等待原因，对应一个等待队列。两个操作：
- Wait()：将自己阻塞到等待队列中，唤醒一个等待者或释放管程的互斥访问。
- Signal()：将等待队列中的一个线程唤醒；如果等待队列为空，则相当于空操作。
```
Class Condition{
	int numWaiting = 0;
	WaitQueue q;
}
Condition::Wait(lock){
	numWaiting ++;
	Add this thread t to q;
	release();
	schedule();
	require(lock);
}
Condition::Signal(){
	if(numWaiting > 0){
		Remove a thread t from q;
		wakeup(t);
		numWaiting --;
	}
}

```
numWaiting为正表示有线程处于等待状态；把它自己放到等待队列中，释放管程使用权，开始调度。在Signal中，把一个进程从等待队列中拿出来，开始执行，numWaiting减一，等待的线程数目减少。

用信号量解决生产者-消费者问题的话，生产者消费者各对应一个函数，其他地方要使用的话直接调用这两个函数即可。首先放到一个管程里，这是由管程进入的申请和释放，如果没有空的，就在条件变量上等待。
```
class BoundedBuffer{
	...
	Lock lock;
	int count = 0;
	Condition notFull, notEmpty;
}
BoundedBuffer::Deposit(c){
	lock->Acquire();
	while(count == n)
		notFull.Wait(&lock);
	Add c to the buffer;
	count ++;
	notEmpty.Signal();
	lock->Release();
}
BoundedBuffer::Remove(c){
	lock->Acquire();
	while(count == 0)
		notEmpty.Wait(&lock);
	Remove c from buffer;
	count --;
	notFull.Signal();
	lock->Release();
}
```
管程可以把PV操作集中在一个函数里。

## 哲学家就餐问题
```
#define N 5
semphore fork[N];
void philosopher(int i){
	while(TRUE){
		think();
		if(i%2 == 0){
			P(fork[i]);
			P(fork[(i+1)%N]);
		} else{
			P(fork[(i+1)%N]);
			P(fork[i]);
		}
		eat();
		V(fork[i]);
		V(fork[(i+1)%N]);		
	}
}
```

## 读者-写者问题
共享数据的两种使用者：读者只读取数据，不修改；写者读取和修改数据。

有三种情况：
- 读读允许：同一时刻允许多个读者同时读
- 读写互斥：没有读者时写者才能写，没有写者时读者才能读
- 写写互斥：没有其他写者时写者才能写

用信号量描述每个约束。信号量WriteMutex是控制读写操作的互斥，初始化为1.读者计数Rcount是对正在读操作的读者数目，初始化为0。信号量CountMutex控制对读者计数的互斥修改，初始化为1。
Writer：
```
P(WriteMutex);
	write();
V(WriteMutex);
```
Reader:
```
P(CountMutex);
	if(Rcount == 0)
		P(WriteMutex);
	++Rcount;
V(CountMutex);
read();
P(CountMutex);
	--Rcount;
	if(Rcount == 0)
		V(WriteMutex);
	++Rcount;
V(CountMutex);
```

管程实现读者-写者问题：
```
Database::Read(){
	StartRead(); 
	//Wait until no writers;
	read database;
	DoneRead();
	//checkout - wakeup waiting writers;
}
```
```
Database::Write(){
	Wait until no reader/writer;
	write database;
	checkout - wakeup waiting reader/writer
}
```
状态变量。正在读和正在写只有一个大于等于0
```
AR = 0;  # of active reader
AW = 0;  # of active writer
WR = 0;  # of waiting reader
WW = 0;  # of waiting writer
Lock lock;
Condition okToRead, okToWrite
```
```
Private Database::StartRead(){
	lock.Acquire();
	while(AW + WW > 0){//写者优先
		WR++;
		okToRead.wait(&lock);
		WR--;
	}
	AR++;
	lock.Release()
}
```
```
Private Database::DoneRead(){
	lock.Acquire();
	AR --;
	if(AR==0 && WW>0) //没有读者，写者在等
		okToWrite.Signal();
	lock.Release();
}
```
```
Private Database::StartWrite(){
	lock.Acquire();
	while(AW + AR > 0){//有正在写的写者或正在读的读者
		WW++;
		okToWrite.wait(&lock);
		WW--;
	}
	AW++;
	lock.Release()
}
```
```
Private Database::DoneWrite(){
	lock.Acquire();
	AW --;
	if(WW>0) //写者优先
		okToWrite.Signal();
	else if(WR > 0)
		okToRead.broadcase();
	lock.Release();
}
```

# 第十九讲 实验七 同步互斥
## 总体介绍

## 底层支撑
定时器：进程睡眠，进入等待状态（do_sleep）。可以添加一个timer。

时钟中断时会遍历timer链表，看哪个进程的定时器到期了。
```
typedef struct{
	unsigned int expires;
	struct proc_struct* proc;
	list_entry_t timer_link;
} timer_t;
```
屏蔽中断完成了互斥的保护，使得这个进程不会被调度或打断。有一个Eflag寄存器，有一个bit叫做Interrupt Enable Flag，这个flag如果置成1，当前允许中断，置成0表示不允许中断。两个指令CLI和STI分别屏蔽中断和使能中断。uCore中使用`local_intr_save`和`local_intr_restore`封装。

等待项和等待队列：
```
typedef struct {
	struct proc_struct* proc;
	uint32_t wakeup_flags;//等待的原因
	wait_queue_t* wait_queue;//等待项在哪个队列中
	list_entry_t wait_link;
} wait_t
typedef struct {
	list_entry_t wait_head;
} wait_queue_t;
```

## 信号量设计实现
```
class Semaphore{
	int sem;
	WaitQueue q;
}
Semaphore::P(){
	sem --;
	if(sem<0){
		Add this thread t to q;
		block(t);
	}
}
Semaphore::V(){
	sem++;
	if(sem<=0){
		Remove a thread t from q;
		wakeup(t);
	}
}
```
## 管程和条件变量设计实现
```
typedef struct monitor{
	semaphore_t mutex;
	semaphore_t next;
	int next_count;
	condvar_t *cv;
}
```

## 哲学家就餐问题

# 第十九讲 实验七 同步互斥
第二十讲 死锁和进程通信
## 死锁概念
由于竞争资源或通信关系，两个或更多线程在执行中弧线，永远相互等待只能由其他进程引发的事件。

进程访问资源的流程：资源类型有R1、R2、R3等，每类资源Ri有Wi个实例，进程访问资源时先申请空闲的资源，再占用，最后释放资源。

可重用资源是不能被删除且在任何时刻都只能有一个进程使用，一个进程释放之后其他进程就可以使用了，比如CPU，文件、数据库等，可以被多个进程交替使用。可能出现死锁。

消耗资源：一个进程创建，并有其他进程使用，比如消息等，可能出现死锁。

资源分配图描述了资源和进程之间的分配和占用关系，是一个有向图。一类顶点是系统中的进程，另一类顶点是资源；一类有向边是资源请求边，另一类有向边是资源分配边。如果有循环等待的话，就会出现死锁。但是有循环也可能不会出现死锁。

出现死锁的条件：
- 互斥：任何时刻只能由一个进程使用一个资源实例，如果资源是共享的不会互斥的则不会死锁；
- 持有并等待：进程保持至少一个资源并正在等待获取其他进程持有的资源；
- 非抢占：资源只在进程使用后自愿放弃，不可以强行剥夺；
- 循环等待：存在等待进程集合，0等1，1等2，。。。n-1等n，n等0，类似这样。

## 死锁处理方法
- 死锁预防：确保系统永远不会进入死锁状态，四个必要条件的任何一个去掉都可以避免死锁，但是这样的话资源利用率低；
- 死锁避免：在使用前进行判断，只允许不会出现死锁的进程请求资源；
- 死锁检测和恢复：在检测到死锁后，进行恢复；
- 通常由应用进程来处理死锁，操作系统忽略死锁的存在。

死锁预防：采用某种机制，限制并发进程对资源的请求，使系统不满足死锁的必要条件。
- 比如可以把互斥的共享资源封装成可以同时访问的，比如打印机，加上缓冲区，在打印机内部协调先后；
- 持有并等待，进程请求资源时，不能占用其他任何资源，想申请资源时，必须把全部资源都申请到，也可以在进程开始执行时一次请求所有需要的资源，资源利用效率低；
- 非抢占：如进程请求不能立即分配的资源，则立即释放自己已占有的资源，只有能同时获取到所有需要资源时，才执行分配操作；
- 循环等待：对资源排序，进程需要按照顺序请求资源，可能先申请的资源后续才用到；

死锁避免：利用额外的先验信息，在分配资源时判断是否会出现死锁，如果可能会出现死锁，则不分配。要求进程声明资源需求的最大数目，限定提供与分配的资源数目，确保满足进程的最大需求，且动态检查资源分配状态，确保不会出现死锁。

进程请求资源时，系统判断是否处于安全状态。
- 针对所有已占用进程，存在安全序列；
- 序列<P1,P2,P3...,Pn>是安全的，则Pi要求的资源<=当前可用资源+所有Pj持有资源（j<\i），如果Pi的资源不能立即分配，则要等待。

## 银行家算法
判断并保证系统处于安全状态。
- n=线程数量，m=资源类型数量；
- Max（总需求量）：n\*m矩阵，线程Ti最多请求类型Rj的资源Max[i,j]个实例
- Available(剩余空闲量)：长度为m的向量，当前有Available[i]个类型Ri的资源实例可用
- Allocation(已分配量)：n\*m矩阵，线程Ti当前分配了Allocation[i,j]个Rj的实例
- Need(未来需求量)：n\*m矩阵，线程Ti未来需要Need[i,j]个Rj资源实例；
- Need[i,j]=Max[i,j]-Allcation[i,j]

安全状态判断：
1.  Work 和 Finish 分别是长度为 m 和 n 的向量初始化： Work = Available，Finish = false for i = 1,2,...,n
2. 寻找线程 Ti ，Finish[i] = false，Need[i] <= Work，找到 Need 比 Work 小的线程 i ，如果没有找到符合条件的 Ti ，转4
3. Work = Work + Allocation[i] ，Finish[i] = true，线程i的资源需求量小于当前系统剩余空闲资源，所以配置给他再回收。转2
4. 如果所有线程Ti满足Finish[i]=true，则系统处于安全状态。
5. 这种迭代循环到最后，则是安全的

初始化：Requesti：线程Ti的资源请求向量，Requesti[j]：线程Ti请求资源Rj的实例

循环：
1. 如果Requesti < Need[i]，转到2，否则拒绝资源申请，因为县城已经超过了其最大要求；
2. 如果Requesti <= Available，转到3，否则Ti必须等待，因为资源部可用；
3. 通过安全状态判断是否分配资源给Ti，生成一个需要判断状态是否安全的资源分配环境：
    - Available=Available-Requesti
    - Allocation[i] = Allocation[i]+Requesti
    - Need[i] = Need[i]-Requesti

## 死锁检测
允许系统进入死锁状态，并维护一个资源分配图，周期性调用死锁检测算法，如果有死锁，就调用死锁处理。

- Available：长度为m的向量，表示每种类型可用资源的数量；
- Allocation：一个n\*m矩阵，表示当前分配给各个进程每种类型资源的数量，当前Pi拥有资源Rj的Allocation[i,j]个实例。

死锁监测算法：
1. Work是系统中的空闲资源量，Finish时线程是否结束。Work = Available，Allocation[i] > 0时，Finish[i] = false；否哦则Finish[i] = true；
2. 寻找线程Ti满足Finish[i] = false且Requesti <= Work，线程没结束且能满足线程资源请求量。
3. Work = Work + Allocation[i]，Finish[i] = true，转到2。
4. 如果某个Finish[i] = false，则系统会死锁。

死锁检测的使用：
- 多长时间检测一次
- 多少进程需要回滚
- 难以分辨造成死锁的关键进程

死锁恢复：
- 终止所有的死锁进程
- 一次终止一个进程，看还会不会死锁
- 终止进程的顺序应该是
    - 进程优先级
    - 进程已运行的时间和还需运行的时间
    - 进程已占用资源
    - 进程完成所需要的资源
    - 进程终止数目
    - 进程是交互还是批处理
方法
    - 选择被抢占的资源
    - 进程回退

## 进程通信（IPC）概念
IPC提供两个基本操作：
- 发送：send(message)
- 接收：recv(message)

流程：
- 建立通信链路
- 通过send/recv交换

通信方式：
- 间接通信：在通信进程和内核之间建立联系，一个进程把信息发送到内核的消息队列中，另一个进程读取，接受发送的时间可以不一样。通过操作系统维护的消息队列通信，每个消息队列有一个唯一的标识，只有共享了相同消息队列的进程，才能够通信。
    - 链接可以单向，也可以双向
    - 每对进程可以共享多个消息队列
    - 创建消息队列、通过消息队列收发消息、撤销消息队列
    - send(A, message)、recv(A, message)，A是消息队列
    - 阻塞发送是发送方发送后进入等待，直到成功发送
    - 阻塞接受是接收后进入等待，直到成功接受
    - 非阻塞发送是发送方发送后返回
    - 非阻塞接受是没有消息发送时，接收者在请求接受消息后，接受不到消息。

- 直接通信：两个进程同时存在，发方向共享信道里发送，收方读取。进程必须正确的命名接收方。
    - 一般自动建立链路
    - 一条链路对应一对通信进程
    - 每对进程之间只有一个链路存在
    - 链路可能单向，也可以双向

进程发送的消息在链路上可能有三种缓冲方式：
- 0容量：发送方必须等待接收方
- 有限容量：通信链路缓冲队列满了，发送方必须等待
- 无限容量：发送方不需等待

## 信号和管道
信号是进程间软件中断通知和处理机制，如果执行过程中有意外需要处理，则需要信号，Ctrl-C可以使进程停止，这个处理是通过信号实现。如SIGKILL，SIGSTOP等。

信号的接收处理：
- 捕获：执行进程指定的信号处理函数被调用
- 忽略：执行操作系统的缺省处理，例如进程终止和挂起等
- 屏蔽：禁止进程接受和处理信号，可能是暂时的。

传送的信息量小，只有一个信号类型，只能做快速的响应知己。

1. 首先进程启动时注册相应的信号处理例程到操作系统；
2. 其他程序发出信号时，操作系统分发信号到进程的信号处理函数；
3. 进程执行信号处理函数。

管道：进程间基于内存文件的通信机制，内存中建立一个临时文件，子进程从父进程继承文件描述符，缺省文件描述符：0 1 2

进程不知道另一端，可能时从键盘、文件等。

系统调用：
- 读管道read(fd,buffer,nbytes)
- 写管道write(fd,buffer,nbytes)
- 创建管道pipe(rgfd)，rgfd时两个文件描述符组成的数组，rgfd[0]是读文件描述符，rgfd[1]是写文件描述符。利用继承的关系在两个进城之间继承文件描述符。

## 消息队列和共享内存
消息队列是操作系统维护的字节序列为基本单位的间接通信机制，若干个进程可以发送到消息队列中，每个消息是一个字节序列，相同标识的消息组成先进先出顺序的队列。
系统调用如下：
- msgget(key,flags)：获取消息队列标识
- msgsnd(QID,buf,size,flags)：发送消息
- msgrcv(QID,buf,size,flags)：接收消息

消息队列独立于进程，进程结束了之后消息队列可以继续存在，实现两个不同生命周期的进程之间的通信。

共享内存是把同一个物理内存区域同时映射到多个进程的内存地址空间的通信机制。每个进程都有私有内存地址空间，需要明确设置共享内存段。同一进程的线程总是共享相同的内存地址空间。