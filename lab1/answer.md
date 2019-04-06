# 操作系统镜像文件ucore.img是如何一步一步生成的？(需要比较详细地解释Makefile中每一条相关命令和命令参数的含义，以及说明命令导致的结果)

用*make "V="*看到了所有的编译命令
第178行 create ucore.img，可以看到call函数，  
totarget = $(addprefix $(BINDIR)$(SLASH),$(1))  
这样就调用了addprefix，把$(BINDIR)$(SLASH)变成$(1)的前缀，在makefile里再把$(1)调用call变成要生成的文件，这里需要bootblock和kernel。  
bootblock需要一些.o文件，makefile里的foreach有如下格式：$(foreach < var >,< list >,< text >)  
这个函数的意思是，把参数< list >;中的单词逐一取出放到参数< var >所指定的变量中，然后再执行< text>;所包含的表达式。每一次< text >会返回一个字符串，循环过程中，< text >的所返回的每个字符串会以空格分隔，最后当整个循环结束时，< text >所返回的每个字符串所组成的整个字符串（以空格分隔）将会是foreach函数的返回值。
- 通过看makefile生成的编译命令，生成bootasm.o需要bootasm.S
```
gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc -c boot/bootasm.S -o obj/boot/bootasm.o
```
参考：-ggdb  生成可供gdb使用的调试信息。这样才能用qemu+gdb来调试bootloader or ucore。  
-m32  生成适用于32位环境的代码。我们用的模拟硬件是32bit的80386，所以ucore也要是32位。  
-gstabs  生成stabs格式的调试信息。这样要ucore的monitor可以显示出便于开发者阅读的函数调用  
-nostdinc  不使用标准库。标准库是给应用程序用的，我们是编译ucore内核，OS内核是提供服务的
，所以所有的服务要自给自足。  
-fno-stack-protector  不生成用于检测缓冲区溢出的代码。这是for 应用程序的，我们是编译内核
，ucore内核好像还用不到此功能。  
-Os  为减小代码大小而进行优化。根据硬件spec，主引导扇区只有512字节，我们写的简单bootload
er的最终大小不能大于510字节。  
-I< dir >  添加搜索头文件的路径
```
 ld -m    elf_i386 -nostdlib -N -e start -Ttext 0x7C00 obj/boot/bootasm.o obj/boot/bootmain.o -o obj/bootblock.o
```
参考：-m <emulation>  模拟为i386上的连接器  
-nostdlib  不使用标准库  
-N  设置代码段和数据段均可读写  
-e <entry>  指定入口  
-Ttext  制定代码段开始位置  

```
kernel = $(call totarget,kernel)

$(kernel): tools/kernel.ld

$(kernel): $(KOBJS)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
	@$(OBJDUMP) -S $@ > $(call asmfile,kernel)
	@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)

$(call create_target,kernel)
```
编译命令：
```
gcc -Ikern/trap/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/trap/trapentry.S -o obj/kern/（o文件）
```
链接器：
```
ld -m    elf_i386 -nostdlib -T tools/kernel.ld -o bin/kernel  obj/kern/init/init.o obj/kern/libs/stdio.o obj/kern/libs/readline.o obj/kern/debug/panic.o obj/kern/debug/kdebug.o obj/kern/debug/kmonitor.o obj/kern/driver/clock.o obj/kern/driver/console.o obj/kern/driver/picirq.o obj/kern/driver/intr.o obj/kern/trap/trap.o obj/kern/trap/vectors.o obj/kern/trap/trapentry.o obj/kern/mm/pmm.o  obj/libs/string.o obj/libs/printfmt.o
```
dd：
> dd：用指定大小的块拷贝一个文件，并在拷贝的同时进行指定的转换。

> 注意：指定数字的地方若以下列字符结尾，则乘以相应的数字：b=512；c=1；k=1024；w=2

> 参数注释：

> 1. if=文件名：输入文件名，缺省为标准输入。即指定源文件。< if=input file >

> 2. of=文件名：输出文件名，缺省为标准输出。即指定目的文件。< of=output file >

> 3. ibs=bytes：一次读入bytes个字节，即指定一个块大小为bytes个字节。

>    obs=bytes：一次输出bytes个字节，即指定一个块大小为bytes个字节。

>    bs=bytes：同时设置读入/输出的块大小为bytes个字节。

> 4. cbs=bytes：一次转换bytes个字节，即指定转换缓冲区大小。

> 5. skip=blocks：从输入文件开头跳过blocks个块后再开始复制。

> 6. seek=blocks：从输出文件开头跳过blocks个块后再开始复制。

> 注意：通常只用当输出文件是磁盘或磁带时才有效，即备份到磁盘或磁带时才有效。

> 7. count=blocks：仅拷贝blocks个块，块大小等于ibs指定的字节数。

> 8. conv=conversion：用指定的参数转换文件。

>    ascii：转换ebcdic为ascii

>    ebcdic：转换ascii为ebcdic

>    ibm：转换ascii为alternate ebcdic

>    block：把每一行转换为长度为cbs，不足部分用空格填充

>    unblock：使每一行的长度都为cbs，不足部分用空格填充

>    lcase：把大写字符转换为小写字符

>    ucase：把小写字符转换为大写字符

>    swab：交换输入的每对字节

>    noerror：出错时不停止

>    notrunc：不截短输出文件

>    sync：将每个输入块填充到ibs个字节，不足部分用空（NUL）字符补齐。
```
生成一个有10000个块的文件，用0填充（答案中说，每个块默认512字节，但是可能要有bs参数指定或者bs默认就是512？）
dd if=/dev/zero of=bin/ucore.img count=10000

把bootblock中的内容写到第一个块
dd if=bin/bootblock of=bin/ucore.img conv=notrunc

从第二个块开始写kernel中的内容
dd if=bin/kernel of=bin/ucore.img seek=1 conv=notrunc
```

# 一个被系统认为是符合规范的硬盘主引导扇区的特征是什么？
上课讲过，合法的主引导扇区最后两个字节有特定值
0x55、0xAA
```
buf一共512个字节
buf[510] = 0x55;
buf[511] = 0xAA;
```
# 练习2：
```
file bin/kernel
set architecture i8086
target remote :1234
b *0x7c00
continue
```
在gdb中输入命令，输出2条instruction
```
x /2i $pc
```
跟bootasm.S里的汇编代码一致！amazing
```
(gdb) x /2i $pc
=> 0x7c00:	cli    
   0x7c01:	cld    
(gdb) x /10i $pc
=> 0x7c00:	cli    
   0x7c01:	cld    
   0x7c02:	xor    %ax,%ax
   0x7c04:	mov    %ax,%ds
   0x7c06:	mov    %ax,%es
   0x7c08:	mov    %ax,%ss
   0x7c0a:	in     $0x64,%al
   0x7c0c:	test   $0x2,%al
   0x7c0e:	jne    0x7c0a
   0x7c10:	mov    $0xd1,%al
```

在Makefile的debug选项中加入*-d in_asm -D q.log*，可以生成一个q.log里边是执行的汇编命令（部分）
```
----------------
IN:
0xfffffff0:  ljmp   $0xf000,$0xe05b

----------------
IN:
0x000fe05b:  cmpl   $0x0,%cs:0x6c48
0x000fe062:  jne    0xfd2e1

----------------
IN:
0x000fe066:  xor    %dx,%dx
0x000fe068:  mov    %dx,%ss

----------------
IN:
0x000fe06a:  mov    $0x7000,%esp

----------------
IN:
0x000fe070:  mov    $0xf3691,%edx
0x000fe076:  jmp    0xfd165

```

# 练习3
分析bootloader进入保护模式的过程。（要求在报告中写出分析）
BIOS将通过读取硬盘主引导扇区到内存，并转跳到对应内存中的位置执行bootloader。请分析bootloader是如何完成从实模式进入保护模式的。  
```
lab1/boot/bootasm.S
```
类似之前，从0x7c00进入，首先
```
.globl start
start:
        .code16
            cli				;禁止中断发生
            cld				;CLD与STD是用来操作方向标志位DF。CLD使DF复位，即D
            				  ;F=0，STD使DF置位，即DF=1.用于串操作指令中。
            xorw %ax, %ax   ;ax置0
            movw %ax, %ds   ;其他寄存器也清空
            movw %ax, %es
            movw %ax, %ss
```
\.globl指示告诉汇编器，\_start这个符号要被链接器用到，所以要在目标文件的符号表中标记它是一个全局符号（在第 5.1 节 “目标文件”详细解释）。\_start就像C程序的main函数一样特殊，是整个程序的入口，链接器在链接时会查找目标文件中的\_start符号代表的地址，把它设置为整个程序的入口地址，所以每个汇编程序都要提供一个\_start符号并且用.globl声明。如果一个符号没有用.globl声明，就表示这个符号不会被链接器用到。   
<br>
开启A20：到了80286，系统的地址总线有原来的20根发展为24根，这样能够访问的内存可以达到2^24=16M。Intel在设计80286时提出的目标是向下兼容。所以，在实模式下，系统所表现的行为应该和8086/8088所表现的完全一样，也就是说，在实模式下，80286以及后续系列，应该和8086/8088完全兼容。但最终，80286芯片却存在一个BUG：因为有了80286有A20线，如果程序员访问100000H-10FFEFH之间的内存，系统将实际访问这块内存，而不是象8086/8088一样从0开始。为了解决上述兼容性问题，IBM使用键盘控制器上剩余的一些输出线来管理第21根地址线（从0开始数是第20根） 的有效性，被称为A20 Gate:
> 如果A20 Gate被打开，则当程序员给出100000H-10FFEFH之间的地址的时候，系统将真正访问这块内存区域； 

> 如果A20 Gate被禁止，则当程序员给出100000H-10FFEFH之间的地址的时候，系统仍然使用8086/8088的方式即取模方式（8086仿真）。绝大多数IBM PC兼容机默认的A20 Gate是被禁止的。现在许多新型PC上存在直接通过BIOS功能调用来控制A20 Gate的功能。
```
        seta20.1:               
            inb $0x64, %al      ;0x64里的数据放到al中，即从I/O端口读取一个字节(BYTE,;HALF-WORD)
            testb $0x2, %al     ;检测
            jnz seta20.1        ;等到这个端口不忙，没有东西传进来

            movb $0xd1, %al     ; 0xd1 写到 0x64
            outb %al, $0x64     ;写8042输出端口

        seta20.2:                
            inb $0x64, %al      
            testb $0x2, %al     
            jnz seta20.2		;等不忙

            movb $0xdf, %al     ;打开A20 0xdf -> port 0x60
            outb %al, $0x60     ;0xdf = 11011111
```
初始化GDT表并打开保护模式
```
    lgdt gdtdesc		   ;让CPU读取gdtr_addr所指向内存内容保存到GDT内存当中
    movl %cr0, %eax		   ;cr0寄存器PE位or置1
    orl $CR0_PE_ON, %eax   
    movl %eax, %cr0
    ljmp $PROT_MODE_CSEG, $protcseg ;长跳改cs，基于段机制的寻址
```
最后初始化堆栈、寄存器，调用bootmain
```
protcseg:
    # 初始化寄存器
    movw $PROT_MODE_DSEG, %ax                       # Our data segment selector
    movw %ax, %ds                                   # -> DS: Data Segment
    movw %ax, %es                                   # -> ES: Extra Segment
    movw %ax, %fs                                   # -> FS
    movw %ax, %gs                                   # -> GS
    movw %ax, %ss                                   # -> SS: Stack Segment

    # Set up the stack pointer and call into C. The stack region is from 0--start(0x7c00)
    movl $0x0, %ebp
    movl $start, %esp
    call bootmain
```
# 练习四
对于bootmain.c，它唯一的工作就是从硬盘的第一个扇区启动格式为ELF的内核镜像；控制从boot.S文件开始--这个文件设置了保护模式和一个栈，这样C代码就可以运行了，然后再调用bootmain()。  

> 对x86.h头文件有：http://www.codeforge.cn/read/234474/x86.h__html
```
static inline uchar
inb(ushort port)
{
 
  uchar data;
 
  asm volatile("in %1,%0" : "=a" (data) : "d" (port));
  //对应 in port,data
  return data;
 
}
```
0x1F7：读 用来存放读操作后的状态  
readsect(void \*dst, uint32_t secno)从secno扇区读取数据到dst  

> 用汇编的方式实现读取1000号逻辑扇区开始的8个扇区  
IDE通道的通讯地址是0x1F0 - 0x1F7  
其中0x1F3 - 0x1F6 4个字节的端口是用来写入LBA地址的  
LBA就是 logical Block Address  
1000的16进制就是0x3E8  
向0x1F3 - 0x1F6写入 0x3E8  
向0x1F2这个地址写入扇区数量，也就是8  
向0X1F7写入要执行的操作命令码，对读操作的命令码是 0x20  
```
out 0x1F3 0x00
out 0x1F4 0x00
out 0x1F5 0x03
out 0x1F6 0xE8
out 0x1F2 0x08
out 0x1F7 0x20
```
outb的定义在x86.h中，封装out命令，将data输出到port端口
```
static inline void
outb(ushort port, uchar data)
{
 
  asm volatile("out %0,%1" : : "a" (data), "d" (port));
 
}
```
业界共同推出了 LBA48，采用 48 个比特来表示逻辑扇区号。如此一来，就可以管理131072 TB 的硬盘容量了。在这里我们采用将采用 LBA28 来访问硬盘。  
第1步：设置要读取的扇区数量。这个数值要写入0x1f2端口。这是个8位端口，因此每次只能读写255个扇区：
```
mov dx,0x1f2
mov al,0x01    ;1 个扇区
out dx,al
```
> 注意：如果写入的值为 0，则表示要读取 256 个扇区。每读一个扇区，这个数值就减一。因此，如果在读写过程中发生错误，该端口包含着尚未读取的扇区数。   

第2步：设置起始LBA扇区号。扇区的读写是连续的，因此只需要给出第一个扇区的编号就可以了。28 位的扇区号太长，需要将其分成 4 段，分别写入端口 0x1f3、0x1f4、0x1f5 和 0x1f6 号端口。其中，0x1f3 号端口存放的是 0～7 位；0x1f4 号端口存放的是 8～15 位；0x1f5 号端口存放的是 16～23 位，最后 4 位在 0x1f6 号端口。  

第3步:
向端口 0x1f7 写入 0x20，请求硬盘读。  

第4步:等待读写操作完成。端口0x1f7既是命令端口，又是状态端口。在通过这个端口发送读写命令之后，硬盘就忙乎开了。在它内部操作期间，它将 0x1f7 端口的第7位置“1”，表明自己很忙。一旦硬盘系统准备就绪，它再将此位清零，说明自己已经忙完了，同时将第3位置“1”，意思是准备好了，请求主机发送或者接收数据。  

第5步:连续取出数据。0x1f0 是硬盘接口的数据端口，而且还是一个16位端口。一旦硬盘控制器空闲，且准备就绪，就可以连续从这个端口写入或者读取数据。

```
    outb(0x1F2, 1);                         // 读取第一个数据块
    outb(0x1F3, secno & 0xFF);
    outb(0x1F4, (secno >> 8) & 0xFF);
    outb(0x1F5, (secno >> 16) & 0xFF);
    outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0);
    outb(0x1F7, 0x20);                      // cmd 0x20 - read sectors

    insl(0x1F0, dst, SECTSIZE / 4)          // 第五步
```
readseg函数简单包装了readsect，可以从设备读取任意长度的内容。
```
static void readseg(uintptr_t va, uint32_t count, uint32_t offset) {
  uintptr_t end_va = va + count;
  va -= offset % SECTSIZE;

  uint32_t secno = (offset / SECTSIZE) + 1;
  // 看是第几块，加1因为0扇区被引导占用,ELF文件从1扇区开始

  for (; va < end_va; va += SECTSIZE, secno ++) {
    readsect((void *)va, secno);//调用之前的封装函数对每一块进行处理
  }
}
```
对不同的文件，执行file命令如下：
```
file link.o 
link.o: ELF 64-bit LSB relocatable, x86-64, version 1 (SYSV), not stripped
 
file libfoo.so 
libfoo.so: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, BuildID[sha1]=871ecaf438d2ccdcd2e54cd8158b9d09a9f971a7, not stripped
 
file p1
p1: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.32, BuildID[sha1]=37f75ef01273a9c77f4b4739bcb7b63a4545d729, not stripped
 
file libfoo.so 
libfoo.so: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, BuildID[sha1]=871ecaf438d2ccdcd2e54cd8158b9d09a9f971a7, stripped
```
以下是主函数。
```
bootmain(void) {
    // read the 1st page off disk
    readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);

    // 看是不是标准的elf
    if (ELFHDR->e_magic != ELF_MAGIC) {
        goto bad;
    }

    struct proghdr *ph, *eph;

    // elf头中有elf文件应该加载到什么位置，将表头地址存在ph中
    ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);
    eph = ph + ELFHDR->e_phnum;
    for (; ph < eph; ph ++) {
        readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
    }

    // 找到内核的入口，这个函数不返回
    ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();

bad:
    outw(0x8A00, 0x8A00);
    outw(0x8A00, 0x8E00);

    /* do nothing */
    while (1);
}
```

> 一般的 ELF 文件包括三个索引表：ELF header，Program header table，Section header table。  
ELF header：在文件的开始，保存了路线图，描述了该文件的组织情况。  
Program header table：告诉系统如何创建进程映像。用来构造进程映像的目标文件必须具有程序头部表，可重定位文件不需要这个表。  
Section header table：包含了描述文件节区的信息，每个节区在表中都有一项，每一项给出诸如节区名称、节区大小这类信息。用于链接的目标文件必须包含节区头部表，其他目标文件可以有，也可以没有这个表。  
```
typedef struct
{
  unsigned char e_ident[EI_NIDENT]; /* Magic number and other info */
  Elf64_Half    e_type;         /* Object file type */
  Elf64_Half    e_machine;      /* Architecture */
  Elf64_Word    e_version;      /* Object file version */
  Elf64_Addr    e_entry;        /* Entry point virtual address */
  Elf64_Off e_phoff;        /* Program header table file offset */
  Elf64_Off e_shoff;        /* Section header table file offset */
  Elf64_Word    e_flags;        /* Processor-specific flags */
  Elf64_Half    e_ehsize;       /* ELF header size in bytes */
  Elf64_Half    e_phentsize;        /* Program header table entry size */
  Elf64_Half    e_phnum;        /* Program header table entry count */
  Elf64_Half    e_shentsize;        /* Section header table entry size */
  Elf64_Half    e_shnum;        /* Section header table entry count */
  Elf64_Half    e_shstrndx;     /* Section header string table index */
} Elf64_Ehdr;
```
ELF文件中有很多段，段表（Section Header Table）就是保存这些段的基本信息的结构，包括了段名、段长度、段在文件中的偏移位置、读写权限和其他段属性。
objdump工具可以查看ELF文件基本的段结构  
```
typedef struct
{
  Elf64_Word    sh_name;        /* Section name (string tbl index) */
  Elf64_Word    sh_type;        /* Section type */
  Elf64_Xword   sh_flags;       /* Section flags */
  Elf64_Addr    sh_addr;        /* Section virtual addr at execution */
  Elf64_Off sh_offset;      /* Section file offset */
  Elf64_Xword   sh_size;        /* Section size in bytes */
  Elf64_Word    sh_link;        /* Link to another section */
  Elf64_Word    sh_info;        /* Additional section information */
  Elf64_Xword   sh_addralign;       /* Section alignment */
  Elf64_Xword   sh_entsize;     /* Entry size if section holds table */
} Elf64_Shdr;
```

# 练习五
一个比较简单但很绕的逻辑，找到每个函数调用压栈时的指针，找到这个指针也就找到了上一个函数的部分，再找它之前的函数调用压栈的内容。  
主要问题是忘记了ebp!=0这个条件，忽视了要用16进制。  
eip是寄存器存放下一个CPU指令存放的内存地址，当CPU执行完当前的指令后，从eip寄存器中读取下一条指令的内存地址，然后继续执行；  
esp是寄存器存放当前线程的栈顶指针；  
ebp存放一个指针，该指针指向系统栈最上面一个栈帧的底部。即EBP寄存器存储的是栈底地址，而这个地址是由ESP在函数调用前传递给EBP的。等到调用结束，EBP会把其地址再次传回给ESP。所以ESP又一次指向了函数调用结束后，栈顶的地址。
```
void print_stackframe(void) {
     /* (1) call read_ebp() to get the value of ebp. the type is (uint32_t);
      * (2) call read_eip() to get the value of eip. the type is (uint32_t);
      * (3) from 0 .. STACKFRAME_DEPTH
      *    (3.1) printf value of ebp, eip
      *    (3.2) (uint32_t)calling arguments [0..4] = the contents in address (uint32_t)ebp +2 [0..4]
      *    (3.3) cprintf("\n");
      *    (3.4) call print_debuginfo(eip-1) to print the C calling function name and line number, etc.
      *    (3.5) popup a calling stackframe
      *           NOTICE: the calling funciton's return addr eip  = ss:[ebp+4]
      *                   the calling funciton's ebp = ss:[ebp]
      */
        uint32_t my_ebp = read_ebp();
        uint32_t my_eip = read_eip();//读取当前的ebp和eip
        int i,j;
        for(i = 0; my_ebp!=0 && i< STACKFRAME_DEPTH; i++){
                cprintf("%0x %0x\n",my_ebp,my_eip);
                for(j=0;j<4;j++){
                        cprintf("%0x\t",((uint32_t*)my_ebp+2)[j]);
                }
                cprintf("\n");
                print_debuginfo(my_eip-1);
                my_ebp = ((uint32_t*)my_ebp)[0];
                my_eip = ((uint32_t*)my_ebp)[1];
        }
}
```
ebp（基指针）寄存器主要通过软件约定与堆栈相关联。 在进入C函数时，函数的初始代码通常将先前函数的基本指针推入堆栈来保存，然后在函数持续时间内将当前esp值复制到ebp中。 如果程序中的所有函数都遵循这个约定，那么在程序执行期间的任何给定点，都可以通过跟踪保存的ebp指针链并确切地确定嵌套的函数调用序列引起这个特定的情况来追溯堆栈。 指向要达到的函数。 例如，当某个特定函数导致断言失败时，因为错误的参数传递给它，但您不确定是谁传递了错误的参数。 堆栈回溯可找到有问题的函数。  
最后一行对应的是第一个使用堆栈的函数，所以在栈的最深一层，就是bootmain.c中的bootmain。 bootloader起始的堆栈从0x7c00开始，使用"call bootmain"转入bootmain函数。 call指令压栈，所以bootmain中ebp为0x7bf8。

# 练习六
```
一个表项的结构如下:

/*lab1/kern/mm/mmu.h*/
/* Gate descriptors for interrupts and traps */
struct gatedesc {
    unsigned gd_off_15_0 : 16;        // low 16 bits of offset in segment
    unsigned gd_ss : 16;            // segment selector
    unsigned gd_args : 5;            // # args, 0 for interrupt/trap gates
    unsigned gd_rsv1 : 3;            // reserved(should be zero I guess)
    unsigned gd_type : 4;            // type(STS_{TG,IG32,TG32})
    unsigned gd_s : 1;                // must be 0 (system)
    unsigned gd_dpl : 2;            // descriptor(meaning new) privilege level
    unsigned gd_p : 1;                // Present
    unsigned gd_off_31_16 : 16;        // high bits of offset in segment
};
```
一个表项占用8字节，其中2-3字节是段选择子，0-1字节和6-7字节拼成位移， 两者联合便是中断处理程序的入口地址。(copy from answer)  
pic_init：中断控制器的初始化；idt_init：建立中断描述符表，并使能中断，intr_enable()
中断向量表可以认为是一个大数组，产生中断时生成一个中断号，来查这个idt表，找到中断服务例程的地址（段选择子加offset）。
主要是调用SETGATE这个宏对interrupt descriptor table进行初始化，是之前看到的对每个字节进行操作。然后调用lidt进行load idt（sti：使能中断）
```
建立一个中断描述符
  - istrap: 1 是一个trap, 0 代表中断
  - sel: 中断处理代码段
  - off: 中断处理代码段偏移
  - dpl: 描述符的优先级
*/
#define SETGATE(gate, istrap, sel, off, dpl)
```
除了系统调用中断(T_SYSCALL)使用陷阱门描述符且权限为用户态权限以外，其它中断均使用特权级(DPL)为０的中断门描述符，权限为内核态权限；
1. 中断描述符表（Interrupt Descriptor Table）中断描述符表把每个中断或异常编号和一个指向中断服务例程的描述符联系起来。同GDT一样，IDT是一个8字节的描述符数组，但IDT的第一项可以包含一个描述符。CPU把中断（异常）号*乘以8*做为IDT的索引。IDT可以位于内存的任意位置，CPU通过IDT寄存器（IDTR）的内容来寻址IDT的起始地址。指令LIDT和SIDT用来操作IDTR。两条指令都有一个显示的操作数：一个6字节表示的内存地址。在保护模式下，最多会存在256个Interrupt/Exception Vectors。
```
        extern uintptr_t __vectors[];
        int i;
        //for(i=0;i<256;i++)
        for(i=0;i< sizeof(idt) / sizeof(struct gatedesc); i++){
                SETGATE(idt[i],0,GD_KTEXT,__vectors[i],DPL_KERNEL);
        }
//      SETGATE(idt[T_SWITCH_TOK], 0, GD_KTEXT, __vectors[T_SWITCH_TOK], DPL_USER);
        SETGATE(idt[T_SWITCH_TOK], 1, GD_KTEXT, __vectors[T_SWITCH_TOK], DPL_USER);
        lidt(&idt_pd);
 ```
 对idt中的每一项，调用SETGATE进行设置，第二个是0表明是一个中断，如果是1表明是一个陷阱；GD_KTEXT是SEG_KTEXT（1，全局段编号）乘8，是处理中断的代码段编号，\_\_vectors[i]是作为在代码段中的偏移量，vectors[i]在kern/trap/vectors.S中定义，定义了255个中断服务例程的地址，这里才是入口，且都跳转到__alltraps。在trap中调用了trap_dispatch，这样就根据传进来的进行switch处理。  
 用户态设置在特权级3，内核态设置在特权级0。

 # 练习七
 这个实验实现用户态和内核态的转换，通过看代码基本明白。在init.c中的lab1_switch_to_user函数时一段汇编代码， 触发中断的话，有‘int %0’，就把第二个冒号（输入的数，T_SWITCH_TOK）替换%0， 这样中断号就是T_SWITCH_TOK。  
 SETGATE设置中断向量表将每个中断处理例程的入口设成__vector[i]的值，然后在有中断时，找到中断向量表中这个中断的处理例程，都是跳到__alltraps，\_\_alltraps把寄存器（ds es fs gs）压栈，把esp压栈，这样假装构造一个trapframe然后调用trap，trap调用了trap_dispatch
 在trap_dispatch中，对从堆栈弹出的段寄存器进行修改，转成User时和转成Kernel时不一样，分别赋值，同时需要修改之前的trapframe，实现中断的恢复。
 ```
     //LAB1 CHALLENGE 1 : YOUR CODE you should modify below codes.
    case T_SWITCH_TOU:
        if(tf->tf_cs != USER_CS){
                tf->tf_cs = USER_CS;
                tf->tf_ds = USER_DS;
                tf->tf_es = USER_DS;
                tf->tf_ss = USER_DS;
                tf->tf_eflags |= FL_IOPL_MASK;
                *((uint32_t*)tf - 1) = (uint32_t)tf;
        }
        break;
    case T_SWITCH_TOK:
        if(tf->tf_cs != KERNEL_CS) {
        tf->tf_cs = KERNEL_CS;
        tf->tf_ds = KERNEL_DS;
        tf->tf_es = KERNEL_DS;
        tf->tf_eflags &= ~FL_IOPL_MASK;
        struct trapframe *switchu2k = (struct trapframe *)(tf->tf_esp - (sizeof(struct trapframe) - 8));
        memmove(switchu2k,tf,sizeof(struct trapframe)-8);
        *((uint32_t *)tf-1)=(uint32_t)switchu2k;
        }
        break;
 ```