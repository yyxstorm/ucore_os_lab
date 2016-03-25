#操作系统Lab1实验报告

2012011289
计22
姚宇轩

##对应知识点
练习一：Makefile语法、课程中编译相关的知识
练习二：学会使用 GDB
练习三：分段机制、保护模式、boot过程
练习四：分析bootloader加载ELF、ELF格式
练习五：实现函数调用堆栈跟踪函数，堆栈的构建方式
练习六：中断处理

  练习一：理解通过make生成执行文件的过程

   操作系统镜像文件ucore.img是如何一步一步生成的

    PROJ	:= challenge
    EMPTY	:=
    SPACE	:= $(EMPTY) $(EMPTY)
    SLASH	:= /
    V       := @
以上各句创建了一些变量。`：=`运算符代表覆盖之前的值。

    #need llvm/cang-3.5+
    #USELLVM := 1
    # try to infer the correct GCCPREFX
以#开头的句子是注释。此次编译需要llvm/cang-3.5+的环境。

    ifndef GCCPREFIX
    GCCPREFIX := $(shell if i386-elf-objdump -i 2>&1 | grep '^elf32-i386$$' >/dev/null 2>&1; \
    	then echo 'i386-elf-'; \
    	elif objdump -i 2>&1 | grep 'elf32-i386' >/dev/null 2>&1; \
    	then echo ''; \
    	else echo "***" 1>&2; \
    	echo "*** Error: Couldn't find an i386-elf version of GCC/binutils." 1>&2; \
    	echo "*** Is the directory with i386-elf-gcc in your PATH?" 1>&2; \
    	echo "*** If your i386-elf toolchain is installed with a command" 1>&2; \
    	echo "*** prefix other than 'i386-elf-', set your GCCPREFIX" 1>&2; \
    	echo "*** environment variable to that prefix and run 'make' again." 1>&2; \
    	echo "*** To turn off this error, run 'gmake GCCPREFIX= ...'." 1>&2; \
    	echo "***" 1>&2; exit 1; fi)
    endif
    
以上这段调用shell用来寻找GCC的位置，创建变量GCCPREFIX。`2>&1`是将STDERR的输出重定向到STDOUT上，这样就不会造成写的错乱；而`1>&2`是将STDOUT的输出重定向到STDERR上，这样可以直接输出在命令行。`/dev/null`是不记录日志的意思。
    
    # try to infer the correct QEMU
    ifndef QEMU
    QEMU := $(shell if which qemu-system-i386 > /dev/null; \
    	then echo 'qemu-system-i386'; exit; \
    	elif which i386-elf-qemu > /dev/null; \
    	then echo 'i386-elf-qemu'; exit; \
    	elif which qemu > /dev/null; \
    	then echo 'qemu'; exit; \
    	else \
    	echo "***" 1>&2; \
    	echo "*** Error: Couldn't find a working QEMU executable." 1>&2; \
    	echo "*** Is the directory containing the qemu binary in your PATH" 1>&2; \
    	echo "***" 1>&2; exit 1; fi)
    endif
    
以上这段用来找到可用的QEMU，创建变量QEMU，其意义和之前寻找GCC差不多。

    # eliminate default suffix rules
    .SUFFIXES: .c .S .h

替换掉默认的文件后缀，使用.c .S .h为后缀的文件。    

    # delete target files if there is an error (or make is interrupted)
    .DELETE_ON_ERROR:

如果发生错误，则删除掉目标文件。    

    # define compiler and flags
    ifndef  USELLVM
    HOSTCC		:= gcc
    HOSTCFLAGS	:= -g -Wall -O2
    CC		:= $(GCCPREFIX)gcc
    CFLAGS	:= -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc $(DEFS)
    CFLAGS	+= $(shell $(CC) -fno-stack-protector -E -x c /dev/null >/dev/null 2>&1 && echo -fno-stack-protector)
    else
    HOSTCC		:= clang
    HOSTCFLAGS	:= -g -Wall -O2
    CC		:= clang
    CFLAGS	:= -fno-builtin -Wall -g -m32 -mno-sse -nostdinc $(DEFS)
    CFLAGS	+= $(shell $(CC) -fno-stack-protector -E -x c /dev/null >/dev/null 2>&1 && echo -fno-stack-protector)
    endif
    
如果之前在第七行没有注释掉USELLVM，那么将采用clang编译；否则，将采用gcc编译。
`-g`：编译的时候产生编译信息，可以被GDB调试；
`-Wall`：开启编译器所有常用警告；
`-O2`：二级编译器优化；
`-fno-builtin`：关闭GCC内置函数功能，让它不能替代C库函数；
`-ggdb`：尽可能的生成gdb的可以使用的调试信息；
`-m32`：编译32位程序；
`-gstabs`：以stabs格式声称调试信息,但是不包括gdb调试信息；
`-nostdinc`：使编译器不再系统缺省的头文件目录里面找头文件；
`-fno-stack-protector`：禁用堆栈保护；
`-E`：只激活预处理；
`-x c`：设定语言为c；

    CTYPE	:= c S
    
    LD      := $(GCCPREFIX)ld
    LDFLAGS	:= -m $(shell $(LD) -V | grep elf_i386 2>/dev/null)
    LDFLAGS	+= -nostdlib
    
    OBJCOPY := $(GCCPREFIX)objcopy
    OBJDUMP := $(GCCPREFIX)objdump
    
    COPY	:= cp
    MKDIR   := mkdir -p
    MV		:= mv
    RM		:= rm -f
    AWK		:= awk
    SED		:= sed
    SH		:= sh
    TR		:= tr
    TOUCH	:= touch -c
    
    OBJDIR	:= obj
    BINDIR	:= bin
    
    ALLOBJS	:=
    ALLDEPS	:=
    TARGETS	:=
    
    include tools/function.mk
    
    listf_cc = $(call listf,$(1),$(CTYPE))
    
以上都是对编译器的设置以及宏定义。`include`一句引入了接下来很多要使用的函数。
`-nostdlib`：不连接系统标准启动文件和标准库文件,只把指定的文件传递给连接器。

    # for cc
    add_files_cc = $(call add_files,$(1),$(CC),$(CFLAGS) $(3),$(2),$(4))
    create_target_cc = $(call create_target,$(1),$(2),$(3),$(CC),$(CFLAGS))
    
    # for hostcc
    add_files_host = $(call add_files,$(1),$(HOSTCC),$(HOSTCFLAGS),$(2),$(3))
    create_target_host = $(call create_target,$(1),$(2),$(3),$(HOSTCC),$(HOSTCFLAGS))

设置CC和HOSTCC。
    
    cgtype = $(patsubst %.$(2),%.$(3),$(1))
    objfile = $(call toobj,$(1))
    asmfile = $(call cgtype,$(call toobj,$(1)),o,asm)
    outfile = $(call cgtype,$(call toobj,$(1)),o,out)
    symfile = $(call cgtype,$(call toobj,$(1)),o,sym)
    
    # for match pattern
    match = $(shell echo $(2) | $(AWK) '{for(i=1;i<=NF;i++){if(match("$(1)","^"$$(i)"$$")){exit 1;}}}'; echo $$?)
    
设置表达匹配。

    # >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>
    # include kernel/user
    
    INCLUDE	+= libs/
    
    CFLAGS	+= $(addprefix -I,$(INCLUDE))
    
    LIBDIR	+= libs
    
    $(call add_files_cc,$(call listf_cc,$(LIBDIR)),libs,)
    
    # -------------------------------------------------------------------
    # kernel
    
    KINCLUDE	+= kern/debug/ \
    			   kern/driver/ \
    			   kern/trap/ \
    			   kern/mm/
    
    KSRCDIR		+= kern/init \
    			   kern/libs \
    			   kern/debug \
    			   kern/driver \
    			   kern/trap \
    			   kern/mm
    
    KCFLAGS		+= $(addprefix -I,$(KINCLUDE))
    
    $(call add_files_cc,$(call listf_cc,$(KSRCDIR)),kernel,$(KCFLAGS))
    
    KOBJS	= $(call read_packet,kernel libs)
    
将各个待编译文件夹加入makefile。

    # create kernel target
    kernel = $(call totarget,kernel)
    
    $(kernel): tools/kernel.ld
    
    $(kernel): $(KOBJS)
    	@echo + ld $@
    	$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
    	@$(OBJDUMP) -S $@ > $(call asmfile,kernel)
    	@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)
    
    $(call create_target,kernel)
   
创建目标程序kernel。实际代码如下：

>  ld -m    elf_i386 -nostdlib -T tools/kernel.ld -o bin/kernel 
> obj/kern/init/init.o obj/kern/libs/readline.o obj/kern/libs/stdio.o
> obj/kern/debug/kdebug.o obj/kern/debug/kmonitor.o
> obj/kern/debug/panic.o obj/kern/driver/clock.o
> obj/kern/driver/console.o obj/kern/driver/intr.o
> obj/kern/driver/picirq.o obj/kern/trap/trap.o
> obj/kern/trap/trapentry.o obj/kern/trap/vectors.o obj/kern/mm/pmm.o 
> obj/libs/printfmt.o obj/libs/string.o

    # -------------------------------------------------------------------
    
    # create bootblock
    bootfiles = $(call listf_cc,boot)
    $(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))
    
    bootblock = $(call totarget,bootblock)
    
    $(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
    	@echo + ld $@
    	$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
    	@$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
    	@$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
    	@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)
    
    $(call create_target,bootblock)
    
创建bootblock。@符号表示执行前不显示命令。

> ld -m    elf_i386 -nostdlib -N -e start -Ttext 0x7C00
> obj/boot/bootasm.o obj/boot/bootmain.o -o obj/bootblock.o

    # -------------------------------------------------------------------
    
    # create 'sign' tools
    $(call add_files_host,tools/sign.c,sign,sign)
    $(call create_target_host,sign,sign)

设定了最后两位为0xAA55！

    # -------------------------------------------------------------------
    
    # create ucore.img
    UCOREIMG	:= $(call totarget,ucore.img)
    
    $(UCOREIMG): $(kernel) $(bootblock)
    	$(V)dd if=/dev/zero of=$@ count=10000
    	$(V)dd if=$(bootblock) of=$@ conv=notrunc
    	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc
    
    $(call create_target,ucore.img)

调用totarget函数。生成一个有10000个块的文件，每个块默认512字节，用0填充；再把bootblock中的内容写到第一个块；最后从第二个块开始写kernel中的内容。

    # >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>
    
    $(call finish_all)
    
    IGNORE_ALLDEPS	= clean \
    				  dist-clean \
    				  grade \
    				  touch \
    				  print-.+ \
    				  handin
    
    ifeq ($(call match,$(MAKECMDGOALS),$(IGNORE_ALLDEPS)),0)
    -include $(ALLDEPS)
    endif
    
在做完上述工作后的收尾，`-include`意味着在include时不用理会错误信息。match调用上文定义的match函数来匹配。

    # files for grade script
    
    TARGETS: $(TARGETS)
    
    .DEFAULT_GOAL := TARGETS
    
    .PHONY: qemu qemu-nox debug debug-nox
    qemu-mon: $(UCOREIMG)
    	$(V)$(QEMU)  -no-reboot -monitor stdio -hda $< -serial null
    qemu: $(UCOREIMG)
    	$(V)$(QEMU) -no-reboot -parallel stdio -hda $< -serial null
    log: $(UCOREIMG)
    	$(V)$(QEMU) -no-reboot -d int,cpu_reset  -D q.log -parallel stdio -hda $< -serial null
    qemu-nox: $(UCOREIMG)
    	$(V)$(QEMU)   -no-reboot -serial mon:stdio -hda $< -nographic
    TERMINAL        :=gnome-terminal

设置qemu选项。

    debug: $(UCOREIMG)
    	$(V)$(QEMU) -S -s -parallel stdio -hda $< -serial null &
    	$(V)sleep 2
    	$(V)$(TERMINAL) -e "gdb -q -tui -x tools/gdbinit"
    	
    debug-nox: $(UCOREIMG)
    	$(V)$(QEMU) -S -s -serial mon:stdio -hda $< -nographic &
    	$(V)sleep 2
    	$(V)$(TERMINAL) -e "gdb -q -x tools/gdbinit"
    
设置两种debug模式。

    .PHONY: grade touch
    
    GRADE_GDB_IN	:= .gdb.in
    GRADE_QEMU_OUT	:= .qemu.out
    HANDIN			:= proj$(PROJ)-handin.tar.gz
    
    TOUCH_FILES		:= kern/trap/trap.c
    
    MAKEOPTS		:= --quiet --no-print-directory
    
    grade:
    	$(V)$(MAKE) $(MAKEOPTS) clean
    	$(V)$(SH) tools/grade.sh
    	
运行grade批处理，进行成绩评定。    

    touch:
    	$(V)$(foreach f,$(TOUCH_FILES),$(TOUCH) $(f))
    
    print-%:
    	@echo $($(shell echo $(patsubst print-%,%,$@) | $(TR) [a-z] [A-Z]))
    
    .PHONY: clean dist-clean handin packall tags
    clean:
    	$(V)$(RM) $(GRADE_GDB_IN) $(GRADE_QEMU_OUT) cscope* tags
    	-$(RM) -r $(OBJDIR) $(BINDIR)
    
    dist-clean: clean
    	-$(RM) $(HANDIN)
    
    handin: packall
    	@echo Please visit http://learn.tsinghua.edu.cn and upload $(HANDIN). Thanks!
    
    packall: clean
    	@$(RM) -f $(HANDIN)
    	@tar -czf $(HANDIN) `find . -type f -o -type d | grep -v '^\.*$$' | grep -vF '$(HANDIN)'`
    
    tags:
    	@echo TAGS ALL
    	$(V)rm -f cscope.files cscope.in.out cscope.out cscope.po.out tags
    	$(V)find . -type f -name "*.[chS]" >cscope.files
    	$(V)cscope -bq 
    	$(V)ctags -L cscope.files

以上都是一些提交时的函数。

###符合规范的硬盘主引导扇区的特征

大小为512字节。在sign.c中可以发现，它还必须以0xAA55作为结尾。

  练习二：使用qemu执行并调试lab1中的软件

###从CPU加电后执行的第一条指令开始，单步跟踪BIOS的执行。

1.修改 lab1/tools/gdbinit，增加
>set architecture i8086

2.在lab1的目录中执行`make debug`
3.在gdb的界面中执行`si`，即可单步跟踪
4.可通过`x /ni $pc`来查看接下来n条汇编指令。

###在初始化位置0x7c00设置实地址断点,测试断点正常
在gdbinit结尾加上三行：
>b *0x7c00 //设置断点于内存0x7c00处
>continue //继续运行至断点
>x /2i $pc //显示当前两条指令

可以看到，gdb调试窗口出现了以下信息：

    Breakpoint 1, kern_init () at kern/init/init.c:17
    Breakpoint 2 at 0x7c00
    
    Breakpoint 2, 0x00007c00 in ?? ()
    => 0x7c00:      cli    
       0x7c01:      cld    

断点正常工作。
###从0x7c00开始跟踪代码运行,将单步跟踪反汇编得到的代码与bootasm.S和 bootblock.asm进行比较
在makefile的第220行进行修改，改为

    $(V)$(QEMU) -d in_asm -D q.log -S -s -parallel stdio -hda $< -serial null &

这样就能将汇编代码记录在q.log中了。

查看q.log，可以看到从0x7c00开始：

    ----------------
    IN: 
    0x00007c00:  cli    
    0x00007c01:  cld    
    0x00007c02:  xor    %ax,%ax
    0x00007c04:  mov    %ax,%ds
    0x00007c06:  mov    %ax,%es
    0x00007c08:  mov    %ax,%ss
    
    ----------------
    IN: 
    0x00007c0a:  in     $0x64,%al
    
    ----------------
    IN: 
    0x00007c0c:  test   $0x2,%al
    0x00007c0e:  jne    0x7c0a
    
    ----------------
    IN: 
    0x00007c10:  mov    $0xd1,%al
    0x00007c12:  out    %al,$0x64
    0x00007c14:  in     $0x64,%al
    0x00007c16:  test   $0x2,%al
    0x00007c18:  jne    0x7c14
    
    ----------------
    IN: 
    0x00007c1a:  mov    $0xdf,%al
    0x00007c1c:  out    %al,$0x60
    0x00007c1e:  lgdtw  0x7c6c
    0x00007c23:  mov    %cr0,%eax
    0x00007c26:  or     $0x1,%eax
    0x00007c2a:  mov    %eax,%cr0
    
    ----------------
    IN: 
    0x00007c2d:  ljmp   $0x8,$0x7c32
    
    ----------------
    IN: 
    0x00007c32:  mov    $0x10,%ax
    0x00007c36:  mov    %eax,%ds
    
    ----------------
    IN: 
    0x00007c38:  mov    %eax,%es
    
    ----------------
    IN: 
    0x00007c3a:  mov    %eax,%fs
    0x00007c3c:  mov    %eax,%gs
    0x00007c3e:  mov    %eax,%ss
    
    ----------------
    IN: 
    0x00007c40:  mov    $0x0,%ebp
    
    ----------------
    IN: 
    0x00007c45:  mov    $0x7c00,%esp
    0x00007c4a:  call   0x7cd1

与bootasm.S和bootblock.S相比较，将标记符号替换后，是一样的。
###自己找一个bootloader或内核中的代码位置，设置断点并进行测试

将断点设在0x7c4a

     Breakpoint 2, 0x00007c45 in ?? ()
     (gdb) x /2i
     0x7c4a:      call   0x7ccf
     0x7c4d:      add    %al,(%bx,%si)

  练习三：分析bootloader进入保护模式的过程

1.从0x7c00进入后，初始化一些变量

    code16                                             # Assemble for 16-bit mode
        cli                                             # Disable interrupts
        cld                                             # String operations increment
    
        # Set up the important data segment registers (DS, ES, SS).
        xorw %ax, %ax                                   # Segment number zero
        movw %ax, %ds                                   # -> Data Segment
        movw %ax, %es                                   # -> Extra Segment
        movw %ax, %ss                                   # -> Stack Segment

2.开启A20模式，此时全部的32条地址线均可使用，可以访问4G的内存空间。

    seta20.1:
        inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
        testb $0x2, %al
        jnz seta20.1
    
        movb $0xd1, %al                                 # 0xd1 -> port 0x64
        outb %al, $0x64                                 # 0xd1 means: write data to 8042's P2 port
    
    seta20.2:
        inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
        testb $0x2, %al
        jnz seta20.2
    
        movb $0xdf, %al                                 # 0xdf -> port 0x60
        outb %al, $0x60                                 # 0xdf = 11011111, means set P2's A20 bit(the 1 bit) to 1

3.初始化gdt表，载入已有的gdt表。

    lgdt gdtdesc
4.进入保护模式，将CR0_PE_ON置为1即可。

	movl %cr0, %eax
    orl $CR0_PE_ON, %eax
    movl %eax, %cr0
5.长跳转，更新CS基址
  
    ljmp $PROT_MODE_CSEG, $protcseg
    .code32
	protcseg:

6.初始化段寄存器

    movw $PROT_MODE_DSEG, %ax
    movw %ax, %ds
    movw %ax, %es
    movw %ax, %fs
    movw %ax, %gs
    movw %ax, %ss
7.建立堆栈
	

    movl $0x0, %ebp
    movl $start, %esp
8.调用主函数

    call bootmain

  练习四：分析bootloader加载ELF格式的OS的过程

- bootloader如何读取硬盘扇区？
bootloader由`readsect`函数来读取扇区的。具体的情况，在注释中已经解释得比较清楚了。它把第secno个扇区读入dst去。

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
    
        // read a sector， dword default
        insl(0x1F0, dst, SECTSIZE / 4);
    }

实际上中间有一层`readseg`包装了`readsect`，可读入任意长的字符串。

    static void
    readseg(uintptr_t va, uint32_t count, uint32_t offset) {
        uintptr_t end_va = va + count;
    
        // round down to sector boundary
        va -= offset % SECTSIZE;
    
        // translate from bytes to sectors; kernel starts at sector 1
        uint32_t secno = (offset / SECTSIZE) + 1;
    
        // If this is too slow, we could read lots of sectors at a time.
        // We'd write more to memory than asked, but it doesn't matter --
        // we load in increasing order.
        for (; va < end_va; va += SECTSIZE, secno ++) {
            readsect((void *)va, secno);
        }
    }
    
- bootloader是如何加载ELF格式的OS？

看入口的bootmain函数，在读入扇区之后，要根据头部`e_magic`值判断是不是ELF格式。将其头部保存的一个地址描述表加载进内存。最后根据头部最后的入口信息，找到Kernel的入口。

    void
    bootmain(void) {
        // read the 1st page off disk
        readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);
    
        // is this a valid ELF?
        if (ELFHDR->e_magic != ELF_MAGIC) {
            goto bad;
        }
    
        struct proghdr *ph, *eph;
    
        // load each program segment (ignores ph flags)
        ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);
        eph = ph + ELFHDR->e_phnum;
        for (; ph < eph; ph ++) {
            readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
        }
    
        // call the entry point from the ELF header
        // note: does not return
        ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();
    
    bad:
        outw(0x8A00, 0x8A00);
        outw(0x8A00, 0x8E00);
    
        /* do nothing */
        while (1);
    }

  练习五：实现函数调用堆栈跟踪函数

我的实现如下：

    uint32_t ebp = read_ebp();
    uint32_t eip = read_eip();
    int i, j;
    //if it is bottom or over stack depth limit, stop
    for(i = 0; ebp != 0 && i < STACKFRAME_DEPTH; i++)
    {
    	cprintf("ebp:0x%08x eip:0x%08x args:", ebp, eip);
    	uint32_t *base = (uint32_t *)ebp +2;
    	for(j = 0; j < 4; j++)
	    	cprintf("0x%08x ",base[j]);
    	cprintf("\n");
    	print_debuginfo(eip-1);
    	eip = ((uint32_t *)ebp)[1];
    	ebp = ((uint32_t *)ebp)[0];
    }
首先获得当前eip和ebp的值，然后遍历直到ebp为0，或者超出了栈的大小限制。每次遍历，首先输出ebp和eip，然后输出ebp所在地址+2后的四个参数。随后调用print_debuginfo去找到当前所在的文件和行数。

最后一行的参数意思：
>ebp:0x00007bf8 eip:0x00007d68 args:0xc031fcfa 0xc08ed88e 0x64e4d08e 0xfa7502a8

此时ebp为0x7bf8，是因为bootloader从0x7c00开始，当call bootmain的时候，进行压栈，于是bootmain的ebp为0x7bf8。

##练习六：完善中断初始化和处理

- 中断向量表中一个表项占多少字节？其中哪几位代表中断处理代码的入口？
中断向量表（IDT）中一个表项占8字节，其中2-3字节是段选择子，0-1字节和6-7字节合起来是位移。他们一起决定了入口位置。
- 完善kern/trap/trap.c中对中断向量表进行初始化的函数idt_init
见代码。基本思路：对每个idt进行初始化，最后装载这个idt。
- 编程完善trap.c中的中断处理函数trap
这个真的很简单，取出`clock.h`中的ticks，然后每当它等于100时就输出一次即可。
