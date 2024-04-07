# [实践]基于risc-v开发一个操作系统

## 00-Bootstrap

本节主要实现启动操作系统内核的启动程序，为了简化，目前只是跑单核，让第`0`个核心`cpu`跳到内核第一行代码开始运行，其他核心进入休眠状态

**platform.h**
定义硬件信息: 最多支持`8`核心的`cpu`

```C
#ifndef __PLATFORM_H__
#define __PLATFORM_H__

/*
 * QEMU RISC-V Virt machine with 16550a UART and VirtIO MMIO
 */

/* 
 * maximum number of CPUs
 * see https://github.com/qemu/qemu/blob/master/include/hw/riscv/virt.h
 * #define VIRT_CPUS_MAX 8
 */
#define MAXNUM_CPU 8

#endif /* __PLATFORM_H__ */
```

**start.S**
启动程序，每个核心运行启动程序代码时，先判断其是否是第`0`个核心，如果是则跳进内核代码，否则让其进入等待中断的休眠状态，达到单核心运行操作系统的目的

启动程序给系统内核分配了`8k`字节的栈空间，并且按照`16`字节对齐以符合`riscv`的调用约定，分配给每个核心的栈空间大小是`1024`字节

```shell
#include "platform.h"

	# size of each hart's stack is 1024 bytes
	.equ	STACK_SIZE, 1024

	.global	_start

	.text
_start:
	# park harts with id != 0
	csrr	t0, mhartid		# read current hart id
	mv	tp, t0			# keep CPU's hartid in its tp for later usage.
	bnez	t0, park		# if we're not on the hart 0
					# we park the hart
	# Setup stacks, the stack grows from bottom to top, so we put the
	# stack pointer to the very end of the stack range.
	slli	t0, t0, 10		# shift left the hart id by 1024
	la	sp, stacks + STACK_SIZE	# set the initial stack pointer
					# to the end of the first stack space
	add	sp, sp, t0		# move the current hart stack pointer
					# to its place in the stack space

	j	start_kernel		# hart 0 jump to c

park:
	wfi
	j	park

	# In the standard RISC-V calling convention, the stack pointer sp
	# is always 16-byte aligned.
.balign 16
stacks:
	.skip	STACK_SIZE * MAXNUM_CPU # allocate space for all the harts stacks

	.end				# End of file
```

**Kernel.c**

内核：暂时让其空转

```C
void start_kernel(void)
{
	while (1) {}; // stop here!
}
```

至此达到了让第`0`个核心`cpu`运行操作系统内核代码的目的

## 01-helloRVOS

本节主要配置串口设备以方便调试内核，最终在窗口中打印出字符


**platform.h**

```C 
#ifndef __PLATFORM_H__
#define __PLATFORM_H__

/*
 * QEMU RISC-V Virt machine with 16550a UART and VirtIO MMIO
 */

/*
 * maximum number of CPUs
 * see https://github.com/qemu/qemu/blob/master/include/hw/riscv/virt.h
 * #define VIRT_CPUS_MAX 8
 */
#define MAXNUM_CPU 8

/*
 * MemoryMap
 * see https://github.com/qemu/qemu/blob/master/hw/riscv/virt.c, virt_memmap[]
 * 0x00001000 -- boot ROM, provided by qemu
 * 0x02000000 -- CLINT
 * 0x0C000000 -- PLIC
 * 0x10000000 -- UART0
 * 0x10001000 -- virtio disk
 * 0x80000000 -- boot ROM jumps here in machine mode, where we load our kernel
 */

/* This machine puts UART registers here in physical memory. */
#define UART0 0x10000000L
#endif /* __PLATFORM_H__ */ 
```

