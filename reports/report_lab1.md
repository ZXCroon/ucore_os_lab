# Lab1实验报告

## 练习1

### 1.1
1. 将内核**kern**目录所有子目录下的源文件和库**libs**目录下的源文件转换成可重定位目标文件（**.o**）。相应代码如下：

	```
	$(call add_files_cc,$(call listf_cc,$(LIBDIR)),libs,)
	...
	$(call add_files_cc,$(call listf_cc,$(KSRCDIR)),kernel,$(KCFLAGS))
	```

	`$(LIBDIR)`和`$(KSRCDIR)`分别是**libs**目录**kern**下所有子目录，通过`listf_cc`取出目录下所有文件名。`$(KCFLAGS)`表示编译标志。`add_files`第一个参数是源文件列表（即`listf_cc的结果`）；第二个参数是一个包名，为生成的**.o**作一个标记，方便之后调用；第三个参数为编译标志。通过`add_files_cc`将指定的源文件编译为**.o**文件，放在**obj/\***下（\*为源文件的路径）。

2. 生成kernel的可执行目标文件。相应代码如下：

	```
	KOBJS = $(call read_packet,kernel libs)
	kernel = $(call totarget,kernel)
	$(kernel): tools/kernel.ld
	$(kernel): $(KOBJS)
	  @echo + ld $@
	  $(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
	  @$(OBJDUMP) -S $@ > $(call asmfile,kernel)
	  @$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)
	```

	`read_packet`将刚刚生成的`kernel`和`libs`包中的所有**.o**文件的名字放入`KOBJS`中。`totarget`即生成最终目标文件的路径名。`$(kernel)`是承载着操作系统的elf文件，生成它需要将所有**.o**文件链接起来。**tools/kernel.ld**为链接所需的配置文件，指明了链接时各段的位置信息同时提供一些符号的值。

	链接过程在以上代码段的第6行。`$(LDFLAGS)`为链接时的一些标志；通过`-T`来制定链接文件；`$@`为目标文件，即`$(kernel)`；`$(KOBJS)`为链接的源文件，即**.o**文件。

	后面的几行是生成链接出来的完整汇编和符号信息，方便调试，并不直接参与**ucore.img**的生成过程。

3. 将**boot**目录下的文件转换成**.o**文件。相应代码如下：

	```
	bootfiles = $(call listf_cc,boot)
	$(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))
	```

	通过`listf_cc`取出boot下所有源文件后用`cc_compile`编译成**.o**文件。`-Os`表示打开优化并同时保持代码长度不发生变化；`-nostdinc`表示识别头文件时不搜索标准系统库。

4. 生成bootblock启动可执行文件，也即主引导扇区。相应代码如下：

	```
	bootblock = $(call totarget,bootblock)
	$(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
	@$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
	@$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
	@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)
	$(call create_target,bootblock)
	```

	与生成kernel的可执行文件相同的部分不加赘述。第4行的链接直接把配置信息写在命令行中：`-N`表示把`.text`和`.data`节设置为可读写，同时取消`.data`节的页对齐和对共享库的链接；`-e`指定入口为`start`；`-Ttext`将代码放在**0x7C00**处——这是BIOS的规范。

	第6行是用来将elf转换为二进制文件，以便写入扇区。最后通过`create_target`把目标文件放在合适的位置。

5. 将kernel和bootblock组合成要写入磁盘的若干扇区。相应代码如下：

	```
	UCOREIMG	:= $(call totarget,ucore.img)
	$(UCOREIMG): $(kernel) $(bootblock)
	$(V)dd if=/dev/zero of=$@ count=10000
	$(V)dd if=$(bootblock) of=$@ conv=notrunc
	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc
	$(call create_target,ucore.img)
	```

	核心在于几个`dd`（功能是按块复制）。第一个`dd`用来把ucore.img前`count`个块清零；第二个`dd`将主引导扇区复制到ucore.img的开头处，并通过`conv=notrunc`禁止文件截断；第三个`dd`将操作系统所在的块复制到**ucore.img**，用`seek=1`跳过主引导扇区，即接在主引导扇区后面。如此一来便完成了写入磁盘的**ucore.img**的制作。

### 1.2
**tools/sign.c**中相关代码如下：

```
buf[510] = 0x55;
buf[511] = 0xAA;
```

可见一个被系统认为是符合规范的硬盘主引导扇区以**0x55**、**0xAA**两个字节作为结尾。

## 练习2

### 2.1
创建**tools/gdbinit\_lab1**，内容如下：

```
file bin/kernel
target remote :1234
set architecture i8086
```

依次执行`qemu-system-i386 -no-reboot -nographic -S -s -hda bin/ucore.img`和`gdb -x tools/gdbinit_lab1`，跳到第一条指令处，之后通过`si`单步跟踪BIOS执行。由于gdb本身的错误，无法正确地反汇编出指令。

### 2.2
在上述步骤后，通过`b *0x7c00`在**0x7c00**处设置断点。通过`c`继续执行程序。程序正确地停在了**0x7c00**处。

### 2.3
通过`x/10i $pc`打印处此时执行到的指令及其之后9条指令，和**bootblock.asm**以及**bootasm.S**里展示出来的`start`符号下的指令（**0x7c00**）对应。

### 2.4
通过`b pmm_init`设断点并`c`继续，使程序停在`pmm_init`起始位置，此过程正常。


## 练习3

### 3.1
在开启A20之前，能访问的地址空间只有20位，超过20位的地址会被映射到20位以内的地址。只有打开了A20，当访问超过20位的地址时，才能真正访问到这块内存。

开启A20需要用到8042键盘控制器，步骤如下：
1. 关闭中断。
2. 等待键盘输入缓冲区为空。做法是循环读取**0x64**端口的字节，直到它变为**0x2**。
3. 往**0x64**端口写入**0xd1**，表示写数据到8042的P2口。
4. 等待键盘输入缓冲区为空（即上述数据写入完成）。方法同上。
5. 往**0x60**端口写入**0xdf**，表示将P2口的A20位置1。

### 3.2
相关代码如下：

```
...
    lgdt gdtdesc
...
.p2align 2                                          # force 4 byte alignment
gdt:
    SEG_NULLASM                                     # null seg
    SEG_ASM(STA_X|STA_R, 0x0, 0xffffffff)           # code seg for bootloader and kernel
    SEG_ASM(STA_W, 0x0, 0xffffffff)                 # data seg for bootloader and kernel

gdtdesc:
    .word 0x17                                      # sizeof(gdt) - 1
    .long gdt                                       # address gdt
```

定义好GDT描述符`gdtdesc`后，用`lgdt`将它装载。对于`gdtdesc`，第一个字为**0x17**，即23，表示GDT表共占24个字节。因为每个段描述符占8个字节，所以这说明GDT表中有3个段描述符。`gdtdesc`接下来的4个字节则表示GDT表的地址。

再来看GDT表的定义。三个段描述符分别空描述符、代码段描述符、数据段描述符，它们通过宏`SEG_NULL`和`SEG`开辟8字节空间。

### 3.3
根据GDT表的定义，代码段描述符和数据段描述符分别在表中是第二和第三个描述符，相对GDT表的偏移分别是**0x8**和**0x10**，因此将即将赋给代码段选择子的值`PROT_MODE_CSEG`和即将赋给数据段选择子的值`PROT_MODE_DSEG`分别设为**0x8**和**0x10**。代码如下：

```
.set PROT_MODE_CSEG,        0x8                     # kernel code segment selector
.set PROT_MODE_DSEG,        0x10                    # kernel data segment selector
```

为了使能和进入保护模式，前提是准备好GDT（具体步骤已描述）。接着将`cr0`寄存器中表示使能保护模式的位置1即可使能保护模式——具体做法是取出`cr0`的值，将它或上一个对应位为1其余位为0的数`CR0_PE_ON`，再放回`cr0`。代码如下：

```
.set CR0_PE_ON,             0x1                     # protected mode enable flag
...
    movl %cr0, %eax
    orl $CR0_PE_ON, %eax
    movl %eax, %cr0
```

然后跳转到设置各段选择子的程序段`protcseg`，注意这个长跳转把`PROT_MODE_CSEG`作为了代码段选择子。在`protcseg`中，把数据段选择子及其他一些选择子都赋成`PROT_MODE_DSEG`。代码如下：

```
...
    ljmp $PROT_MODE_CSEG, $protcseg

.code32                                             # Assemble for 32-bit mode
protcseg:
    movw $PROT_MODE_DSEG, %ax                       # Our data segment selector
    movw %ax, %ds                                   # -> DS: Data Segment
    movw %ax, %es                                   # -> ES: Extra Segment
    movw %ax, %fs                                   # -> FS
    movw %ax, %gs                                   # -> GS
    movw %ax, %ss                                   # -> SS: Stack Segment
```

至此便进入了保护模式。接下来就是压栈传参调用bootloader的核心过程`bootmain`来启动操作系统了。

## 练习4

### 4.1
读取扇区的代码（**boot/bootmain.c**中的`readsect`函数）如下：

```
static void
readsect(void *dst, uint32_t secno) {
    // wait for disk to be ready
    waitdisk();

    outb(0x1F2, 1);                         // count = 1
    outb(0x1F3, secno & 0xFF);
    outb(0x1F4, (secno >> 8) & 0xFF);
    outb(0x1F5, (secno >> 16) & 0xFF);
    outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0);
    outb(0x1F7, 0x20);                      // cmd 0x20 - read sectors

    // wait for disk to be ready
    waitdisk();

    // read a sector
    insl(0x1F0, dst, SECTSIZE / 4);
}
```

以下对具体步骤进行说明：
1. 等待磁盘准备好，即调用`waitdisk`函数。`waitdisk`函数不断地读取硬盘**0x1F7**处的值（用作磁盘的状态和命令寄存器），检查相应位从而得到硬盘是否空闲地信息。
2. 往硬盘**0x1F2**写入要读取的扇区数；这里每次都读1个扇区。
3. 接下来将要读的扇区号写入硬盘相应的寄存器。分别往**0x1F3**、**0x1F4**、**0x1F5**、**0x1F6**写入扇区号`secno`第0-7位、8-15位、16-23位、24-27位的值。注意同时写入**0x1F6**的还有硬盘的主从信息，写入的是**0x1F6**的第4位，写入0来表示这是个主盘（通过或**0xE0**实现这个位的写入）。
4. 往硬盘**0x1F7**写入**0x20**表示发出读扇区的命令。
5. 再次调用`waitdisk`等待硬盘工作完毕。
6. 调用`insl`从硬盘的数据寄存器**0x1F0**读出数据。注意`insl`的读取长度是以双字为单位的，所以应该传入的是`SECTSIZE/4`。

### 4.2
首先分析`readseg`函数，其功能是读取硬盘中的一段区域到一个指定的内存地址。代码如下：

```
static void
readseg(uintptr_t va, uint32_t count, uint32_t offset) {
    uintptr_t end_va = va + count;
    va -= offset % SECTSIZE;
    uint32_t secno = (offset / SECTSIZE) + 1;
    for (; va < end_va; va += SECTSIZE, secno ++) {
        readsect((void *)va, secno);
    }
}
```

核心思想是读取这一段所在的若干扇区。`secno`是接下来要读取的扇区号，计算时要注意跳过第0号扇区（主引导扇区）。循环前要将目标地址与扇区边界对齐，以保证读进完整扇区后，想要读的第一个字节正好放在了想要存放数据的地址上。

接下来分析解析elf文件的过程。通过`readseg`从硬盘中读取一个页（8个扇区），用`ELFHDR`指向这个页在内存中的位置，以便接下来将这块内存看作elf头来进行解析。在解析之前，需要查看`e_magic`域是否是一个约定值来对这个elf文件的合法性做出一个初步判断。相关代码如下：

```
readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);
if (ELFHDR->e_magic != ELF_MAGIC) {
	goto bad;
}
```

如果不合法就跳到`bad`程序段处，对错误进行处理。

然后把视线转向程序头。第一个程序头的位置和程序头的数量都可以在elf头中找到（其中前者是以相对于elf头的基址的偏移的形式给出的，两个量分别存在`e_phoff`域和`e_phnum`域）。有了这两个信息，便可以对程序头进行遍历。对于每个程序头对应一段代码，同时包含三个关键信息：代码相对于elf的偏移`p_offset`、代码的长度`p_memsz`、代码要放的位置`p_va`。基于这三个信息，调用`readseg`，便能从硬盘中把这段代码复制到内存的指定位置。最后把程序跳转到入口`e_entry`，把控制权交给操作系统。相关代码如下：

```
struct proghdr *ph, *eph;

ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);
eph = ph + ELFHDR->e_phnum;
for (; ph < eph; ph ++) {
	readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
}

((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();
```

## 练习5

