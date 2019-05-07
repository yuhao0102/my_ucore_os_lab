# 实验三
## 实验内容
在实验二的基础上，借助页表机制和实验一中涉及的中断异常处理机制，完成Pgfault异常处理和FIFO页替换算法的实现，结合磁盘提供的缓存空间，从而能够支持虚存管理，提供一个比实际物理内存空间“更大”的虚拟内存空间给系统使用。  
这个实验与实际操作系统中的实现比较起来要简单，不过需要了解实验一和实验二的具体实现。实际操作系统系统中的虚拟内存管理设计与实现是相当复杂的，涉及到与进程管理系统、文件系统等的交叉访问。

## 简单原理
> copy from gitbook  
通过内存地址虚拟化，可以使得软件在没有访问某虚拟内存地址时不分配具体的物理内存，而只有在实际访问某虚拟内存地址时，操作系统再动态地分配物理内存，建立虚拟内存到物理内存的页映射关系，这种技术称为按需分页（demand paging）。  

> 把不经常访问的数据所占的内存空间临时写到硬盘上，这样可以腾出更多的空闲内存空间给经常访问的数据；当CPU访问到不经常访问的数据时，再把这些数据从硬盘读入到内存中，这种技术称为页换入换出（page swap in/out）。这种内存管理技术给了程序员更大的内存“空间”，从而可以让更多的程序在内存中并发运行。

参考ucore总控函数kern_init的代码，在调用完成虚拟内存初始化的vmm_init函数之前，需要首先调用pmm_init函数完成物理内存的管理，调用pic_init函数完成中断控制器的初始化，调用idt_init函数完成中断描述符表的初始化。  
在调用完idt_init函数之后，将进一步调用新函数**vmm_init、ide_init、swap_init**。  
do_pgfault函数会申请一个空闲物理页，并建立好虚实映射关系，从而使得这样的“合法”虚拟页有实际的物理页帧对应。  
ide_init就是完成对用于页换入换出的硬盘（简称swap硬盘）的初始化工作。完成ide_init函数后，ucore就可以对这个swap硬盘进行读写操作了。

vmm设计包括两部分：mm_struct（mm）和vma_struct（vma）。mm是具有相同PDT的连续虚拟内存区域集的内存管理器。 vma是一个连续的虚拟内存区域。 vma中存在线性链接列表，mm的vma的redblack链接列表。（redblack是啥？）
建立mm_struct和vma_struct数据结构。当访问内存产生pagefault异常时，可获得访问的内存的方式（读或写）以及具体的虚拟内存地址，这样ucore就可以查询此地址，看是否属于vma_struct数据结构中描述的合法地址范围中，如果在，则可根据具体情况进行请求调页/页换入换出处理；如果不在，则报错。  
两种数据结构：
```
struct mm_struct {
    // 链接所有属于同一页目录表的虚拟内存空间
    list_entry_t mmap_list;
    // 指向当前正在使用的虚拟内存空间，直接使用这个指针就能找到下一次要用到的虚拟空间
    struct vma_struct *mmap_cache;
    pde_t *pgdir; // 第一级页表的起始地址，即页目录表项PDT。通过访问pgdir可以查找某虚拟地址对应的页表项是否存在以及页表项的属性等
    int map_count; // 记录了链接了的vma_struct个数，共享了几次
    void *sm_priv; // 指向记录页访问情况的链表头。
};

struct vma_struct {
	// 描述应用程序对虚拟内存“需求”
    struct mm_struct *vm_mm; // 指向更高抽象层次的数据结构
    // the set of vma using the same PDT
    uintptr_t vm_start; // 连续地址虚拟内存空间的起始位置
    uintptr_t vm_end; // 连续地址虚拟内存空间的结束位置
    uint32_t vm_flags; // 标志属性（读/写/执行）
    //link将一系列虚拟内存空间连接起来
    list_entry_t list_link;
};
vm_flags：
#define VM_READ 0x00000001 //只读
#define VM_WRITE 0x00000002 //可读写
#define VM_EXEC 0x00000004 //可执行
```
具体函数：
```
// mm_create -  alloc a mm_struct & initialize it.
struct mm_struct * mm_create(void) {
    struct mm_struct *mm = kmalloc(sizeof(struct mm_struct));
    if (mm != NULL) {
        list_init(&(mm->mmap_list));
        mm->mmap_cache = NULL;
        mm->pgdir = NULL;
        mm->map_count = 0;

        if (swap_init_ok) swap_init_mm(mm);
        else mm->sm_priv = NULL;
    }
    return mm;
}

// mm_destroy - free mm and mm internal fields
void mm_destroy(struct mm_struct *mm) {
    list_entry_t *list = &(mm->mmap_list), *le;
    while ((le = list_next(list)) != list) {
        list_del(le);
        kfree(le2vma(le, list_link),sizeof(struct vma_struct));  //kfree vma
    }
    kfree(mm, sizeof(struct mm_struct)); //kfree mm
    mm=NULL;
}
```
设备驱动程序或者内核模块中动态开辟内存，不是用malloc，而是kmalloc ,vmalloc，
释放内存用的是kfree,vfree，kmalloc函数返回的是虚拟地址(线性地址)。

kmalloc特殊之处在于它分配的内存是物理上连续的,这对于要进行DMA的设备十分重要。
而用vmalloc分配的内存只是线性地址连续,物理地址不一定连续,不能直接用于DMA。vmalloc函数的工作方式类似于kmalloc，只不过前者分配的内存虚拟地址是连续的，而物理地址则无需连续。

通过vmalloc获得的页必须一个一个地进行映射，效率不高， 因此，只在不得已(一般是为了获得大块内存)时使用。vmalloc函数返回一个指针，指向逻辑上连续的一块内存区，其大小至少为size。在发生错误 时，函数返回NULL。
```
// vma_create - 新建一个vma_struct并且初始化(地址范围： vm_start~vm_end)
struct vma_struct * vma_create(uintptr_t vm_start, uintptr_t vm_end, uint32_t vm_flags) {
    struct vma_struct *vma = kmalloc(sizeof(struct vma_struct));

    if (vma != NULL) {
        vma->vm_start = vm_start;
        vma->vm_end = vm_end;
        vma->vm_flags = vm_flags;
    }
    return vma;
}

```
## Page Fault异常处理
处理该异常主要用do_pgfault函数，当启动分页机制以后，如果一条指令或数据的虚拟地址所对应的物理页框不在内存中或者访问的类型有错误（比如写一个只读页或用户态程序访问内核态的数据等），就会发生页访问异常。产生页访问异常的原因主要有：
> 目标页帧不存在（页表项全为0，即该线性地址与物理地址尚未建立映射或者已经撤销)；  
> 相应的物理页帧不在内存中（页表项非空，但Present标志位=0，比如在swap分区或磁盘文件上)；  
> 不满足访问权限（此时页表项P标志=1，但低权限的程序试图访问高权限的地址空间，或者有程序试图写只读页面）。

当出现上面情况之一，那么就会产生页面page fault（#PF）异常。CPU会把产生异常的线性地址存储在CR2中，并且把表示页访问异常类型的值（简称页访问异常错误码，errorCode）保存在中断栈中。CR2是页故障线性地址寄存器，保存最后一次出现页故障的全32位线性地址。CR2用于发生页异常时报告出错信息。产生页访问异常后，CPU把引起页访问异常的线性地址装到寄存器CR2中，并给出了出错码errorCode，说明了页访问异常的类型。操作系统中对应的中断服务例程可以检查CR2的内容，从而查出线性地址空间中的哪个页引起本次异常。

CPU在当前**内核栈**保存当前被打断的程序现场，即依次压入当前被打断程序使用的EFLAGS，CS，EIP，errorCode；由于页访问异常的中断号是0xE，CPU把异常中断号0xE对应的中断服务例程的地址（vectors.S中的标号vector14处）加载到CS和EIP寄存器中，开始执行中断服务例程。

这时ucore开始处理异常中断，首先需要保存硬件没有保存的寄存器。在vectors.S中的标号vector14处先把中断号压入内核栈，然后再在trapentry.S中的标号__alltraps处把DS、ES和其他通用寄存器都压栈。自此，被打断的程序执行现场（context）被保存在内核栈中。接下来，在trap.c的trap函数开始了中断服务例程的处理流程，大致调用关系为：
> trap --> trap_dispatch --> pgfault_handler --> do_pgfault

ucore中do_pgfault函数是完成页访问异常处理的主要函数，它根据从CPU的控制寄存器CR2中获取的页访问异常的物理地址以及根据errorCode的错误类型来查找此地址是否在某个VMA的地址范围内以及是否满足正确的读写权限，如果在此范围内并且权限也正确，这认为这是一次**合法访问，但没有建立虚实对应关系**。所以需要分配一个空闲的内存页，并修改页表完成虚地址到物理地址的映射，刷新TLB，然后调用iret产生软中断，返回到产生页访问异常的指令处重新执行此指令。如果该虚地址不在某VMA范围内，则认为是一次非法访问。

## 页面置换机制的实现
当缺页中断发生时，操作系统把应用程序当前需要的数据或代码放到内存中来，然后重新执行应用程序产生异常的访存指令。如果在把硬盘中对应的数据或代码调入内存前，操作系统发现物理内存已经没有空闲空间了，这时操作系统必须把它认为“不常用”的页换出到磁盘上去，以腾出内存空闲空间给应用程序所需的数据或代码。 

- 先进先出：选择在内存中驻留时间最久的页予以淘汰。将调入内存的页按照调入的先后顺序链接成一个队列，队列头指向内存中驻留时间最久的页，队列尾指向最近被调入内存的页。因为那些常被访问的页，往往在内存中也停留得最久，结果它们因变“老”而不得不被置换出去。FIFO算法的另一个缺点是，它有一种异常现象（Belady现象），即在增加放置页的页帧的情况下，反而使页访问异常次数增多。

- 时钟替换算法：是LRU算法的一种近似实现。时钟页替换算法把各个页面组织成环形链表的形式，类似于一个钟的表面。然后把一个指针（简称当前指针）指向最老的那个页面，即最先进来的那个页面。另外，时钟算法需要在页表项（PTE）中设置了一位访问位来表示此页表项对应的页当前是否被访问过。当该页被访问时，CPU中的MMU硬件将把访问位置“1”。当操作系统需要淘汰页时，对当前指针指向的页所对应的页表项进行查询，如果访问位为“0”，则淘汰该页，如果该页被写过，则还要把它换出到硬盘上；如果访问位为“1”，则将该页表项的此位置“0”，继续访问下一个页。该算法近似地体现了LRU的思想，且易于实现，开销少，需要硬件支持来设置访问位。时钟页替换算法在本质上与FIFO算法是类似的，不同之处是在时钟页替换算法中跳过了访问位为1的页。

- 改进时钟页替换算法：在时钟置换算法中，淘汰一个页面时只考虑了页面是否被访问过，但在实际情况中，还应考虑被淘汰的页面是否被修改过。因为淘汰修改过的页面还需要写回硬盘，使得其置换代价大于未修改过的页面，所以**优先淘汰没有修改的页**，减少磁盘操作次数。改进的时钟置换算法除了**考虑页面的访问情况，还需考虑页面的修改情况**。即该算法不但希望淘汰的页面是最近未使用的页，而且还希望被淘汰的页是在主存驻留期间其页面内容未被修改过的。这需要为每一页的对应页表项内容中增加一位引用位和一位修改位。当该页被访问时，CPU中的MMU硬件将把访问位置“1”。当该页被“写”时，CPU中的MMU硬件将把修改位置“1”。这样这两位就存在四种可能的组合情况：（0，0）表示最近未被引用也未被修改，首先选择此页淘汰；（0，1）最近未被使用，但被修改，其次选择；（1，0）最近使用而未修改，再次选择；（1，1）最近使用且修改，最后选择。该算法与时钟算法相比，可进一步减少磁盘的I/O操作次数。

## 页面置换机制
### 可以被换出的页
只有映射到用户空间且被用户程序直接访问的页面才能被交换，被内核直接使用的内核空间的页面不能被换出！！！操作系统是执行的关键代码，需要保证运行的高效性和实时性，如果在操作系统执行过程中，发生了缺页现象，则操作系统不得不等很长时间（硬盘的访问速度比内存的访问速度慢2到3个数量级），这将导致整个系统运行低效。

**当一个Page Table Entry用来描述一般意义上的物理页时，它维护各种权限和映射关系，以及应该有PTE_P标记；但当它用来描述一个被置换出去的物理页时，它被用来维护该物理页与swap磁盘上扇区的映射关系，并且该PTE不应该由MMU将它解释成物理页映射(即没有 PTE_P 标记)**。

与此同时对应的权限则交由mm_struct来维护，当对位于该页的内存地址进行访问的时候，必然导致 page fault，然后ucore能够根据 PTE 描述的swap项将相应的物理页重新建立起来，并根据虚存所描述的权限重新设置好 PTE 使得内存访问能够继续正常进行。

### 虚存中的页与硬盘上的扇区之间的映射关系
一个页被换出到硬盘，则PTE最低位present位应该是0，表示虚实地址映射关系不存在，接下来7位为保留位，表示页帧号的24位地址用来表示在硬盘上的地址。  
\-----------------------------  
| offset     |       reserved       |       0       |  
\-----------------------------  
24 bits  &nbsp;&nbsp; 7 bits &nbsp;&nbsp; 1 bit  

### 执行换入换出的时机
当ucore或应用程序访问地址所在的页不在内存时，就会产生page fault异常，引起调用do_pgfault函数，此函数会判断产生访问异常的地址属于check_mm_struct某个vma表示的合法虚拟地址空间，且保存在硬盘swap文件中。

ucore目前大致有两种策略来实现换出操作，即**积极换出策略**和**消极换出策略**。积极换出策略是指操作系统周期性地（或在系统不忙的时候）主动把某些认为“不常用”的页换出到硬盘上，从而确保系统中总有一定数量的空闲页存在，这样当需要空闲页时，基本上能够及时满足需求；消极换出策略是指，只是当试图得到空闲页时，发现当前没有空闲的物理页可供分配，这时才开始查找“不常用”页面，并把一个或多个这样的页换出到硬盘上。

### 页替换算法的数据结构设计
```
struct Page {  
……   
list_entry_t pra_page_link;   
uintptr_t pra_vaddr;   
};
```
pra_page_link构造了按页的第一次访问时间进行排序的一个链表，这个链表的开始表示第一次访问时间最近的页，链表结尾表示第一次访问时间最远的页。当然链表头可以就可设置为pra_list_head（定义在swap_fifo.c中），构造的时机是在page fault发生后，进行do_pgfault函数时。pra_vaddr可以用来记录此物理页对应的虚拟页起始地址。

当一个物理页（struct Page）需要被swap出去的时候，首先需要确保它已经分配了一个位于磁盘上的swap page（由连续的8个扇区组成）。这里为了简化设计，在swap_check函数中建立了每个虚拟页唯一对应的swap page，其对应关系设定为：`虚拟页对应的PTE的索引值 = swap page的扇区起始位置*8`。

```
struct swap_manager  
{  
    const char *name;  
    /* swap manager 全局初始化 */  
    int (*init) (void);  
    /* 对mm_struct中的数据进行初始化 */  
    int (*init_mm) (struct mm_struct *mm);  
    /* 时钟中断处理 */  
    int (*tick_event) (struct mm_struct *mm);  
    /* Called when map a swappable page into the mm_struct */  
    int (*map_swappable) (struct mm_struct *mm, uintptr_t addr, struct Page *page, int swap_in);   
    /* When a page is marked as shared, this routine is called to delete the addr entry from the swap manager */
    int (*set_unswappable) (struct mm_struct *mm, uintptr_t addr);  
    /* Try to swap out a page, return then victim */  
    int (*swap_out_victim) (struct mm_struct *mm, struct Page *ptr_page, int in_tick);  
    /* check the page relpacement algorithm */  
    int (*check_swap)(void);   
};
```
map_swappable函数用于记录页访问情况相关属性，swap_out_vistim函数用于挑选需要换出的页。显然第二个函数依赖于第一个函数记录的页访问情况。tick_event函数指针也很重要，结合定时产生的中断，可以实现一种积极的换页策略。

1. 准备：为了实现FIFO置换算法，我们应该管理所有可交换的页面，因此我们可以根据时间顺序将这些页面链接到pra_list_head。 使用list.h中的struct list。 struct list是一个简单的双向链表实现，具体函数包括：list_init，list_add（list_add_after），list_add_before，list_del，list_next，list_prev。 将通用列表结构转换为特殊结构（例如结构页面）。可以找到一些宏：le2page（在memlayout.h中），le2vma（在vmm.h中），le2proc（在proc.h中）等；
2. \_fifo_init_mm：初始化pra_list_head并让mm -> sm_priv指向pra_list_head的addr。 现在，从内存控制struct mm_struct，我们可以调用FIFO算法；
3. \_fifo_map_swappable：将最近访问的页放到 pra_list_head 队列最后；
4. \_fifo_swap_out_victim：最早访问的页面从pra_list_head队列中剔除，然后\*ptr_page赋值为这一页。

## 读代码
```
/*
与虚拟地址范围[VPT，VPT + PTSIZE]对应的页面目录条目(page directory entry,PDE)指向页面目录本身。 因此，页面目录被视为页面表和页面目录。
将页面目录视为页表的一个结果是可以通过虚拟地址VPT处的“虚拟页表(virtual page table,VPT)”访问所有PTE。 数字n的PTE存储在vpt[n]中。
第二个结果是当前页面目录的内容将始终在虚拟地址PGADDR（PDX（VPT），PDX（VPT），0）处可用，vpd设置如下。
*/
pte_t * const vpt = (pte_t *)VPT;
pde_t * const vpd = (pde_t *)PGADDR(PDX(VPT), PDX(VPT), 0);
```

## 练习1：给未被映射的地址映射上物理页
完成do_pgfault（mm/vmm.c）函数，给未被映射的地址映射上物理页。设置访问权限的时候 需要参考页面所在VMA的权限，同时需要注意映射物理页时需要操作内存控制结构所指定的页表，而不是内核的页表。

引入虚拟内存后，可能会出现某一些虚拟内存空间是合法的（在vma中），但是还没有为其分配具体的内存页，这样的话，在访问这些虚拟页的时候就会产生pagefault异常，从而使得OS可以在异常处理时完成对这些虚拟页的物理页分配，在中端返回之后就可以正常进行内存的访问了。将出现了异常的线性地址保存在cr2寄存器中；再到trap_dispatch函数，在该函数中会根据中断号，将page fault的处理交给pgfault_handler函数，进一步交给do_pgfault函数进行处理。产生页面异常的原因主要有:

- 目标页面不存在（页表项全为0，即该线性地址与物理地址尚未建立映射或者已经撤销）；
- 相应的物理页面不在内存中（页表项非空，但Present标志位=0，比如在swap分区或磁盘文件上）；
- 访问权限不符合（此时页表项P标志=1，比如企图写只读页面）。
```
 do_pgfault - 处理缺页中断的中断处理例程 interrupt handler to process the page fault execption
 @mm         : the control struct for a set of vma using the same PDT
 @error_code : the error code recorded in trapframe->tf_err which is setted by x86 hardware
 @addr       : the addr which causes a memory access exception, (the contents of the CR2 register)
```
调用栈： trap--> trap_dispatch-->pgfault_handler-->do_pgfault
处理器为ucore的do_pgfault函数提供了两项信息，以帮助诊断异常并从中恢复。
(1) CR2寄存器的内容。 处理器使用产生异常的32位线性地址加载CR2寄存器。 do_pgfault可以使用此地址来查找相应的页面目录和页表条目。
(2) 在内核栈中的错误码。缺页错误码与其他异常的错误码不同，错误码可以通知中断处理例程以下信息:
- P flag(bit 0) 表明异常是否是因为一个不存在的页(0)或违反访问权限或使用保留位(1)；
- W/R flag(bit 1) 表明引起异常的访存操作是读(0)还是写(1)；
- U/S flag (bit 2) 表明引起异常时处理器是在用户态(1)还是内核态(0)

> `do_pgfault(struct mm_struct *mm, uint32_t error_code, uintptr_t addr)`  
第一个是一个mm_struct变量，其中保存了所使用的PDT，合法的虚拟地址空间（使用链表组织），以及与后文的swap机制相关的数据；而第二个参数是产生pagefault的时候硬件产生的error code，可以用于帮助判断发生page fault的原因，而最后一个参数则是出现page fault的线性地址（保存在cr2寄存器中的线性地址）。

1. 查询mm_struct中的虚拟地址链表（线性地址对等映射，因此线性地址等于虚拟地址），确定出现page_fault的线性地址是否合法；
2. 使用error code（包含了这次内存访问为读/写，对应物理页是否存在）判断是否出现权限问题，如果出现问题则直接返回；
3. 根据合法虚拟地址（mm_struct中保存的合法虚拟地址链表中）生成对应产生的物理页的权限；
4. 使用get_pte获取出错的线性地址所对应的虚拟页起始地址对应到的页表项，同时使用页表项保存物理地址（P为1）和被换出的物理页在swap中的位置（P为0），并规定swap中第0个页空出来不用于交换。
```
int do_pgfault(struct mm_struct *mm, uint32_t error_code, uintptr_t addr) {
    int ret = -E_INVAL;

    //根据传进的mm和地址addr，找一个vma，这个vma是在mm的mmap_cache中的，find_vma主要是先找mm中的mmap_cache，如果还不存在，就在mm的mmap_list中找，这个vma用le2vma宏进行转换，直到找到一个地址空间合适的vma，把这个vma赋值给mmap_cache。
    struct vma_struct *vma = find_vma(mm, addr);

    pgfault_num++;
    //检查找到的vma是否为空或符合地址范围
    if (vma == NULL || vma->vm_start > addr) {
        cprintf("not valid addr %x, and  can not find it in vma\n", addr);
        goto failed;
    }
    //如果present位是0，代表没有映射关系，不存在物理页和虚拟页帧的对应关系
    //error_code在cr2寄存器中的后几位，对这个errorcode进行判断，确定读写权限和p位是否为1
    switch (error_code & 3) {
    default:
            /* error code flag : default is 3 ( W/R=1, P=1): write, present */
    case 2: /* error code flag : (W/R=1, P=0): write, not present */
        if (!(vma->vm_flags & VM_WRITE)) {
            cprintf("do_pgfault failed: error code flag = write AND not present, but the addr's vma cannot write
\n");
            goto failed;
        }
        break;
    case 1: /* error code flag : (W/R=0, P=1): read, present */
        cprintf("do_pgfault failed: error code flag = read AND present\n");
        goto failed;
    case 0: /* error code flag : (W/R=0, P=0): read, not present */
        if (!(vma->vm_flags & (VM_READ | VM_EXEC))) {
            cprintf("do_pgfault failed: error code flag = read AND not present, but the addr's vma cannot read or exec\n");
            goto failed;
        }
    }

    /* IF (write an existed addr ) OR
     *    (write an non_existed addr && addr is writable) OR
     *    (read  an non_existed addr && addr is readable)
     * THEN
     *    continue process
     * 写一个存在的地址、写一个不存在的地址但是地址是可写的、读一个不存在的地址但是地址是可读的
     */
    uint32_t perm = PTE_U;
    if (vma->vm_flags & VM_WRITE) {
        perm |= PTE_W;
    }
    // 生成一个权限控制
    addr = ROUNDDOWN(addr, PGSIZE);
    // 向下舍入到n的最接近的倍数

    ret = -E_NO_MEM;

    pte_t *ptep=NULL;
    /*LAB3 EXERCISE 1: YOUR CODE
    * 本次实验用到的宏和定义：
    *   get_pte : 获得pte，返回pte的线性地址、虚拟地址
    *             if the PT contians this pte didn't exist, alloc a page for PT (notice the 3th parameter '1')
    *   pgdir_alloc_page : 调用alloc_page 和 page_insert 分配一个页大小的内存空间，设置物理地址和线性地址的映射关系
    * DEFINES:
    *   VM_WRITE  : If vma->vm_flags & VM_WRITE == 1/0, then the vma is writable/non writable
    *   PTE_W           0x002                   // page table/directory entry flags bit : Writeable
    *   PTE_U           0x004                   // page table/directory entry flags bit : User can access
    * VARIABLES:
    *   mm->pgdir : the PDT of these vma
    *
    */

    /*LAB3 EXERCISE 1: YOUR CODE*/
    ptep = get_pte(mm->pgdir, addr, 1);
    // 第三个参数create代表是否在查找page_directory的过程中没找到的话要不要创建，在这里要创建
    if (ptep == NULL) {
        cprintf("get_pte return a NULL.\n");
        goto failed;
    }
    //(1) 找到一个pte，如果需要的物理页是没有分配而不是被换出到外存中
    //如果物理地址不存在，则分配一个页面并使用逻辑地址映射物理地址，pgdir_alloc_page一个函数就能分配页和设置映射关系
    if (*ptep == 0) {
        struct Page* page = pgdir_alloc_page(mm->pgdir, addr, perm);
        if(page == NULL) {
            cprintf("pgdir_alloc_page return a NULL.\n");
            goto failed;
        }
    }
    else {
    /*LAB3 EXERCISE 2: YOUR CODE
    * 现在我们认为这个pte是一个swap的，我们应该将数据从disk加载到带有物理地址的页面，并将物理地址映射到逻辑地址，触发交换管理器来记录该页面的访问情况。
    *
    *  MACROs or Functions:
    *    swap_in(mm, addr, &page) : 分配一个内存页，根据PTE中的swap地址找到磁盘页的地址，读进内存页中
    *    page_insert ： 创建页的物理地址和线性地址的映射关系
    *    swap_map_swappable ： 设置这一个页是可交换的
    */
        if(swap_init_ok) {    // 判断是否当前交换机制正确被初始化
            struct Page *page=NULL;
            ret = swap_in(mm, addr, &page);   // 将物理页换入到内存中
            if (ret != 0) {
                cprintf("swap_in failed.\n");
                goto failed;
            }
            page_insert(mm->pgdir, page, addr, perm);
            //(2) According to the mm, addr AND page, setup the map of phy addr <---> logical addr
            // 将物理页与虚拟页建立映射关系
            swap_map_swappable(mm, addr, page, 1);
            //(3) make the page swappable。设置当前的物理页为可交换的
            page->pra_vaddr = addr;
            //同时在物理页中维护其对应到的虚拟页的信息；
            //网上有人说这个语句最好应当放置在page_insert函数中，
            //在该建立映射关系的函数外对物理page对应的虚拟地址进行维护显得有些不太合适（感觉好有道理）
        }
        else {
            cprintf("no swap_init_ok but ptep is %x, failed\n",*ptep);
            goto failed;
        }
   }

   ret = 0;
failed:
    return ret;
}
```


问题：
- 请描述页目录项（Page Director Entry）和页表（Page Table Entry）中组成部分对ucore实现页替换算法的潜在用处。

首先不妨先分析PDE以及PTE中各个组成部分以及其含义；

接下来先描述页目录项的每个组成部分，PDE（页目录项）的具体组成如下图所示；描述每一个组成部分的含义如下：
- 前20位表示4K对齐的该PDE对应的页表起始位置（物理地址，该物理地址的高20位即PDE中的高20位，低12位为0）；
- 第9-11位未被CPU使用，可保留给OS使用；
- 接下来的第8位可忽略；
- 第7位用于设置Page大小，0表示4KB；
- 第6位恒为0；
- 第5位用于表示该页是否被使用过；
- 第4位设置为1则表示不对该页进行缓存；
- 第3位设置是否使用write through缓存写策略；
- 第2位表示该页的访问需要的特权级；
- 第1位表示是否允许读写；
- 第0位为该PDE的存在位；

接下来描述页表项（PTE）中的每个组成部分的含义，具体组成如下图所示：
- 高20位与PDE相似的，用于表示该PTE指向的物理页的物理地址；
- 9-11位保留给OS使用；
- 7-8位恒为0；
- 第6位表示该页是否为dirty，即是否需要在swap out的时候写回外存；
- 第5位表示是否被访问；
- 3-4位恒为0；
- 0-2位分别表示存在位、是否允许读写、访问该页需要的特权级；

可以发现无论是PTE还是TDE，都具有着一些保留的位供操作系统使用，也就是说ucore可以利用这些位来完成一些其他的内存管理相关的算法，比如可以在这些位里保存最近一段时间内该页的被访问的次数（仅能表示0-7次），用于辅助近似地实现虚拟内存管理中的换出策略的LRU之类的算法；也就是说这些保留位有利于OS进行功能的拓展；
> 作者：AmadeusChan
> 链接：https://www.jianshu.com/p/8d6ce61ac678
> 来源：简书

如果ucore的缺页服务例程在执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？

考虑到ucore的缺页服务例程如果在访问内容中出现了缺页异常，则会有可能导致ucore最终无法完成缺页的处理，因此一般不应该将缺页的ISR以及OS中的其他一些关键代码或者数据换出到外存中，以确保操作系统的正常运行；如果缺页ISR在执行过程中遇到页访问异常，则最终硬件需要完成的处理与正常出现页访问异常的处理相一致，均为：

- 将发生错误的线性地址保存在cr2寄存器中;
- 在中断栈中依次压入EFLAGS，CS, EIP，以及页访问异常码errorcode，由于ISR一定是运行在内核态下的，因此不需要压入ss和esp以及进行栈的切换；
- 根据中断描述符表查询到对应页访问异常的ISR，跳转到对应的ISR处执行，接下来将由软件进行处理；


## 练习2：补充完成基于FIFO的页面替换算法
维基百科：最简单的页面替换算法（Page Replace Algorithm）是FIFO算法。先进先出页面替换算法是一种低开销算法。这个想法从名称中可以明显看出 - 操作系统跟踪队列中内存中的所有页面，最近到达的放在后面，最早到达的放在前面。当需要更换页面时，会选择队列最前面的页面（最旧的页面）。虽然FIFO开销小且直观，但在实际应用中表现不佳。因此，它很少以未修改的形式使用。该算法存在Belady异常。

 FIFO的详细信息
1. 准备：为了实现FIFO，我们应该管理所有可交换的页面，这样我们就可以按照时间顺序将这些页面链接到pra_list_head。将通用list换为特殊结构（例如Page）；
2. \_fifo_init_mm：初始化pra_list_head并让mm-> sm_priv指向pra_list_head的addr。 现在，从内存控制struct mm_struct，我们可以访问FIFO；
3. \_fifo_map_swappable: 最近到达的页需要放到pra_list_head队列的最末尾；
4. \_fifo_swap_out_victim: 最早到达的页面在pra_list_head队列最前边，我们应该将它踢出去。

```
将当前的物理页面插入到FIFO算法中维护的可被交换出去的物理页面链表中的末尾，从而保证该链表中越接近链表头的物理页面在内存中的驻留时间越长；
static int _fifo_map_swappable(struct mm_struct *mm, uintptr_t addr, struct Page *page, int swap_in)
{
    list_entry_t *head=(list_entry_t*) mm->sm_priv;
    // 找到链表入口
    list_entry_t *entry=&(page->pra_page_link); 
    // 找到当前物理页用于组织成链表的list_entry_t

    assert(entry != NULL && head != NULL);
    /*LAB3 EXERCISE 2: YOUR CODE*/
    // link the most recent arrival page at the back of the pra_list_head qeueue
    // 将当前指定的物理页插入到链表的末尾
    list_add(head, entry);
    return 0;
}
```
```
static int
_fifo_swap_out_victim(struct mm_struct *mm, struct Page ** ptr_page, int in_tick)
{
     list_entry_t *head=(list_entry_t*) mm->sm_priv; 
     // 找到链表的入口
     assert(head != NULL);
     assert(in_tick==0);
     /* Select the victim */
     /*LAB3 EXERCISE 2: YOUR CODE*/
     // unlink the earliest arrival page in front of pra_list_head qeueue
     //list_entry_t *le = head->prev; the given answer
     list_entry_t *le = list_next(head);
     // 取出链表头，即最早进入的物理页面
     assert(le != NULL);
     // 确保链表非空
     struct Page *p = le2page(le,pra_page_link);
     // 找到对应的物理页面的Page结构
     list_del(le);
     // 从链表上删除取出的即将被换出的物理页面
     assert(p != NULL);
     *ptr_page = p;
     // assign the value of *ptr_page to the addr of this page
     return 0;
}
```
如果在_fifo_map_swappable函数中使用的是list_add_before的话，在_fifo_swap_out_victim中应该使用list_next(head)取得要被删除的页；如果在_fifo_map_swappable函数中使用的是list_add的话，在_fifo_swap_out_victim中应该使用head->prev取得要被删除的页；这个链表是双向循环链表！

如果要在ucore上实现"extended clock页替换算法"请给你的设计方案，现有的swap_manager框架是否足以支持在ucore中实现此算法？如果是，请给你的设计方案。如果不是，请给出你的新的扩展和基此扩展的设计方案。并需要回答如下问题

在现有框架基础上可以支持Extended clock算法。

根据上文中提及到的PTE的组成部分可知，PTE中包含了dirty位和访问位，因此可以确定某一个虚拟页是否被访问过以及写过，但是，考虑到在替换算法的时候是将物理页面进行换出，而可能存在着多个虚拟页面映射到同一个物理页面这种情况，也就是说某一个物理页面是否dirty和是否被访问过是有这些所有的虚拟页面共同决定的，而在原先的实验框架中，物理页的描述信息Page结构中默认只包括了一个对应的虚拟页的地址，应当采用链表的方式，在Page中扩充一个成员，把物理页对应的所有虚拟页都给保存下来；而物理页的dirty位和访问位均为只需要某一个对应的虚拟页对应位被置成1即可置成1；
完成了上述对物理页描述信息的拓展之后，考虑对FIFO算法的框架进行修改得到拓展时钟算法的框架，由于这两种算法都是将所有可以换出的物理页面均按照进入内存的顺序连成一个环形链表，因此初始化，将某个页面置为可以/不可以换出这些函数均不需要进行大的修改(小的修改包括在初始化当前指针等)，唯一需要进行重写的函数是选择换出物理页的函数swap_out_victim，对该函数的修改如下：

从当前指针开始，对环形链表进行扫描，根据指针指向的物理页的状态（表示为(access, dirty)）来确定应当进行何种修改：如果状态是(0, 0)，则将该物理页面从链表上去下，该物理页面记为换出页面，但是由于这个时候这个页面不是dirty的，因此事实上不需要将其写入swap分区；
如果状态是(0,1)，则将该物理页对应的虚拟页的PTE中的dirty位都改成0，并且将该物理页写入到外存中，然后指针跳转到下一个物理页；如果状态是(1, 0), 将该物理页对应的虚拟页的PTE中的访问位都置成0，然后指针跳转到下一个物理页面；如果状态是(1, 1)，则该物理页的所有对应虚拟页的PTE中的访问为置成0，然后指针跳转到下一个物理页面；


需要被换出的页的特征是什么？

该物理页在当前指针上一次扫过之前没有被访问过；
该物理页的内容与其在外存中保存的数据是一致的, 即没有被修改过;

在ucore中如何判断具有这样特征的页？

在ucore中判断具有这种特征的页的方式已经在上文设计方案中提及过了，具体为：

假如某物理页对应的所有虚拟页中存在一个dirty的页，则认为这个物理页为dirty，否则不这么认为；
假如某物理页对应的所有虚拟页中存在一个被访问过的页，则认为这个物理页为被访问过的，否则不这么认为；

何时进行换入和换出操作？

在产生page fault的时候进行换入操作；
换出操作源于在算法中将物理页的dirty从1修改成0的时候，因此这个时候如果不进行写出到外存，就会造成数据的不一致，具体写出内存的时机是比较细节的问题, 可以在修改dirty的时候写入外存，或者是在这个物理页面上打一个需要写出的标记，到了最终删除这个物理页面的时候，如果发现了这个写出的标记，则在这个时候再写入外存；后者使用一个写延迟标记，有利于多个写操作的合并，从而降低缺页的代价；

