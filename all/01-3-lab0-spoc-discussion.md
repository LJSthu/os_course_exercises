# lec2：lab0 SPOC思考题

## **提前准备**
（请在上课前完成，option）

- 完成lec2的视频学习
- git pull ucore_os_lab, os_tutorial_lab, os_course_exercises  in github repos。这样可以在本机上完成课堂练习。
- 了解代码段，数据段，执行文件，执行文件格式，堆，栈，控制流，函数调用,函数参数传递，用户态（用户模式），内核态（内核模式）等基本概念。思考一下这些基本概念在不同操作系统（如linux, ucore,etc.)与不同硬件（如 x86, riscv, v9-cpu,etc.)中是如何相互配合来体现的。
- 安装好ucore实验环境，能够编译运行ucore labs中的源码。
- 会使用linux中的shell命令:objdump，nm，file, strace，gdb等，了解这些命令的用途。
- 会编译，运行，使用v9-cpu的dis,xc, xem命令（包括启动参数），阅读v9-cpu中的v9\-computer.md文档，了解汇编指令的类型和含义等，了解v9-cpu的细节。
- 了解基于v9-cpu的执行文件的格式和内容，以及它是如何加载到v9-cpu的内存中的。
- 在piazza上就学习中不理解问题进行提问。

---

## 思考题

- 你理解的对于类似ucore这样需要进程/虚存/文件系统的操作系统，在硬件设计上至少需要有哪些直接的支持？至少应该提供哪些功能的特权指令？
* 答：
    * 进程之间的切换：需要硬件支持的中断
    * 虚存管理：需要硬件支持的地址映射，MMU,TLB等模块
    * 文件系统：需要硬件提供存储介质，寄存器堆、内存、cache等部分
    * 特权指令：
        * 有关对I/O设备使用的指令 如启动I/O设备指令、测试I/O设备工作状态和控制I/O设备动作的指令等
        * 允许和禁止中断，控制中断禁止屏蔽位
        * 在进程间切换处理
        * 设置时钟
        * 处理页表、快表相关的操作
        * 清理内存
- 你理解的x86的实模式和保护模式有什么区别？物理地址、线性地址、逻辑地址的含义分别是什么？
- 答：
    - 根本区别在于进程内存是否受保护
        * 实模式：将整个物理内存看成分段的区域,程序代码和数据位于不同区域，系统程序和用户程序没有区别对待，而且每一个指针都是指向"实在"的
        物理地址。这样一来，用户程序的一个指针如果指向了系统程序区域或其他用户程序区域，并改变了值，那么对于这个被修改的系统程序或用户程序，
        其后果就很可能是灾难性的
        * 保护模式：物理内存地址不能直接被程序访问，程序内部的地址（虚拟地址）要由操作系统转化为物理地址去访问，程序对此一无所知。
    - 可以访问的内存空间大小上也有区别，实模式只能访问1MB的空间，保护模式可以使用全部的4GB内存空间(32位)

- 你理解的risc-v的特权模式有什么区别？不同模式在地址访问方面有何特征？
- 答：

- 理解ucore中list_entry双向链表数据结构及其4个基本操作函数和ucore中一些基于它的代码实现（此题不用填写内容）

- 对于如下的代码段，请说明":"后面的数字是什么含义
* 答：代表了这个变量在结构体中所占的二进制位数,使用了C语言中的位域。
```
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


- 对于如下的代码段，

```
#define SETGATE(gate, istrap, sel, off, dpl) {            \
    (gate).gd_off_15_0 = (uint32_t)(off) & 0xffff;        \
    (gate).gd_ss = (sel);                                \
    (gate).gd_args = 0;                                    \
    (gate).gd_rsv1 = 0;                                    \
    (gate).gd_type = (istrap) ? STS_TG32 : STS_IG32;    \
    (gate).gd_s = 0;                                    \
    (gate).gd_dpl = (dpl);                                \
    (gate).gd_p = 1;                                    \
    (gate).gd_off_31_16 = (uint32_t)(off) >> 16;        \
}
```
如果在其他代码段中有如下语句，
```
unsigned intr;
intr=8;
SETGATE(intr, 1,2,3,0);
```
请问执行上述指令后， intr的值是多少？
* 答：gatedesc结构体有64位，因使用了位域，所以不存在struct中不同成员变量之间的对齐问题，下面分析代码的执行
    * (gate).gd_off_15_0 = (uint32_t)(off) & 0xffff;之后，结构体的低16位为：0x0003
    * (gate).gd_ss = (sel);之后，结构体的16~31位为0002,低32位为：0x00020003
    * (gate).gd_args = 0; (gate).gd_rsv1 = 0; (gate).gd_type = (istrap) ? STS_TG32 : STS_IG32; (gate).gd_s = 0; (gate).gd_dpl = (dpl); (gate).gd_p = 1;
    之后，结构体的低48位为：0x1f0000020003
    * (gate).gd_off_31_16 = (uint32_t)(off) >> 16;之后整个结构体用16进制表示为:0x0000ee0000020003
    * 最终输出的是unsigned int类型，截取低32位，结果为：0x20003

### 课堂实践练习

#### 练习一

1. 请在ucore中找一段你认为难度适当的AT&T格式X86汇编代码，尝试解释其含义。
- 代码段： trap/trapentry.S
- 解释：vector.S中对中断向量进行了设置，之后全部跳转到了_alltraps位置；其余关于代码的解释见注释
```
.text
.globl __alltraps
__alltraps:
    # 先保存寄存器的值到栈中，其中%ds是data segment寄存器，%es,%fs,%gs三个附加段寄存器
    pushl %ds
    pushl %es
    pushl %fs
    pushl %gs
    
    # 保存了所有的通用寄存器的值，与后面的popal相对应
    pushal
    # load GD_KDATA into %ds and %es to set up data segments for kernel
    movl $GD_KDATA, %eax
    movw %ax, %ds
    movw %ax, %es
    # 将%esp中的值作为参数传递给了trap()
    pushl %esp
    # 调用trap()
    call trap
    # 恢复了%esp
    popl %esp
    # 继续执行下面的中断恢复部分.

.globl __trapret
__trapret:
    # restore registers from stack

    popal
    # restore %ds, %es, %fs and %gs
    popl %gs
    popl %fs
    popl %es
    popl %ds
    # get rid of the trap number and error code
    addl $0x8, %esp
    # 恢复
    iret
```
2. (option)请在rcore中找一段你认为难度适当的RV汇编代码，尝试解释其含义。
- 代码：arch/riscv32/boot/trap.asm
- 解释：这段代码不长，主要是中断处理，首先通过SAVE_ALL保存现场，之后跳转到中断处理，最后再恢复。值得注意的是，这里的SAVE_ALL与RESTORE_ALL
使用了宏来完成，在下一问中会进行一点分析。
```
.section .text
    .globl __alltraps
__alltraps:
    SAVE_ALL
    mv a0, sp
    jal rust_trap
    .globl __trapret
__trapret:
    RESTORE_ALL
    # return from supervisor call
    XRET
```


#### 练习二

宏定义和引用在内核代码中很常用。请枚举ucore或rcore中宏定义的用途，并举例描述其含义。
- 答：rcore/arch/riscv32/trap.asm中使用了宏，分别为SAVE_ALL与RESTORE_ALL，用来完成处理中断之前的保存与处理后的恢复操作。
```
.macro SAVE_ALL

    # If coming from userspace, preserve the user stack pointer and load

    # the kernel stack pointer. If we came from the kernel, sscratch

    # will contain 0, and we should continue on the current stack.

    csrrw sp, (xscratch), sp

    bnez sp, _save_context

_restore_kernel_sp:

    csrr sp, (xscratch)

    # sscratch = previous-sp, sp = kernel-sp

_save_context:

    # provide room for trap frame

    addi sp, sp, -36 * XLENB

    # save x registers except x2 (sp)

    STORE x1, 1

    STORE x3, 3

    # tp(x4) = hartid. DON'T change.

    # STORE x4, 4

    STORE x5, 5

    STORE x6, 6

    STORE x7, 7

    STORE x8, 8

    STORE x9, 9

    STORE x10, 10

    STORE x11, 11

    STORE x12, 12

    STORE x13, 13

    STORE x14, 14

    STORE x15, 15

    STORE x16, 16

    STORE x17, 17

    STORE x18, 18

    STORE x19, 19

    STORE x20, 20

    STORE x21, 21

    STORE x22, 22

    STORE x23, 23

    STORE x24, 24

    STORE x25, 25

    STORE x26, 26

    STORE x27, 27

    STORE x28, 28

    STORE x29, 29

    STORE x30, 30

    STORE x31, 31



    # get sp, sstatus, sepc, stval, scause

    # set sscratch = 0

    csrrw s0, (xscratch), x0

    csrr s1, (xstatus)

    csrr s2, (xepc)

    csrr s3, (xtval)

    csrr s4, (xcause)

    # store sp, sstatus, sepc, sbadvaddr, scause

    STORE s0, 2

    STORE s1, 32

    STORE s2, 33

    STORE s3, 34

    STORE s4, 35

.endm



.macro RESTORE_ALL

    LOAD s1, 32             # s1 = sstatus

    LOAD s2, 33             # s2 = sepc

    TEST_BACK_TO_KERNEL

    bnez s0, _restore_context   # s0 = back to kernel?

_save_kernel_sp:

    addi s0, sp, 36*XLENB

    csrw (xscratch), s0         # sscratch = kernel-sp

_restore_context:

    # restore sstatus, sepc

    csrw (xstatus), s1

    csrw (xepc), s2



    # restore x registers except x2 (sp)

    LOAD x1, 1

    LOAD x3, 3

    # LOAD x4, 4

    LOAD x5, 5

    LOAD x6, 6

    LOAD x7, 7

    LOAD x8, 8

    LOAD x9, 9

    LOAD x10, 10

    LOAD x11, 11

    LOAD x12, 12

    LOAD x13, 13

    LOAD x14, 14

    LOAD x15, 15

    LOAD x16, 16

    LOAD x17, 17

    LOAD x18, 18

    LOAD x19, 19

    LOAD x20, 20

    LOAD x21, 21

    LOAD x22, 22

    LOAD x23, 23

    LOAD x24, 24

    LOAD x25, 25

    LOAD x26, 26

    LOAD x27, 27

    LOAD x28, 28

    LOAD x29, 29

    LOAD x30, 30

    LOAD x31, 31

    # restore sp last

    LOAD x2, 2

.endm
```


#### reference
 - [Intel格式和AT&T格式汇编区别](http://www.cnblogs.com/hdk1993/p/4820353.html)
 - [x86汇编指令集  ](http://hiyyp1234.blog.163.com/blog/static/67786373200981811422948/)
 - [PC Assembly Language, Paul A. Carter, November 2003.](https://pdos.csail.mit.edu/6.828/2016/readings/pcasm-book.pdf)
 - [*Intel 80386 Programmer's Reference Manual*, 1987](https://pdos.csail.mit.edu/6.828/2016/readings/i386/toc.htm)
 - [IA-32 Intel Architecture Software Developer's Manuals](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html)
