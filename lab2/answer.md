# 读代码
在bootloader进入保护模式前进行探测物理内存分布和大小，基本方式是通过BIOS中断调用，在实模式下完成，在boot/bootasm.S中从probe_memory处到finish_probe处的代码部分完成。以下应该是检测到的物理内存信息：
```
memory management: default_pmm_manager
e820map:
  memory: 0009fc00, [00000000, 0009fbff], type = 1.
  memory: 00000400, [0009fc00, 0009ffff], type = 2.
  memory: 00010000, [000f0000, 000fffff], type = 2.
  memory: 07ee0000, [00100000, 07fdffff], type = 1.
  memory: 00020000, [07fe0000, 07ffffff], type = 2.
  memory: 00040000, [fffc0000, ffffffff], type = 2.
```
参考：type是物理内存空间的类型，1是可以使用的，2是暂时不能够使用的。
```
之前是开启A20的16位地址线，实现20位地址访问。通过写键盘控制器8042的64h端口与60h端口。先转成实模式！
获取的物理内存信息是用这种结构存的（内存映射地址描述符），一共20字节：
struct e820map {
    int nr_map;
    struct {
        uint64_t addr;    //8字节，unsigned long long，基地址？
        uint64_t size;    //8字节，unsigned long long，大小
        uint32_t type;    //4字节，unsigned long，内存类型
    } __attribute__((packed)) map[E820MAX];
};
每探测到一块内存空间，对应的内存映射描述符被写入指定表，
以下是通过向INT 15h中断传入e820h参数来探测物理内存空间的信息
"$"美元符号修饰立即数，"%"修饰寄存器。
probe_memory:
    movl $0, 0x8000     #把0这个立即数写入0x8000地址，
    xorl %ebx, %ebx     #相当于我们设置在0x8000处存放struct e820map, 并清除e820map中的nr_map置0
    movw $0x8004, %di   #0x8004正好就是第一个内存映射地址描述符的地址，因为nr_map是四个字节
start_probe:
    movl $0xE820, %eax  #传入0xE820作为参数，
    movl $20, %ecx      #内存映射地址描述符的大小是20个字节
    movl $SMAP, %edx    #SMAP之前定义是0x534d4150，不知道何用
    int $0x15           #调用INT 15H中断
    jnc cont            #CF=0,则跳转到cont
    movw $12345, 0x8000 
    jmp finish_probe
cont:
    addw $20, %di       #设置下一个内存映射地址描述符的地址
    incl 0x8000         #E820map中的nr_map加一
    cmpl $0, %ebx       #如果INT0x15返回的ebx为零，表示探测结束，如果还有就继续找
    jnz start_probe
finish_probe:
```
Below are from CSDN
> 调用中断int 15h 之前，需要填充如下寄存器：  
- eax  int 15h 可以完成许多工作，主要有ax的值决定，我们想要获取内存信息，需要将ax赋值为0E820H。  
- ebx  放置着“后续值(continuation value)”，第一次调用时ebx必须为0.  
- es:di 指向一个地址范围描述结构 ARDS(Address Range Descriptor Structure), BIOS将会填充此结构。
- ecx  es:di所指向的地址范围描述结构的大小，以字节为单位。无论es:di所指向的结构如何设置，BIOS最多将会填充ecx字节。不过，通常情况下无论ecx为多大，BIOS只填充20字节，有些BIOS忽略ecx的值，总是填充20字节。  
- edx  0534D4150h('SMAP')——BIOS将会使用此标志，对调用者将要请求的系统映像信息进行校验，这些信息被BIOS放置到es:di所指向的结构中。  

> 中断调用之后，结果存放于下列寄存器之中。  
- CF  CF=0表示没有错误，否则存在错误。  
- eax   0534D4150h('SMAP')  
- es:di  返回的地址范围描述符结构指针，和输入值相同。  
- ecx BIOS填充在地址范围描述符中的字节数量，被BIOS所返回的最小值是20字节。  
- ebx  这里放置着为等到下一个地址描述符所需要的后续值，这个值得实际形势依赖于具体的BIOS的实现，调用者不必关心它的具体形式，自需在下一次迭代时将其原封不动地放置到ebx中，就可以通过它获取下一个地址范围描述符。如果它的值为0，并且CF没有进位，表示它是最后一个地址范围描述符。  

由于一个物理页需要占用一个Page结构的空间，Page结构在设计时须尽可能小，以减少对内存的占用。
```
struct Page {                       // 描述了一个Page
    int ref;                        // 这一页被页表的引用计数，一个页表项设置了一个虚拟页的映射
    uint32_t flags;                 // 描述这个Page的状态，可能每个位表示不同的意思
    unsigned int property;          // property表示这个块中空闲页的数量，用到此成员变量的这个Page比较特殊，
                                    // 是这个连续内存空闲块地址最小的一页（即头一页， Head Page）。
    list_entry_t page_link;         // 链接比它地址小和大的其他连续内存空闲块。
};
```
flag用到了两个bit
```
#define PG_reserved       0       // 表明了是否被保留，如果被保留，则bit 0会设置位1，且不能放到空闲列表里
#define PG_property       1       // bit 1表示此页是否是free的，如果设置为1，表示这页是free的，可以被分配；
                                  // 如果设置为0，表示这页已经被分配出去了，不能被再二次分配。
```
总结来说：
一个页，里边有各种属性和双向链表的指针段
- 1、ref表示这个页被页表的引用记数，是映射此物理页的虚拟页个数。一旦某页表中有一个页表项设置了虚拟页到这个Page管理的物理页的映射关系，就会把Page的ref加一。反之，若是解除，那就减一。
- 2、 flags表示此物理页的状态标记，有两个标志位，第一个表示是否被保留，如果被保留了则设为1（比如内核代码占用的空间）。第二个表示此页是否是free的。如果设置为1，表示这页是free的，可以被分配；如果设置为0，表示这页已经被分配出去了，不能被再二次分配。
- 3、property用来记录某连续内存空闲块的大小，这里需要注意的是用到此成员变量的这个Page一定是连续内存块的开始地址（第一页的地址）。
- 4、page_link是便于把多个连续内存空闲块链接在一起的双向链表指针，连续内存空闲块利用第一个页的成员变量page_link来链接比它地址小和大的其他连续内存空闲块，用到这个成员变量的是这个块的地址最小的一页。
下面简单看看mm/pmm.c中的pmm_init()
```
/* pmm_init - initialize the physical memory management */
static void
page_init(void) {
    struct e820map *memmap = (struct e820map *)(0x8000 + KERNBASE);
    uint64_t maxpa = 0;

    cprintf("e820map:\n");
    int i;
    for (i = 0; i < memmap->nr_map; i ++) {
        uint64_t begin = memmap->map[i].addr, end = begin + memmap->map[i].size;
        cprintf("  memory: %08llx, [%08llx, %08llx], type = %d.\n",
                memmap->map[i].size, begin, end - 1, memmap->map[i].type);
        if (memmap->map[i].type == E820_ARM) {
            if (maxpa < end && begin < KMEMSIZE) {
                maxpa = end;
            }
        }
    }
    if (maxpa > KMEMSIZE) {
        maxpa = KMEMSIZE;
    }

    extern char end[];

    npage = maxpa / PGSIZE;
    //起始物理内存地址位0，所以需要管理的页个数为npage，需要管理的所有页的大小位sizeof(struct Page)*npage
    pages = (struct Page *)ROUNDUP((void *)end, PGSIZE);
    // pages的地址，最末尾地址按照页大小取整。
    for (i = 0; i < npage; i ++) {
        SetPageReserved(pages + i);
    }
    //当前的这些页设置为已占用的

    uintptr_t freemem = PADDR((uintptr_t)pages + sizeof(struct Page) * npage);
    // 之前设置了占用的页，那空闲的页就是从（pages+sizeof(struct Page)*npage）以上开始的

    for (i = 0; i < memmap->nr_map; i ++) {
        uint64_t begin = memmap->map[i].addr, end = begin + memmap->map[i].size;
        if (memmap->map[i].type == E820_ARM) {
            if (begin < freemem) {
                begin = freemem;
            }
            if (end > KMEMSIZE) {
                end = KMEMSIZE;
            }
            if (begin < end) {
                begin = ROUNDUP(begin, PGSIZE);
                end = ROUNDDOWN(end, PGSIZE);
                if (begin < end) {
                    init_memmap(pa2page(begin), (end - begin) / PGSIZE);
                    // 通过调用本函数进行空闲的标记
                }   
            }
        }
    }
}
```
SetPageReserved表示把物理地址对应的Page结构中的flags标志设置为PG_reserved ，表示这些页已经被使用了，将来不能被用于分配。而init_memmap函数把空闲物理页对应的Page结构中的flags和引用计数ref清零，并加到free_area.free_list指向的双向列表中。
```
struct pmm_manager {
            const char *name;                                 //物理内存页管理器的名字
            void (*init)(void);                               //初始化内存管理器
            void (*init_memmap)(struct Page *base, size_t n); //初始化管理空闲内存页的数据结构
            struct Page *(*alloc_pages)(size_t n);            //分配n个物理内存页
            void (*free_pages)(struct Page *base, size_t n);  //释放n个物理内存页
            size_t (*nr_free_pages)(void);                    //返回当前剩余的空闲页数
            void (*check)(void);                              //用于检测分配/释放实现是否正确
};
```

```
free_area_t - 维护一个双向链表记录没有用到的Page。
typedef struct {
    list_entry_t free_list;         // 整个双向链表的头节点
    unsigned int nr_free;           // 表示空闲页的数量
} free_area_t;
```
```
typedef struct list_entry list_entry_t;
struct list_entry {
    struct list_entry *prev, *next;
};
类似Linux里的双向链表，这只是指针部分，数据部分在其他定义里
```


# 练习1 实现first-fit连续物理内存分配算法
重写函数: default_init, default_init_memmap,default_alloc_pages, default_free_pages。  
在实现first_fit的回收函数时，注意连续地址空间之间的合并操作。在遍历空闲页块链表时，需要按照空闲块起始地址来排序，形成一个有序的的链表。

首次适应算法（First Fit）：该算法从空闲分区链首开始查找，直至找到一个能满足其大小要求的空闲分区为止。然后再按照需求的大小，从该分区中划出一块内存分配给请求者，余下的空闲分区仍留在空闲分区链中。多使用内存中低地址部分的空闲区，在高地址部分的空闲区很少被利用，从而保留了高地址部分的空闲区。显然为以后到达的大作业分配大的内存空间创造了条件。但是低地址部分不断被划分，留下许多难以利用、很小的空闲区，每次查找又都从低地址部分开始，会增加查找的开销。  

在First Fit算法中，分配器维护一个空闲块列表（free表）。一旦收到内存分配请求，
它遍历列表找到第一个满足的块。如果所选块明显大于请求的块，则分开，其余的空间将被添加到列表中下一个free块中。  
>1. 准备：实现First Fit我们需要使用链表管理空闲块，free_area_t被用来管理free块，首先，找到list.h中的"struct list"。结构"list"是一个简单的双向链表实现。使用"list_init"，"list_add"（"list_add_after"和"list_add_before"），"list_del"，
 "list_next"，"list_prev"。有一个棘手的方法是将一般的"list"结构转换为一个特殊结构（如struct"page"），使用以下宏："le2page"（在memlayout.h中）。
>2. "default_init"：重用例子中的"default_init"函数来初始化"free_list"并将"nr_free"设置为0。"free_list"用于记录空闲内存块，"nr_free"是可用内存块的总数。
>3. "default_init_memmap"：调用栈为"kern_init" -> "pmm_init" -> "page_init" -> "init_memmap" -> "pmm_manager" -> "init_memmap"。此函数用于初始化空闲块（使用参数"addr_base"，"page_mumber"）。为了初始化一个空闲块，首先，应该在这个空闲块中初始化每个页面（在memlayout.h中定义）。这个
 程序包括：
- 设置"p -> flags"的'PG_property'位，表示该页面为有效。在函数"pmm_init"（在pmm.c中），"p-> flags"的位'PG_reserved"已经设置好了。
- 如果此页面是free的且不是free区块的第一页，"p-> property"应该设置为0。
- 如果此页面是free的且是free区块的第一页，"p-> property"应该设置为本空闲块的总页数。
>4. "default_alloc_pages"：在空闲列表中搜索第一个空闲块（块大小>=n），返回该块的地址作为所需的地址.

空闲页管理链表的初始化：
```
static void default_init(void) {
    list_init(&free_list);
    nr_free = 0;
}
static inline void list_init(list_entry_t *elm) {
    elm->prev = elm->next = elm;
}
把free_list的双向链表中的指针都指向自己，且计数器为0
```
```
初始化空闲页链表，初始化每一个空闲页，然后计算空闲页的总数。
static void default_init_memmap(struct Page *base, size_t n) {   
    assert(n > 0);
    struct Page *p = base;
    for (; p != base + n; p ++) {
        assert(PageReserved(p));
        //这个页是否为保留页,PageReserved(p)返回true才会继续，如果返回true了，说明是保留页
        //设置标志位
        p->flags = 0；
        SetPageProperty(p);
        p->property = 0;    //应该只有第一个页的这个参数有用
        set_page_ref(p, 0);//清空引用，现在是没有虚拟内存引用它的
        list_add_before(&free_list, &(p->page_link));//插入空闲页的链表里面
    }
    nr_free += n;  //连续有n个空闲块，空闲链表的个数加n
    base->property=n; //连续内存空闲块的大小为n，属于物理页管理链表
    //所有的页都在这个双向链表里且只有第0个页有这个块的信息
}
```
default_alloc_pages从空闲页链表中查找n个空闲页，如果成功，返回第一个页表的地址。遍历空闲链表，一旦发现有大于等于n的连续空闲页块，便将这n个页从空闲页链表中取出，同时使用SetPageReserved和ClearPageProperty表示该页为使用状态，同时如果该连续页的数目大于n，则从第n+1开始截断，之后为截断的块，重新计算相应的property的值。在贴代码之前先说说几个宏。
```
/* 将这个le转换成一个Page */
#define le2page(le, member)                 \
    to_struct((le), struct Page, member)
    
/* *
 * to_struct - get the struct from a ptr
 * @ptr:    a struct pointer of member
 * @type:   the type of the struct this is embedded in
 * @member: the name of the member within the struct
 * 一般用的时候传进来的type是Page类型的，ptr是这个（Page+双向链表的两个指针）块的双向链表指针的开始地址。offsetof算出了page_link在Page中的偏移值，ptr减去双向链表第一个指针的偏移量得到了这个Page的地址
 */
#define to_struct(ptr, type, member)                               \
    ((type *)((char *)(ptr) - offsetof(type, member)))

/* Return the offset of 'member' relative to the beginning of a struct type */
0不代表具体地址，这个offsetof代表这个member在这个type中的偏移值
#define offsetof(type, member)                                      \
    ((size_t)(&((type *)0)->member))
```
```
static struct Page * default_alloc_pages(size_t n) {
    assert(n > 0);
    if (n > nr_free) {
        return NULL;
    }
    // n 一定要大于0，且n要小于当前可用的空闲块数
    list_entry_t *le, *len;
    le = &free_list;
    struct Page *p=NULL;
    while((le=list_next(le)) != &free_list) {
        p = le2page(le, page_link);
        if(p->property>=n)
            break;
    }
    //在free_list里遍历每一页，用le2page转换成Page
    //如果找到了一个property大于n的就说明找到了这个符合要求的块
    if(p != NULL){
        int i;
        for(i=0;i<n;i++){
            len = list_next(le);
            struct Page *pp = le2page(le, page_link);
            SetPageReserved(pp);
            ClearPageProperty(pp);
            list_del(le);
            le = len;
        }
        // 如果我现在找到的块是大于n的，那就拆开
        if(p->property>n){
            (le2page(le,page_link))->property = p->property - n;
        }
        ClearPageProperty(p);
        SetPageReserved(p);
        nr_free -= n;
        return p;
    }
    return NULL;
}
```
default_free_pages将base为起始地址的n个页面放回到free_list中
```
static void default_free_pages(struct Page *base, size_t n) {
    assert(n > 0);
    list_entry_t *le = &free_list;
    struct Page *p = base;
    //找到比base大的页面地址
    while((le=list_next(le)) != &free_list){
        p = le2page(le,page_link);
        if(p > base)
            break;
    }
    //在找到的p之前逐个插入
    for(p = base; p < base + n; p ++){
       list_add_before(le,&(p->page_link));
    }
    base->flags=0;
    set_page_ref(base,0);
    ClearPageProperty(base);
    SetPageProperty(base);
    base->property = n;
    // 清空flag的信息，清空引用的信息，清空property信息，设置这个Page又是可以被引用的了
    // 当前的base又是n个空闲块的头
    p = le2page(le,page_link);
    if(base+n==p){
        base->property+=p->property;
        p->property=0;
    }
    //看是不是可以跟后边的块恰好连在一起，如果连在一起的话就可以合并了
    le=list_prev(&(base->page_link));
    p = le2page(le, page_link);
    //看是不是可以跟前边的连在一起，如果可以的话这个base就可以把property设成0了
    if(le!=&free_list && p==base-1){
        while(le!=&free_list){
            if(p->property){
                p->property+=base->property;
                base->property=0;
                break;
            }
            le = list_prev(le);
            p=le2page(le,page_link);
        }
    }
    nr_free +=n;
    cprintf("release %d page,last %d.\n",n,nr_free);
}
```
运行中出现提示，表明本题成功：
```
release 1 page,last 1.
release 1 page,last 2.
release 1 page,last 3.
release 1 page,last 1.
release 1 page,last 32291.
release 1 page,last 32292.
release 1 page,last 32293.
release 3 page,last 3.
release 1 page,last 1.
release 3 page,last 4.
release 1 page,last 4.
release 2 page,last 4.
release 1 page,last 5.
release 5 page,last 32293.
check_alloc_page() succeeded!
```
first_fit有一种改进，next_fit，第一次找到之后不暂停，第二次找到之后才真正给分配空间。修改比较简单，第一次找到之后记一个flag，下次再找到就可以分配了。
# 练习二 
## 系统执行中的地址映射。  
mooc中讲到了在段页式管理机制下运行这整个过程中，虚拟地址到物理地址的映射产生了多次变化，实现了最终的段页式映射关系：
> virt addr = linear addr = phy addr + 0xC0000000  
第一个阶段（**开启保护模式，创建启动段表**）是bootloader阶段，即从bootloader的start函数（在boot/bootasm.S中）到执行ucore kernel的kern_entry函数之前，其虚拟地址、线性地址以及物理地址之间的映射关系与lab1的一样，即：
> virt addr = linear addr = phy addr  
第二个阶段（**创建初始页目录表，开启分页模式**）从kern_entry函数开始，到pmm_init函数被执行之前。通过几条汇编指令（在kern/init/entry.S中）使能分页机制，主要做了两件事：
```
通过movl %eax, %cr3指令把页目录表的起始地址存入CR3寄存器中；
通过movl %eax, %cr0指令把cr0中的CR0_PG标志位设置上。
```
在此之后，进入了分页机制，地址映射关系如下：
```
virt addr = linear addr = phy addr # 线性地址在0~4MB之内三者的映射关系
virt addr = linear addr = phy addr + 0xC0000000 # 线性地址在0xC0000000~0xC0000000+4MB之内三者的映射关系
```
仅仅比第一个阶段增加了下面一行的0xC0000000偏移的映射，并且作用范围缩小到了0~4M。在下一个节点，会将作用范围继续扩充到0~KMEMSIZE。  
此时的内核（EIP）还在0~4M的低虚拟地址区域运行，而在之后，这个区域的虚拟内存是要给用户程序使用的。为此，需要使用一个绝对跳转来使内核跳转到高虚拟地址（代码在kern/init/entry.S中）：
```
    # update eip
    # now, eip = 0x1.....
    leal next, %eax
    # set eip = KERNBASE + 0x1.....
    jmp *%eax
next:
```
跳转完毕后，通过把boot_pgdir[0]对应的第一个页目录表项（0~4MB）清零来取消了临时的页映射关系：
```
    # unmap va 0 ~ 4M, it's temporary mapping
    xorl %eax, %eax
    movl %eax, __boot_pgdir
```
最终的地址映射关系如下：
```
 lab2 stage 2: virt addr = linear addr = phy addr + 0xC0000000 # 线性地址在0~4MB之内三者的映射关系
```
第三个阶段（**完善段表和页表**）从pmm_init函数被调用开始。pmm_init函数将页目录表项补充完成（从0~4M扩充到0~KMEMSIZE）。然后，更新了段映射机制，使用了一个新的段表。这个新段表除了包括内核态的代码段和数据段描述符，还包括用户态的代码段和数据段描述符以及TSS（段）的描述符。理论上可以在第一个阶段，即bootloader阶段就将段表设置完全，然后在此阶段继续使用，但这会导致内核的代码和bootloader的代码产生过多的耦合，于是就有了目前的设计。
这时形成了我们期望的虚拟地址、线性地址以及物理地址之间的映射关系：
```
 lab2 stage 3: virt addr = linear addr = phy addr + 0xC0000000
```

```
请描述页目录项（Pag Director Entry）和页表（Page Table Entry）中每个组成部分的含义和以及对ucore而言的潜在用处。

页目录项（Pag Director Entry）每一位的含义：
前20位表示4K对齐的该PDE对应的页表起始位置（物理地址，该物理地址的高20位即PDE中的高20位，低12位为0）；
第9-11位未被CPU使用，可保留给OS使用；
接下来的第8位可忽略；
第7位用于设置Page大小，0表示4KB；
第6位恒为0；
第5位用于表示该页是否被使用过；
第4位设置为1则表示不对该页进行缓存；
第3位设置是否使用write through缓存写策略；
第2位表示该页的访问需要的特权级；
第1位表示是否允许读写；
第0位为该PDE的存在位；

页表项（PTE）中的每项的含义：
高20位与PDE相似的，用于表示该PTE指向的物理页的物理地址；
9-11位保留给OS使用；
7-8位恒为0；
第6位表示该页是否为dirty，即是否需要在swap out的时候写回外存；
第5位表示是否被访问；
3-4位恒为0；
0-2位分别表示存在位、是否允许读写、访问该页需要的特权级；

PTE和PDE都有一些保留位供操作系统使用，ucore利用保留位来完成一些其他的内存管理相关的算法。

当ucore执行过程中出现了页访问异常，硬件需要完成的事情分别如下：
- 将发生错误的线性地址保存在cr2寄存器中;
- 在中断栈中依次压入EFLAGS，CS, EIP，以及页访问异常码error code，如果pgfault是发生在用户态，则还需要先压入ss和esp，并且切换到内核栈；
- 根据中断描述符表查询到对应page fault的处理例程地址如后，跳转到对应处执行。
```


## 建立虚拟页和物理页帧的地址映射关系
整个页目录表和页表所占空间大小取决与二级页表要管理和映射的物理页数。  
假定当前物理内存0\~16MB，每物理页（也称Page Frame）大小为4KB，则有4096个物理页，也就意味这有4个页目录项和4096个页表项需要设置。一个页目录项（Page Directory Entry，PDE）和一个页表项（Page Table Entry，PTE）占4B。即使是4个页目录项也需要一个完整的页目录表（占4KB）。而4096个页表项需要16KB（即4096\*4B）的空间，也就是4个物理页，16KB的空间。所以对16MB物理页建立一一映射的16MB虚拟页，需要4+1=5个物理页，即20KB的空间来形成二级页表。  

把0~KERNSIZE（明确ucore设定实际物理内存**不能超过KERNSIZE值，即0x38000000字节，896MB，3670016个物理页**）的物理地址一一映射到页目录项和页表项的内容，其大致流程如下：
1. 指向页目录表的指针已存储在boot_pgdir变量中。
2. 映射0~4MB的首个页表已经填充好。
3. 调用boot_map_segment函数进一步建立一一映射关系，具体处理过程以页为单位进行设置，即:
> linear addr = phy addr + 0xC0000000

设一个32bit线性地址la有一个对应的32bit物理地址pa，如果在以la的高10位为索引值的页目录项中的存在位（PTE_P）为0，表示缺少对应的页表空间，则可通过alloc_page获得一个空闲物理页给页表，页表起始物理地址是按4096字节对齐的，这样填写页目录项的内容为：
> 页目录项内容 = (页表起始物理地址 & ~0x0FFF) | PTE_U | PTE_W | PTE_P
      

进一步对于页表中以线性地址la的中10位为索引值对应页表项的内容为：
> 页表项内容 = (pa & ~0x0FFF) | PTE_P | PTE_W

其中：
> PTE_U：位3，表示用户态的软件可以读取对应地址的物理内存页内容  
> PTE_W：位2，表示物理内存页内容可写  
> PTE_P：位1，表示物理内存页存在  

ucore的内存管理经常需要查找页表：  
给定一个虚拟地址，找出这个虚拟地址在二级页表中对应的项。通过更改此项的值可以方便地将虚拟地址映射到另外的页上。可完成此功能的这个函数是get_pte函数。它的原型为
```
pte_t *get_pte(pde_t *pgdir, uintptr_t la, bool create)
```
这里涉及到三个类型**pte_t**、**pde_t**和**uintptr_t**。这三个都是unsigned int类型。  
pde_t：page directory entry，一级页表的表项。  
pte_t：page table entry，表示二级页表的表项。  
uintptr_t：表示为线性地址，由于段式管理只做直接映射，所以它也是逻辑地址。  
pgdir：给出页表起始地址。通过查找这个页表，我们需要给出二级页表中对应项的地址。  
可以在需要时再添加对应的二级页表。如果在查找二级页表项时，发现对应的二级页表不存在，则需要**根据create参数的值来处理是否创建新的二级页表**。如果create参数为0，则get_pte返回NULL；如果create参数不为0，则get_pte需要申请一个新的物理页（通过alloc_page来实现，可在mm/pmm.h中找到它的定义），再在一级页表中添加页目录项指向表示二级页表的新物理页。  
> 注意，新申请的页必须全部设定为零，因为这个页所代表的虚拟地址都没有被映射。  

当建立从一级页表到二级页表的映射时，需要注意设置控制位。这里应该设置同时设置上PTE_U、PTE_W和PTE_P（定义可在mm/mmu.h）。如果原来就有二级页表，或者新建立了页表，则只需返回对应项的地址即可。  
虚拟地址只有映射上了物理页才可以正常的读写。在完成映射物理页的过程中，除了要在页表的对应表项上填上相应的物理地址外，还要设置正确的控制位。  

只有当一级二级页表的项都设置了用户写权限后，用户才能对对应的物理地址进行读写。由于一个物理页可能被映射到不同的虚拟地址上去（譬如一块内存在不同进程间共享），**当这个页需要在一个地址上解除映射时，操作系统不能直接把这个页回收，而是要先看看它还有没有映射到别的虚拟地址上**。这是通过查找管理该物理页的Page数据结构的成员变量ref（用来表示虚拟页到物理页的映射关系的个数）来实现的，如果ref为0了，表示没有虚拟页到物理页的映射关系了，就可以把这个物理页给回收了，从而这个物理页是free的了，可以再被分配。  

page_insert函数将物理页映射在了页表上。可参看page_insert函数的实现来了解ucore内核是如何维护这个变量的。当不需要再访问这块虚拟地址时，可以把这块物理页回收并在将来用在其他地方。取消映射由page_remove来做，这其实是page_insert的逆操作。
建立好一一映射的二级页表结构后，由于分页机制在前一节所述的前两个阶段已经开启，分页机制到此初始化完毕。当执行完毕gdt_init函数后，新的段页式映射已经建立好了。

预备知识copy完了，上练习二和练习三
## 练习二代码
**预备知识不够用了**  
上mmu.h的代码读读
```

A linear address 'la' has a three-part structure as follows:
+--------10------+-------10-------+---------12----------+
| Page Directory |   Page Table   | Offset within Page  |
|      Index     |     Index      |                     |
+----------------+----------------+---------------------+
 \--- PDX(la) --/ \--- PTX(la) --/ \---- PGOFF(la) ----/
 \----------- PPN(la) -----------/
The PDX, PTX, PGOFF, and PPN macros decompose linear addresses as shown.
To construct a linear address la from PDX(la), PTX(la), and PGOFF(la),
use PGADDR(PDX(la), PTX(la), PGOFF(la)).

```

```
//get_pte - get Page Table Entry and return the kernel virtual address of this Page Table Entry for la
//        - if the PT contians this Page Table Entry didn't exist, alloc a page for PT
// parameter:
//  pgdir:  the kernel virtual base address of PDT （页目录表的入口）
//  la:     the linear address need to map         （线性地址）
//  create: a logical value to decide if alloc a page for PT
// return vaule: the kernel virtual address of this pte （返回这个页表项的虚拟地址）
pte_t *
get_pte(pde_t *pgdir, uintptr_t la, bool create) {
/*   *   使用KADDR()获得物理地址
     *   PDX(la) = 虚拟地址la在page directory entry 的 index
     *   KADDR(pa) : takes a physical address and returns the corresponding kernel virtual address.
     *   set_page_ref(page,1) : means the page be referenced by one time，这一页被引用了
     *   page2pa(page): get the physical address of memory which this (struct Page *) page manages
     *                  得到这个页管理的内存的物理地址
     *   struct Page * alloc_page() : allocation a page
     *   memset(void *s, char c, size_t n) : sets the first n bytes of the memory area pointed by s
     *                                       to the specified value c.
     * DEFINEs:
     *   PTE_P           0x001                   // page table/directory entry flags bit : Present
     *   PTE_W           0x002                   // page table/directory entry flags bit : Writeable
     *   PTE_U           0x004                   // page table/directory entry flags bit : User can access
     */

    pde_t *pdep = &pgdir[PDX(la)];       // (1) find page directory entry
    struct Page *page;
    if (!(*pdep & PTE_P) ) {             // (2) check if entry is not present
        if (!create || (page = alloc_page()) == NULL) {
	    return NULL;
     	}	                             // (3) check if creating is needed, then alloc page for page table
        // CAUTION: this page is used for page table, not for common data page
        set_page_ref(page, 1);           // (4) set page reference
        uintptr_t pa = page2pa(page);    // (5) get linear address of page
        memset(KADDR(pa),0,PGSIZE);      // (6) clear page content using memset
        *pdep = pa | PTE_U | PTE_W | PTE_P;// (7) set page directory entry's permission
    }
    return &((pte_t *)KADDR(PDE_ADDR(*pdep)))[PTX(la)];  // (8) return page table entry

}
```
## 练习三
```
//page_remove_pte - free an Page sturct which is related linear address la
//                - and clean(invalidate) pte which is related linear address la
//note: PT is changed, so the TLB need to be invalidate
static inline void
page_remove_pte(pde_t *pgdir, uintptr_t la, pte_t *ptep) {
    /* LAB2 EXERCISE 3: YOUR CODE
     *
     * Please check if ptep is valid, and tlb must be manually updated if mapping is updated
     *
     * Maybe you want help comment, BELOW comments can help you finish the code
     *
     * Some Useful MACROs and DEFINEs, you can use them in below implementation.
     * MACROs or Functions:
     *   struct Page *page pte2page(*ptep): get the according page from the value of a ptep
     *   free_page : free a page
     *   page_ref_dec(page) : decrease page->ref. NOTICE: ff page->ref == 0 , then this page should be free.
     *   tlb_invalidate(pde_t *pgdir, uintptr_t la) : Invalidate a TLB entry, but only if the page tables being
     *                        edited are the ones currently in use by the processor.
     * DEFINEs:
     *   PTE_P           0x001                   // page table/directory entry flags bit : Present
     */
#if 0
    if (0) {                      //(1) check if this page table entry is present
        struct Page *page = NULL; //(2) find corresponding page to pte
                                  //(3) decrease page reference
                                  //(4) and free this page when page reference reachs 0
                                  //(5) clear second page table entry
                                  //(6) flush tlb
    }
#endif
    if (*ptep & PTE_P) { // 确保传进来的二级页表时可用的
        struct Page *page = pte2page(*ptep);// 获取页表项对应的物理页的Page结构
        if (page_ref_dec(page) == 0) {	    // page_ref_dec被用于page->ref自减1，
        									// 如果返回值是0，那么就说明不存在任何虚拟页指向该物理页，释放该物理页
            free_page(page);
        }
        *ptep = 0;					 // 将PTE的映射关系清空
        tlb_invalidate(pgdir, la);   // 刷新TLB，确保TLB的缓存中不会有错误的映射关系
    }
}
```
问题：
```
数据结构Page的全局变量（其实是一个数组）的每一项与页表中的页目录项和页表项有无对应关系？如果有，其对应关系是啥？

存在对应关系：由于页表项中存放着对应的物理页的物理地址，因此可以通过这个物理地址来获取到对应到的Page数组的对应项，具体做法为将物理地址除以一个页的大小，然后乘上一个Page结构的大小获得偏移量，使用偏移量加上Page数组的基地址皆可以或得到对应Page项的地址；

如果希望虚拟地址与物理地址相等，则需要如何修改lab2，完成此事？ 鼓励通过编程来具体完成这个问题。

由于在完全启动了ucore之后，虚拟地址和线性地址相等，都等于物理地址加上0xc0000000，如果需要虚拟地址和物理地址相等，可以考虑更新gdt，更新段映射，使得virtual address = linear address - 0xc0000000，这样的话就可以实现virtual address = physical address；
reference：https://www.jianshu.com/p/abbe81dfe016
```
