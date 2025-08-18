# 实验报告

## 练习1：理解通过make生成执行文件的过程。（要求在报告中写出对下述问题的回答）
1. 操作系统镜像文件ucore.img是如何一步一步生成的？(需要比较详细地解释Makefile中每一条相关命令和命令参数的含义，以及说明命令导致的结果)

ucore 由两部分组成<br>
第0个扇区是 bootloader<br>
第1个扇区以后的多个扇区是 kern<br>

```make
$(UCOREIMG): $(kernel) $(bootblock)
	$(V)dd if=/dev/zero of=$@ count=10000
	$(V)dd if=$(bootblock) of=$@ conv=notrunc
	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc

```
/dev/zero 是特殊设备，会生成无限的空字节

dd 是一个在 Unix/Linux 系统中用于复制和转换文件的命令行工具，功能非常强大，常用于处理磁盘镜像、备份数据、转换文件格式等场景。<br>
if=输入文件 <br>
of=输出文件 <br>
count=块数：指定复制的块数量（总大小 = bs × count）<br>
seek=块数：输出时跳过指定数量的块（从文件开头开始算）<br>
conv=转换方式：文件转换选项，常见值 <br>
1. notrunc：不截断输出文件（保留原有大小，只覆盖写入部分）<br>
2. sync：将每个输入块填充到 bs 大小，不足部分用空字节填充 <br>
3. fdatasync：写入后强制刷新数据到磁盘 <br>

### bootloader

```make
$(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
	@$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
	@$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
	@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)
```

```make
	$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
```
-N：设置文本段为可写（方便引导程序操作）<br>
-e start：指定程序入口点为start函数 <br>
-Ttext 0x7C00：指定代码段加载地址为 0x7C00（BIOS 会将 MBR 加载到这个地址执行 <br>

```make
	@$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
```
objdump -S：将目标文件反汇编，并混合源代码（方便调试） <br>
输出：反汇编文本文件（如obj/bootblock.asm） <br>

```make
	@$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
```
objcopy -O binary：将目标文件转换为纯二进制格式（去除 ELF 等格式信息）<br>
-S：不复制重定位和符号信息）<br>
输出：二进制文件 obj/bootblock.out <br>

```make
	@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)
```
sign是一个辅助工具，作用是： <br>
确保最终的bootblock大小恰好为 512 字节（MBR 要求） <br>
在最后两个字节添加0x55AA签名（BIOS 识别为可引导扇区的标志） <br>
输出：最终可引导的bootblock文件 <br>

### kern
```make
$(kernel): $(KOBJS)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
	@$(OBJDUMP) -S $@ > $(call asmfile,kernel)
	@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)
```

```make
	$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
```
-T tools/kernel.ld：指定链接脚本（kernel.ld），用于定义内核的内存布局（如代码段、数据段、堆、栈的地址分布）<br>
```make
	@$(OBJDUMP) -S $@ > $(call asmfile,kernel)
```
objdump -S：将内核文件反汇编，并尽可能关联源代码（方便调试时对照源码和汇编指令）<br>
输出：反汇编文本文件（如 asm/kernel.asm），包含内核的所有汇编指令<br>

```make
	@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)
```
objdump -t：显示内核文件中的符号表（包含函数名、变量名及其地址）<br>
$(SED) ...：通过 sed 命令过滤和清理输出：<br>
1,/SYMBOL TABLE/d：删除 SYMBOL TABLE 之前的内容<br>
s/ .* / /：简化符号表格式（保留地址和符号名）<br>
/^$$/d：删除空行<br>
输出：符号表文件（如 sym/kernel.sym），记录内核中所有符号的地址，用于调试时定位函数 / 变量位置<br>

### 链接脚本
```ld
/* Simple linker script for the JOS kernel.
   See the GNU ld 'info' manual ("info ld") to learn the syntax. */

OUTPUT_FORMAT("elf32-i386", "elf32-i386", "elf32-i386")
OUTPUT_ARCH(i386)
ENTRY(kern_init)

SECTIONS {
	/* Load the kernel at this address: "." means the current address */
	. = 0x100000;

	.text : {
		*(.text .stub .text.* .gnu.linkonce.t.*)
	}

	PROVIDE(etext = .);	/* Define the 'etext' symbol to this value */

	.rodata : {
		*(.rodata .rodata.* .gnu.linkonce.r.*)
	}

	/* Include debugging information in kernel memory */
	.stab : {
		PROVIDE(__STAB_BEGIN__ = .);
		*(.stab);
		PROVIDE(__STAB_END__ = .);
		BYTE(0)		/* Force the linker to allocate space
				   for this section */
	}

	.stabstr : {
		PROVIDE(__STABSTR_BEGIN__ = .);
		*(.stabstr);
		PROVIDE(__STABSTR_END__ = .);
		BYTE(0)		/* Force the linker to allocate space
				   for this section */
	}

	/* Adjust the address for the data segment to the next page */
	. = ALIGN(0x1000);

	/* The data segment */
	.data : {
		*(.data)
	}

	PROVIDE(edata = .);

	.bss : {
		*(.bss)
	}

	PROVIDE(end = .);

	/DISCARD/ : {
		*(.eh_frame .note.GNU-stack)
	}
}

```


### 链接flag
```make
LDFLAGS	:= -m $(shell $(LD) -V | grep elf_i386 2>/dev/null)
LDFLAGS	+= -nostdlib
```
$(LD) -V：让链接器（ld）输出其支持的目标格式和架构信息 <br>
grep elf_i386：从输出中筛选出 32 位 x86 架构对应的目标格式（通常是elf_i386） <br>
2>/dev/null：忽略错误输出（避免在不支持该架构的系统上显示错误） <br>
-m <格式>：告诉链接器生成指定格式的可执行文件（这里是elf_i386，即 32 位 x86 的 ELF 格式） <br>
最终这行会生成类似 LDFLAGS = -m elf_i386 的效果，确保链接器针对 32 位 x86 架构进行处理。 <br>
-nostdlib：告诉链接器不链接标准系统库（如 C 标准库libc） <br>
这在操作系统内核、引导程序等底层开发中是必需的，因为这些程序运行时没有操作系统提供的标准库支持，需要自己实现必要的基础功能（如内存管理、字符串操作等） <br>

### 编译flag
```make
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
```
- HOSTCC 和 HOSTCFLAGS：<br>
用于编译主机端工具（如辅助脚本、签名工具sign等）的编译器和选项：<br>
HOSTCC := gcc：使用 GCC 作为主机编译器<br>
HOSTCFLAGS := -g -Wall -O2：主机编译选项（-g调试信息，-Wall开启所有警告，-O2优化）<br>

-fno-builtin：禁用 GCC 内置函数（如printf的内置实现，确保使用自定义版本）<br>
-Wall：开启所有警告，增强代码健壮性<br>
-ggdb：生成 GDB 兼容的详细调试信息<br>
-m32：生成 32 位架构代码（适用于 x86 32 位内核）<br>
-gstabs：生成 stabs 格式的调试信息（某些调试工具依赖）<br>
-nostdinc：不使用标准系统头文件（内核开发需使用自定义头文件）<br>
$(DEFS)：附加的宏定义（如-DDEBUG等）<br>

```make
CFLAGS	+= $(shell $(CC) -fno-stack-protector -E -x c /dev/null >/dev/null 2>&1 && echo -fno-stack-protector)
```
-fno-stack-protector的作用是禁用编译器的栈保护机制（如 GCC 的 StackGuard）。在操作系统内核、引导程序等底层开发中，通常需要禁用此功能，原因是：<br>
栈保护机制依赖用户态库的实现，而内核运行时没有这些库<br>
会增加额外的代码和内存开销，对于资源受限的底层环境不适用<br>
可能导致内核在某些特殊场景下（如中断处理）出现异常<br>

核心判断逻辑：$(CC) -fno-stack-protector -E -x c /dev/null >/dev/null 2>&1<br>
$(CC)：当前使用的编译器（可能是 gcc 或 clang）<br>
-fno-stack-protector：要检测的编译选项（禁用栈保护机制）<br>
-E：只进行预处理，不编译、不链接<br>
-x c /dev/null：指定输入为 C 语言，输入文件为/dev/null（空文件）<br>
>/dev/null 2>&1：屏蔽所有输出（标准输出和错误输出）<br>
条件动作：&& echo -fno-stack-protector<br>
如果前面的编译器命令执行成功（返回 0），说明编译器支持-fno-stack-protector选项，就输出该选项<br>
如果执行失败（返回非 0），说明编译器不支持该选项，就不输出任何内容<br>

2. 一个被系统认为是符合规范的硬盘主引导扇区的特征是什么？<br>

大小恰好为 512 字节（MBR 要求）<br>
以0x55AA 结尾（BIOS 识别为可引导扇区的标志）<br>

## 练习2：使用qemu执行并调试lab1中的软件。（要求在报告中简要写出练习过程）
为了熟悉使用qemu和gdb进行的调试工作，我们进行如下的小练习：<br>

1. 从CPU加电后执行的第一条指令开始，单步跟踪BIOS的执行。<br>
```make
debug-mon1-nox: $(UCOREIMG)
	$(V)$(QEMU) -S -s -serial mon:stdio -hda $< -nographic &
	$(V)sleep 2
	$(V)$(TERMINAL) -e "gdb -q -x tools/gdbinit-mon1"
```
gdbinit-mon1
```gdb
# 连接到QEMU的GDB服务器（默认端口1234）
target remote localhost:1234
# 开启实模式调试支持（x86实模式地址解析）
set architecture i8086
# 在0xFFFF0设置断点（CPU加电后执行的第一条指令地址）
break *0xFFFF0

# 定义停止时自动显示当前指令的钩子
define hook-stop
# 显示当前及后续4条指令
  x/5i $pc
end

file bin/kernel
break *0x7c00
break kern_init
```

-S：QEMU 启动后立即暂停（不自动执行），等待调试器连接后再开始运行（核心调试选项）。<br>
-s： shorthand for -gdb tcp::1234，在 1234 端口开启 GDB 服务器，允许 GDB 远程连接。<br>
-serial mon:stdio：将虚拟机串口输出重定向到当前终端，并支持 QEMU 监控器（按Ctrl+A再按c切换）。<br>
-hda $<：将内核镜像文件（$(UCOREIMG)，$< 表示第一个依赖）作为虚拟机的第一个硬盘（主硬盘）。<br>
-nographic：无图形界面，纯命令行运行。<br><br>
暂停 2 秒，确保 QEMU 有足够时间启动并准备好 GDB 服务器，避免 GDB 连接时虚拟机尚未就绪。<br><br>
-q：安静模式启动 GDB。<br>
-x tools/gdbinit-mon1：执行 tools/gdbinit-mon1 脚本中的 GDB 命令<br>

2. 在初始化位置0x7c00设置实地址断点,测试断点正常。<br>
略<br>
3. 从0x7c00开始跟踪代码运行,将单步跟踪反汇编得到的代码与bootasm.S和 bootblock.asm进行比较。<br>
```make
$(V)$(QEMU) -S -s -parallel stdio -d in_asm -D $(BINDIR)/q.log -hda $< -serial null &
```
-d：启用 QEMU 的调试日志功能，后跟调试选项（多个选项用逗号分隔）。<br>
in_asm：指定日志类型为 “执行的汇编指令”，QEMU 会将 CPU 执行的每一条汇编指令（包括指令地址、机器码、汇编格式）记录到日志中<br>
-D：指定日志输出文件（全称-debugfile）。<br>

4. 自己找一个bootloader或内核中的代码位置，设置断点并进行测试。<br>
略<br>

## 练习3：分析bootloader进入保护模式的过程。（要求在报告中写出分析）

### 为何开启A20，以及如何开启A20
实模式下，x86 CPU 受限于设计兼容早期 8086 处理器，仅使用 20 根地址线（A0~A19），最大可访问内存为2^20 = 1MB。<br>
此时若地址超过 1MB（如0x100000），会发生 “地址回卷”（Wrap Around），即0x100000被当作0x00000处理。<br>
保护模式下，CPU 支持 32 根地址线，可访问 4GB 内存。<br>
若 A20 地址线未开启，第 21 位地址（A20）始终为 0，导致0x100000 ~ 0x10FFFF的内存访问错误映射到0x00000 ~ 0x00FFFF，无法正常使用 1MB 以上内存。<br>
因此，进入保护模式前必须开启 A20 地址线，确保 32 位地址能正确寻址。<br><br>

通过向8042芯片0x60端口写入0xdf开启<br>
### 如何初始化GDT表
编写 GDT 表数据并初始化gdt表
```
lgdt gdtdesc
```

### 如何使能和进入保护模式
开启保护模式（设置 CR0 的 PE 位）
```asm
movl %cr0, %eax
orl $CR0_PE_ON, %eax
movl %eax, %cr0
```
长跳转更新cs寄存器
```asm
ljmp $PROT_MODE_CSEG, $protcseg
```

## 练习4：分析bootloader加载ELF格式的OS的过程。（要求在报告中写出分析）

从磁盘读取八个扇区<br>
然后从判断是否是合法elf文件<br>
加载代码段和数据段到对应位置<br>
调用函数入口<br>
```sh
# 看elf头
readelf -h bin/kernel
# 看程序头
readelf -l bin/kernel
```
实际加载大小和位置和参考答案不同<br>

## 练习5：实现函数调用堆栈跟踪函数 （需要编程）

```c
void print_stackframe(void) {
  /* LAB1 YOUR CODE : STEP 1 */
  /* (1) call read_ebp() to get the value of ebp. the type is (uint32_t);
   * (2) call read_eip() to get the value of eip. the type is (uint32_t);
   * (3) from 0 .. STACKFRAME_DEPTH
   *    (3.1) printf value of ebp, eip
   *    (3.2) (uint32_t)calling arguments [0..4] = the contents in address
   * (unit32_t)ebp +2 [0..4] (3.3) cprintf("\n"); (3.4) call
   * print_debuginfo(eip-1) to print the C calling function name and line
   * number, etc. (3.5) popup a calling stackframe NOTICE: the calling
   * funciton's return addr eip  = ss:[ebp+4] the calling funciton's ebp =
   * ss:[ebp]
   */
  uint32_t ebp = read_ebp();
  uint32_t eip = read_eip();
  int i, j;
  for (i = 0; ebp != 0 && i < STACKFRAME_DEPTH; i++) {
    cprintf("ebp:0x%08x ", ebp);
    cprintf("eip:0x%08x ", eip);
    uint32_t next_ebp = *(uint32_t *)(ebp);
    uint32_t p_arg = ebp + 8;

    cprintf("args: ");
    for (j = 0; j < 4; j++) {
      uint32_t arg = *(uint32_t *)(p_arg);
      cprintf("0x%08x ", arg);
      p_arg += 4;
    }
    cprintf("\n ");
    print_debuginfo(eip - 1);

    eip = *(uint32_t *)(ebp + 4);
    ebp = next_ebp;
  }
}
```

## 练习6：完善中断初始化和处理 （需要编程）

### 中断描述符表（也可简称为保护模式下的中断向量表）中一个表项占多少字节？其中哪几位代表中断处理代码的入口？

中断描述符表中一个表项占8字节<br>
0-15位 是偏移的 0-15位<br>
16-31位 是gdt的段选择子<br>
48-63位 是偏移的 16-31位<br>

### 请编程完善kern/trap/trap.c中对中断向量表进行初始化的函数idt_init。在idt_init函数中，依次对所有中断入口进行初始化。使用mmu.h中的SETGATE宏，填充idt数组内容。每个中断的入口由tools/vectors.c生成，使用trap.c中声明的vectors数组即可。


### 请编程完善trap.c中的中断处理函数trap，在对时钟中断进行处理的部分填写trap函数中处理时钟中断的部分，使操作系统每遇到100次时钟中断后，调用print_ticks子程序，向屏幕上打印一行文字”100 ticks”。


