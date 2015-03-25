# 练习1 理解通过make生成执行文件的过程

## 操作系统镜像文件ucore.img是如何一步一步生成的

1. create kernel target

    gcc ...
    ld -m    elf_i386 -nostdlib -T tools/kernel.ld -o bin/kernel

2. create bootblock.out

    gcc ...
    ld -m    elf_i386 -nostdlib -N -e start -Ttext 0x7C00 obj/boot/bootasm.o obj/boot/bootmain.o -o obj/bootblock.o
    objcopy -S -O binary obj/bootblock.o obj/bootblock.out

3. create sign tools

    gcc -Itools/ -g -Wall -O2 -c tools/sign.c -o obj/sign/tools/sign.o
    gcc -g -Wall -O2 obj/sign/tools/sign.o -o bin/sign

4. use sign to create 512 bytes bootblock

    bin/sign obj/bootblock.out bin/bootblock

5. create ucore.img

    dd if=/dev/zero of=bin/ucore.img count=10000
    dd if=bin/bootblock of=bin/ucore.img conv=notrunc
    dd if=bin/kernel of=bin/ucore.img seek=1 conv=notrunc

## 硬盘主引导扇区的特征

硬盘主引导扇区占据一个扇区,共512(200H)个字节，具体结构如下：

1. 硬盘主引导程序，位于该扇区的0－1BD处；

2. 硬盘分区表，位于1BEH－1FDH处，每个分区表占用16个字节，共4个分区表，16个字节各字节意义如下：

　　0:自举标志,80H为可引导分区,00为不可引导分区;

　　1～3:本分区在硬盘上的开始物理地址；

　　4:分区类型，其中1表示为12位FAT表的基本DOS分区；

　　4为16位FAT表的基本DOS分区；5为扩展DOS分区；

　　6为大于32M的DOS分区；

　　其它为非DOS分区。

　　5～7:本分区的结束地址；

　　8～11:该分区之前的扇区数，即此分区第一扇区的绝对扇区号；

　　12～15:该分区占用的总扇区数。

3. 引导扇区的有效标志,位于1FEH－1FFH处,固定值为AA55。

# 练习2 使用qemu执行并调试lab1中的软件

在调用qemu时增加`-d in_asm -D q.log`参数，便可以将运行的汇编指令保存在q.log中。

删除 `tools/gdbinit` 中的 `continue` 行，并将断点设置在 0x7c00

# 练习3 分析bootloader进入保护模式的过程

从地址 0x7c00 开始执行 bootloader

1. 禁止中断，清理环境

2. 开启A20

3. 加载 GDT 表

	GDT 表是用 asm.h 的一个宏设置的

	![描述符](http://chyyuu.gitbooks.io/ucorebook/content/zh/chapter-1/figures/3.15.3.png)

4. 使能cr0，进入保护模式

5. 长跳转，更新CS基地址，设置环境，call bootmain

# 练习4 分析 bootloader 加载 ELF 格式的 OS 的过程

bootloader让80386处理器进入保护模式后，下一步的工作就是从硬盘上加载并运行OS。

硬盘模式采用 ATA PIO 模式，28bit 地址（最大空间137G，另有48bit，能突破这一限制；工作模式还有 DMA 方式），具体参考 [Wikipedia ATA_PI0_MODE](http://wiki.osdev.org/ATA_PIO_Mode)。这里用到了两个函数，底层的 readsect 从设备的第secno扇区读取数据到dst位置，封装的 readseg 可以从设备读取任意长度的内容。

bootmain 过程：从磁盘读取ucore内核(ELF文件)，检查magic数，按 ELF 文件头内容将ucore内核加载到指定内存位置，从entry开始运行。

# 练习5 实现函数调用堆栈跟踪函数

在kdebug.c中实现print_stackframe

~~~~
void
print_stackframe(void) {
     /* LAB1 YOUR CODE : STEP 1 */
     /* (1) call read_ebp() to get the value of ebp. the type is (uint32_t);
      * (2) call read_eip() to get the value of eip. the type is (uint32_t);
      * (3) from 0 .. STACKFRAME_DEPTH
      *    (3.1) printf value of ebp, eip
      *    (3.2) (uint32_t)calling arguments [0..4] = the contents in address (unit32_t)ebp +2 [0..4]
      *    (3.3) cprintf("\n");
      *    (3.4) call print_debuginfo(eip-1) to print the C calling function name and line number, etc.
      *    (3.5) popup a calling stackframe
      *           NOTICE: the calling funciton's return addr eip  = ss:[ebp+4]
      *                   the calling funciton's ebp = ss:[ebp]
      */
    uint32_t ebp = read_ebp();
    uint32_t eip = read_eip();

    int i;
    for (i = 0; i < STACKFRAME_DEPTH; i++) {
        cprintf("ebp:0x%08x eip:0x%08x args:", ebp, eip);
        uint32_t *args = (uint32_t *)ebp + 2;
        uint32_t *args_end = args + 4;
        for (; args < args_end; args++)
            cprintf("0x%08x ", *args);
        cprintf("\n");
        print_debuginfo(eip - 1);
        eip = *((uint32_t *)ebp + 1);
        ebp = *((uint32_t *)ebp);
    }
}
~~~~

输出
~~~~
ebp:0x00007bc8 eip:0x00100050 args:0x00000000 0x00000000 0x00010094 0x00000000
    kern/init/init.c:28: kern_init+79
~~~~

ebp: 栈基址；eip：指令寄存器，指令地址；args：参数列表

# 练习6 完善中断初始化和处理

中断向量表中一个表项占8个字节，低32位代表中断处理代码入口：0～15为offset，16～31为段选择子
