# 实验报告

## 练习1 
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

2. 一个被系统认为是符合规范的硬盘主引导扇区的特征是什么？

大小恰好为 512 字节（MBR 要求）<br>
以0x55AA 结尾（BIOS 识别为可引导扇区的标志）<br>