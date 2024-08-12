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

定义了qemu中UART串口设备的起始地址

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

**types.h**

```C
#ifndef __TYPES_H__
#define __TYPES_H__

typedef unsigned char uint8_t;
typedef unsigned short uint16_t;
typedef unsigned int  uint32_t;
typedef unsigned long long uint64_t;

#endif /* __TYPES_H__ */
```

**uart.c**

uart驱动程序

```C
#include "types.h"
#include "platform.h"

/*
 * The UART control registers are memory-mapped at address UART0. 
 * This macro returns the address of one of the registers.
 */
#define UART_REG(reg) ((volatile uint8_t *)(UART0 + reg))

/*
 * Reference
 * [1]: TECHNICAL DATA ON 16550, http://byterunner.com/16550.html
 */

/*
 * UART control registers map. see [1] "PROGRAMMING TABLE"
 * note some are reused by multiple functions
 * 0 (write mode): THR/DLL
 * 1 (write mode): IER/DLM
 */
#define RHR 0	// Receive Holding Register (read mode)
#define THR 0	// Transmit Holding Register (write mode)
#define DLL 0	// LSB of Divisor Latch (write mode)
#define IER 1	// Interrupt Enable Register (write mode)
#define DLM 1	// MSB of Divisor Latch (write mode)
#define FCR 2	// FIFO Control Register (write mode)
#define ISR 2	// Interrupt Status Register (read mode)
#define LCR 3	// Line Control Register
#define MCR 4	// Modem Control Register
#define LSR 5	// Line Status Register
#define MSR 6	// Modem Status Register
#define SPR 7	// ScratchPad Register

/*
 * POWER UP DEFAULTS
 * IER = 0: TX/RX holding register interrupts are both disabled
 * ISR = 1: no interrupt penting
 * LCR = 0
 * MCR = 0
 * LSR = 60 HEX
 * MSR = BITS 0-3 = 0, BITS 4-7 = inputs
 * FCR = 0
 * TX = High
 * OP1 = High
 * OP2 = High
 * RTS = High
 * DTR = High
 * RXRDY = High
 * TXRDY = Low
 * INT = Low
 */

/*
 * LINE STATUS REGISTER (LSR)
 * LSR BIT 0:
 * 0 = no data in receive holding register or FIFO.
 * 1 = data has been receive and saved in the receive holding register or FIFO.
 * ......
 * LSR BIT 5:
 * 0 = transmit holding register is full. 16550 will not accept any data for transmission.
 * 1 = transmitter hold register (or FIFO) is empty. CPU can load the next character.
 * ......
 */
#define LSR_RX_READY (1 << 0)
#define LSR_TX_IDLE  (1 << 5)

#define uart_read_reg(reg) (*(UART_REG(reg)))
#define uart_write_reg(reg, v) (*(UART_REG(reg)) = (v))

void uart_init()
{
	/* disable interrupts. */
	uart_write_reg(IER, 0x00);

	/*
	 * Setting baud rate. Just a demo here if we care about the divisor,
	 * but for our purpose [QEMU-virt], this doesn't really do anything.
	 *
	 * Notice that the divisor register DLL (divisor latch least) and DLM (divisor
	 * latch most) have the same base address as the receiver/transmitter and the
	 * interrupt enable register. To change what the base address points to, we
	 * open the "divisor latch" by writing 1 into the Divisor Latch Access Bit
	 * (DLAB), which is bit index 7 of the Line Control Register (LCR).
	 *
	 * Regarding the baud rate value, see [1] "BAUD RATE GENERATOR PROGRAMMING TABLE".
	 * We use 38.4K when 1.8432 MHZ crystal, so the corresponding value is 3.
	 * And due to the divisor register is two bytes (16 bits), so we need to
	 * split the value of 3(0x0003) into two bytes, DLL stores the low byte,
	 * DLM stores the high byte.
	 */
	uint8_t lcr = uart_read_reg(LCR);
	uart_write_reg(LCR, lcr | (1 << 7));
	uart_write_reg(DLL, 0x03);
	uart_write_reg(DLM, 0x00);

	/*
	 * Continue setting the asynchronous data communication format.
	 * - number of the word length: 8 bits
	 * - number of stop bits：1 bit when word length is 8 bits
	 * - no parity
	 * - no break control
	 * - disabled baud latch
	 */
	lcr = 0;
	uart_write_reg(LCR, lcr | (3 << 0));
}

int uart_putc(char ch)
{
	while ((uart_read_reg(LSR) & LSR_TX_IDLE) == 0);
	return uart_write_reg(THR, ch);
}

void uart_puts(char *s)
{
	while (*s) {
		uart_putc(*s++);
	}
}
```

**kernel.c**

```C
extern void uart_init(void);
extern void uart_puts(char *s);

void start_kernel(void)
{
	uart_init();
	uart_puts("Hello, RVOS!\n");

	while (1) {}; // stop here!
}
```

### 代码整理

添加一个printf函数，为接下来的章节作准备

**os.h**

rvos头文件

```C
#ifndef __OS_H__
#define __OS_H__

#include "types.h"
#include "platform.h"

#include <stddef.h>
#include <stdarg.h>

/* uart */
extern int uart_putc(char ch);
extern void uart_puts(char *s);

/* printf */
extern int  printf(const char* s, ...);
extern void panic(char *s);

#endif /* __OS_H__ */
```

**printf.c**

```C
#include "os.h"

/*
 * ref: https://github.com/cccriscv/mini-riscv-os/blob/master/05-Preemptive/lib.c
 */

static int _vsnprintf(char * out, size_t n, const char* s, va_list vl)
{
	int format = 0;
	int longarg = 0;
	size_t pos = 0;
	for (; *s; s++) {
		if (format) {
			switch(*s) {
			case 'l': {
				longarg = 1;
				break;
			}
			case 'p': {
				longarg = 1;
				if (out && pos < n) {
					out[pos] = '0';
				}
				pos++;
				if (out && pos < n) {
					out[pos] = 'x';
				}
				pos++;
			}
			case 'x': {
				long num = longarg ? va_arg(vl, long) : va_arg(vl, int);
				int hexdigits = 2*(longarg ? sizeof(long) : sizeof(int))-1;
				for(int i = hexdigits; i >= 0; i--) {
					int d = (num >> (4*i)) & 0xF;
					if (out && pos < n) {
						out[pos] = (d < 10 ? '0'+d : 'a'+d-10);
					}
					pos++;
				}
				longarg = 0;
				format = 0;
				break;
			}
			case 'd': {
				long num = longarg ? va_arg(vl, long) : va_arg(vl, int);
				if (num < 0) {
					num = -num;
					if (out && pos < n) {
						out[pos] = '-';
					}
					pos++;
				}
				long digits = 1;
				for (long nn = num; nn /= 10; digits++);
				for (int i = digits-1; i >= 0; i--) {
					if (out && pos + i < n) {
						out[pos + i] = '0' + (num % 10);
					}
					num /= 10;
				}
				pos += digits;
				longarg = 0;
				format = 0;
				break;
			}
			case 's': {
				const char* s2 = va_arg(vl, const char*);
				while (*s2) {
					if (out && pos < n) {
						out[pos] = *s2;
					}
					pos++;
					s2++;
				}
				longarg = 0;
				format = 0;
				break;
			}
			case 'c': {
				if (out && pos < n) {
					out[pos] = (char)va_arg(vl,int);
				}
				pos++;
				longarg = 0;
				format = 0;
				break;
			}
			default:
				break;
			}
		} else if (*s == '%') {
			format = 1;
		} else {
			if (out && pos < n) {
				out[pos] = *s;
			}
			pos++;
		}
    	}
	if (out && pos < n) {
		out[pos] = 0;
	} else if (out && n) {
		out[n-1] = 0;
	}
	return pos;
}

static char out_buf[1000]; // buffer for _vprintf()

static int _vprintf(const char* s, va_list vl)
{
	int res = _vsnprintf(NULL, -1, s, vl);
	if (res+1 >= sizeof(out_buf)) {
		uart_puts("error: output string size overflow\n");
		while(1) {}
	}
	_vsnprintf(out_buf, res + 1, s, vl);
	uart_puts(out_buf);
	return res;
}

int printf(const char* s, ...)
{
	int res = 0;
	va_list vl;
	va_start(vl, s);
	res = _vprintf(s, vl);
	va_end(vl);
	return res;
}

void panic(char *s)
{
	printf("panic: ");
	printf(s);
	printf("\n");
	while(1){};
}
```

## 02-memanagement

内存管理主要分为三类
1:自动管理内存-栈
2:静态内存-全局变量和局部静态变量
3:动态管理内存-堆

本节主要实现内存按页动态分配和释放

**types.h**

```C
#ifndef __TYPES_H__
#define __TYPES_H__

typedef unsigned char uint8_t;
typedef unsigned short uint16_t;
typedef unsigned int  uint32_t;
typedef unsigned long long uint64_t;

typedef uint32_t ptr_t;

#endif /* __TYPES_H__ */
```

**os.ld**

链接器脚本

定义了128M的ram，起始地址位于0x80000000

定义了几个段区，由下往上依次是：

.text
.rodata
.data
.bss

根据链接器脚本开始链接，将所有代码段放在.text
注意.data段最好按照4k对齐，因为分页系统中定义一页大小就是4k，提高内存访问效率



```C
/*
 * rvos.ld
 * Linker script for outputting to RVOS
 */

/*
 * https://sourceware.org/binutils/docs/ld/Miscellaneous-Commands.html
 * OUTPUT_ARCH command specifies a particular output machine architecture.
 * "riscv" is the name of the architecture for both 64-bit and 32-bit
 * RISC-V target. We will further refine this by using -march
 * and -mabi when calling gcc.
 */
OUTPUT_ARCH( "riscv" )

/*
 * https://sourceware.org/binutils/docs/ld/Entry-Point.html
 * ENTRY command is used to set the "entry point", which is the first instruction
 * to execute in a program.
 * The argument of ENTRY command is a symbol name, here is "_start" which is
 * defined in start.S.
 */
ENTRY( _start )

/*
 * https://sourceware.org/binutils/docs/ld/MEMORY.html
 * The MEMORY command describes the location and size of blocks of memory in
 * the target.
 * The syntax for MEMORY is:
 * MEMORY
 * {
 *     name [(attr)] : ORIGIN = origin, LENGTH = len
 *     ......
 * }
 * Each line defines a memory region.
 * Each memory region must have a distinct name within the MEMORY command. Here
 * we only define one region named as "ram".
 * The "attr" string is an optional list of attributes that specify whether to
 * use a particular memory region for an input section which is not explicitly
 * mapped in the linker script. Here we assign 'w' (writeable), 'x' (executable),
 * and 'a' (allocatable). We use '!' to invert 'r' (read-only) and
 * 'i' (initialized).
 * The "ORIGIN" is used to set the start address of the memory region. Here we
 * place it right at the beginning of 0x8000_0000 because this is where the
 * QEMU-virt machine will start executing.
 * Finally LENGTH = 128M tells the linker that we have 128 megabyte of RAM.
 * The linker will double check this to make sure everything can fit.
 */
MEMORY
{
	ram   (wxa!ri) : ORIGIN = 0x80000000, LENGTH = 128M
}

/*
 * https://sourceware.org/binutils/docs/ld/SECTIONS.html
 * The SECTIONS command tells the linker how to map input sections into output
 * sections, and how to place the output sections in memory.
 * The format of the SECTIONS command is:
 * SECTIONS
 * {
 *     sections-command
 *     sections-command
 *     ......
 * }
 *
 * Each sections-command may of be one of the following:
 * (1) an ENTRY command
 * (2) a symbol assignment
 * (3) an output section description
 * (4) an overlay description
 * We here only demo (2) & (3).
 *
 * We use PROVIDE command to define symbols.
 * https://sourceware.org/binutils/docs/ld/PROVIDE.html
 * The PROVIDE keyword may be used to define a symbol.
 * The syntax is PROVIDE(symbol = expression).
 * Such symbols as "_text_start", "_text_end" ... will be used in mem.S.
 * Notice the period '.' tells the linker to set symbol(e.g. _text_start) to
 * the CURRENT location ('.' = current memory location). This current memory
 * location moves as we add things.
 */
SECTIONS
{
	/*
	 * We are going to layout all text sections in .text output section,
	 * starting with .text. The asterisk("*") in front of the
	 * parentheses means to match the .text section of ANY object file.
	 */
	.text : {
		PROVIDE(_text_start = .);
		*(.text .text.*)
		PROVIDE(_text_end = .);
	} >ram

	.rodata : {
		PROVIDE(_rodata_start = .);
		*(.rodata .rodata.*)
		PROVIDE(_rodata_end = .);
	} >ram

	.data : {
		/*
		 * . = ALIGN(4096) tells the linker to align the current memory
		 * location to 4096 bytes. This will insert padding bytes until
		 * current location becomes aligned on 4096-byte boundary.
		 * This is because our paging system's resolution is 4,096 bytes.
		 */
		. = ALIGN(4096);
		PROVIDE(_data_start = .);
		/*
		 * sdata and data are essentially the same thing. We do not need
		 * to distinguish sdata from data.
		 */
		*(.sdata .sdata.*)
		*(.data .data.*)
		PROVIDE(_data_end = .);
	} >ram

	.bss :{
		/*
		 * https://sourceware.org/binutils/docs/ld/Input-Section-Common.html
		 * In most cases, common symbols in input files will be placed
		 * in the ‘.bss’ section in the output file.
		 */
		PROVIDE(_bss_start = .);
		*(.sbss .sbss.*)
		*(.bss .bss.*)
		*(COMMON)
		PROVIDE(_bss_end = .);
	} >ram

	PROVIDE(_memory_start = ORIGIN(ram));
	PROVIDE(_memory_end = ORIGIN(ram) + LENGTH(ram));

	PROVIDE(_heap_start = _bss_end);
	PROVIDE(_heap_size = _memory_end - _heap_start);
}
```

**mem.S**

映射链接脚本中定义的符号：
堆内存起始地址和大小
指令部分起始地址和终止地址
数据部分起始地址和终止地址


```C
#define SIZE_PTR .word

.section .rodata
.global HEAP_START
HEAP_START: SIZE_PTR _heap_start

.global HEAP_SIZE
HEAP_SIZE: SIZE_PTR _heap_size

.global TEXT_START
TEXT_START: SIZE_PTR _text_start

.global TEXT_END
TEXT_END: SIZE_PTR _text_end

.global DATA_START
DATA_START: SIZE_PTR _data_start

.global DATA_END
DATA_END: SIZE_PTR _data_end

.global RODATA_START
RODATA_START: SIZE_PTR _rodata_start

.global RODATA_END
RODATA_END: SIZE_PTR _rodata_end

.global BSS_START
BSS_START: SIZE_PTR _bss_start

.global BSS_END
BSS_END: SIZE_PTR _bss_end
```


下面的stacks标签后的未初始化的地址编译汇编链接后位于.bss段

**start.s**

```C
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

	# Set all bytes in the BSS section to zero.
	la	a0, _bss_start
	la	a1, _bss_end
	bgeu	a0, a1, 2f
1:
	sw	zero, (a0)
	addi	a0, a0, 4
	bltu	a0, a1, 1b
2:
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


**page.c**

页数据结构

堆的前8页用于存放页的状态，取个名字就叫页状态区吧，页状态区每个Byte表示页分配区中一个页的状态，每个Byte第0位bit表示是否已分配，第1位bit表示是否是已分配的内存块中的最后一页，一页是4kB，8页是32kB，一共可以表示32k页(32k\*4k=128M)，最多可以支持128M的堆内存

```C
/*
 * Page Descriptor 
 * flags:
 * - bit 0: flag if this page is taken(allocated)
 * - bit 1: flag if this page is the last page of the memory block allocated
 */
struct Page {
	uint8_t flags;
};
```



以下主要实现堆内存初始化
主要做的逻辑是：将页状态区清0，这样表示页分配区所有的页都是未分配的状态；以及求出页分配区的起始地址和终止地址

```C
void page_init()
{
	/* 
	 * We reserved 8 Page (8 x 4096) to hold the Page structures.
	 * It should be enough to manage at most 128 MB (8 x 4096 x 4096) 
	 */
	_num_pages = (HEAP_SIZE / PAGE_SIZE) - 8;
	printf("HEAP_START = %p, HEAP_SIZE = 0x%lx, num of pages = %d\n", HEAP_START, HEAP_SIZE, _num_pages);
	
	struct Page *page = (struct Page *)HEAP_START;
	for (int i = 0; i < _num_pages; i++) {
		_clear(page);
		page++;	
	}

	_alloc_start = _align_page(HEAP_START + 8 * PAGE_SIZE);
	_alloc_end = _alloc_start + (PAGE_SIZE * _num_pages);

	printf("TEXT:   %p -> %p\n", TEXT_START, TEXT_END);
	printf("RODATA: %p -> %p\n", RODATA_START, RODATA_END);
	printf("DATA:   %p -> %p\n", DATA_START, DATA_END);
	printf("BSS:    %p -> %p\n", BSS_START, BSS_END);
	printf("HEAP:   %p -> %p\n", _alloc_start, _alloc_end);
}
```


分配页分配区的页

input需要分配的页数，分配连续的页，返回连续的页的起始地址

主要逻辑：
使用了非常基本的线性搜索，遍历页状态区，找到连续的符合要求的页数

```C
/*
 * Allocate a memory block which is composed of contiguous physical pages
 * - npages: the number of PAGE_SIZE pages to allocate
 */
void *page_alloc(int npages)
{
	/* Note we are searching the page descriptor bitmaps. */
	int found = 0;
	struct Page *page_i = (struct Page *)HEAP_START;
	for (int i = 0; i <= (_num_pages - npages); i++) {
		if (_is_free(page_i)) {
			found = 1;
			/* 
			 * meet a free page, continue to check if following
			 * (npages - 1) pages are also unallocated.
			 */
			struct Page *page_j = page_i + 1;
			for (int j = i + 1; j < (i + npages); j++) {
				if (!_is_free(page_j)) {
					found = 0;
					break;
				}
				page_j++;
			}
			/*
			 * get a memory block which is good enough for us,
			 * take housekeeping, then return the actual start
			 * address of the first page of this memory block
			 */
			if (found) {
				struct Page *page_k = page_i;
				for (int k = i; k < (i + npages); k++) {
					_set_flag(page_k, PAGE_TAKEN);
					page_k++;
				}
				page_k--;
				_set_flag(page_k, PAGE_LAST);
				return (void *)(_alloc_start + i * PAGE_SIZE);
			}
		}
		page_i++;
	}
	return NULL;
}
```

释放页分配区的页
主要逻辑：找到待释放页所在的块中的首页地址，并释放

```C
/*
 * Free the memory block
 * - p: start address of the memory block
 */
void page_free(void *p)
{
	/*
	 * Assert (TBD) if p is invalid
	 */
	if (!p || (ptr_t)p >= _alloc_end) {
		return;
	}
	/* get the first page descriptor of this memory block */
	struct Page *page = (struct Page *)HEAP_START;
	page += ((ptr_t)p - _alloc_start)/ PAGE_SIZE;
	/* loop and clear all the page descriptors of the memory block */
	while (!_is_free(page)) {
		if (_is_last(page)) {
			_clear(page);
			break;
		} else {
			_clear(page);
			page++;;
		}
	}
}
```

总览：

```C
#include "os.h"

/*
 * Following global vars are defined in mem.S
 */
extern ptr_t TEXT_START;
extern ptr_t TEXT_END;
extern ptr_t DATA_START;
extern ptr_t DATA_END;
extern ptr_t RODATA_START;
extern ptr_t RODATA_END;
extern ptr_t BSS_START;
extern ptr_t BSS_END;
extern ptr_t HEAP_START;
extern ptr_t HEAP_SIZE;

/*
 * _alloc_start points to the actual start address of heap pool
 * _alloc_end points to the actual end address of heap pool
 * _num_pages holds the actual max number of pages we can allocate.
 */
static ptr_t _alloc_start = 0;
static ptr_t _alloc_end = 0;
static uint32_t _num_pages = 0;

#define PAGE_SIZE 4096
#define PAGE_ORDER 12

#define PAGE_TAKEN (uint8_t)(1 << 0)
#define PAGE_LAST  (uint8_t)(1 << 1)

/*
 * Page Descriptor 
 * flags:
 * - bit 0: flag if this page is taken(allocated)
 * - bit 1: flag if this page is the last page of the memory block allocated
 */
struct Page {
	uint8_t flags;
};

static inline void _clear(struct Page *page)
{
	page->flags = 0;
}

static inline int _is_free(struct Page *page)
{
	if (page->flags & PAGE_TAKEN) {
		return 0;
	} else {
		return 1;
	}
}

static inline void _set_flag(struct Page *page, uint8_t flags)
{
	page->flags |= flags;
}

static inline int _is_last(struct Page *page)
{
	if (page->flags & PAGE_LAST) {
		return 1;
	} else {
		return 0;
	}
}

/*
 * align the address to the border of page(4K)
 */
static inline ptr_t _align_page(ptr_t address)
{
	ptr_t order = (1 << PAGE_ORDER) - 1;
	return (address + order) & (~order);
}

void page_init()
{
	/* 
	 * We reserved 8 Page (8 x 4096) to hold the Page structures.
	 * It should be enough to manage at most 128 MB (8 x 4096 x 4096) 
	 */
	_num_pages = (HEAP_SIZE / PAGE_SIZE) - 8;
	printf("HEAP_START = %p, HEAP_SIZE = 0x%lx, num of pages = %d\n", HEAP_START, HEAP_SIZE, _num_pages);
	
	struct Page *page = (struct Page *)HEAP_START;
	for (int i = 0; i < _num_pages; i++) {
		_clear(page);
		page++;	
	}

	_alloc_start = _align_page(HEAP_START + 8 * PAGE_SIZE);
	_alloc_end = _alloc_start + (PAGE_SIZE * _num_pages);

	printf("TEXT:   %p -> %p\n", TEXT_START, TEXT_END);
	printf("RODATA: %p -> %p\n", RODATA_START, RODATA_END);
	printf("DATA:   %p -> %p\n", DATA_START, DATA_END);
	printf("BSS:    %p -> %p\n", BSS_START, BSS_END);
	printf("HEAP:   %p -> %p\n", _alloc_start, _alloc_end);
}

/*
 * Allocate a memory block which is composed of contiguous physical pages
 * - npages: the number of PAGE_SIZE pages to allocate
 */
void *page_alloc(int npages)
{
	/* Note we are searching the page descriptor bitmaps. */
	int found = 0;
	struct Page *page_i = (struct Page *)HEAP_START;
	for (int i = 0; i <= (_num_pages - npages); i++) {
		if (_is_free(page_i)) {
			found = 1;
			/* 
			 * meet a free page, continue to check if following
			 * (npages - 1) pages are also unallocated.
			 */
			struct Page *page_j = page_i + 1;
			for (int j = i + 1; j < (i + npages); j++) {
				if (!_is_free(page_j)) {
					found = 0;
					break;
				}
				page_j++;
			}
			/*
			 * get a memory block which is good enough for us,
			 * take housekeeping, then return the actual start
			 * address of the first page of this memory block
			 */
			if (found) {
				struct Page *page_k = page_i;
				for (int k = i; k < (i + npages); k++) {
					_set_flag(page_k, PAGE_TAKEN);
					page_k++;
				}
				page_k--;
				_set_flag(page_k, PAGE_LAST);
				return (void *)(_alloc_start + i * PAGE_SIZE);
			}
		}
		page_i++;
	}
	return NULL;
}

/*
 * Free the memory block
 * - p: start address of the memory block
 */
void page_free(void *p)
{
	/*
	 * Assert (TBD) if p is invalid
	 */
	if (!p || (ptr_t)p >= _alloc_end) {
		return;
	}
	/* get the first page descriptor of this memory block */
	struct Page *page = (struct Page *)HEAP_START;
	page += ((ptr_t)p - _alloc_start)/ PAGE_SIZE;
	/* loop and clear all the page descriptors of the memory block */
	while (!_is_free(page)) {
		if (_is_last(page)) {
			_clear(page);
			break;
		} else {
			_clear(page);
			page++;;
		}
	}
}

void page_test()
{
	void *p = page_alloc(2);
	printf("p = %p\n", p);
	//page_free(p);

	void *p2 = page_alloc(7);
	printf("p2 = %p\n", p2);
	page_free(p2);

	void *p3 = page_alloc(4);
	printf("p3 = %p\n", p3);
}
```

运行以上page_test():

```Shell
zhangdongdong:02-memanagement/ (main✗) $ make run                                      [20:13:21]
Press Ctrl-A and then X to exit QEMU
------------------------------------
Hello, RVOS!
HEAP_START = 800033f4, HEAP_SIZE = 07ffcc0c, num of pages = 32756
TEXT:   0x80000000 -> 0x80002d0c
RODATA: 0x80002d0c -> 0x80002e9b
DATA:   0x80003000 -> 0x80003000
BSS:    0x80003000 -> 0x800033f4
HEAP:   0x8000c000 -> 0x88000000
p = 0x8000c000
p2 = 0x8000e000
p3 = 0x8000e000
QEMU: Terminated
```


### 03-contextswitch

本节关于上下文切换，首先实现单任务的调度，下一节实现协作式多任务调度

**types.h**

```C
#ifndef __TYPES_H__
#define __TYPES_H__

typedef unsigned char uint8_t;
typedef unsigned short uint16_t;
typedef unsigned int  uint32_t;
typedef unsigned long long uint64_t;

/*
 * Register Width
 */
typedef uint32_t reg_t;
typedef uint32_t ptr_t;

#endif /* __TYPES_H__ */
```


**定义上下文**

数据结构如下：

每个任务的上下文就是存在寄存器中的内容，当切换任务运行时，需要保存当前任务的上下文内容，并且恢复将要运行的任务的上下文

```C
/* task management */
struct context {
	/* ignore x0 */
	reg_t ra;
	reg_t sp;
	reg_t gp;
	reg_t tp;
	reg_t t0;
	reg_t t1;
	reg_t t2;
	reg_t s0;
	reg_t s1;
	reg_t a0;
	reg_t a1;
	reg_t a2;
	reg_t a3;
	reg_t a4;
	reg_t a5;
	reg_t a6;
	reg_t a7;
	reg_t s2;
	reg_t s3;
	reg_t s4;
	reg_t s5;
	reg_t s6;
	reg_t s7;
	reg_t s8;
	reg_t s9;
	reg_t s10;
	reg_t s11;
	reg_t t3;
	reg_t t4;
	reg_t t5;
	reg_t t6;
};
```

**sched.c**

task_stack：为每个任务分配1024字节的栈空间（目前只有一个任务）

ctx_task：当前运行任务的上下文

sched_init：实现调度初始化，将0写入mscratch寄存器（用于标识是第一次调度），准备当前将要运行的任务的上下文，即sp指针指向一块分配的栈空间，ra指针指向将要运行任务的地址

scheule：开始任务调度，调用switch_to，把将要运行的任务的上下文地址放入a0寄存器

switch_to：首次调度，将a0寄存器中存的上下文地址中的上下文信息恢复到寄存器中，然后ret程序自动跳转到初始化时放在ra寄存器中的地址开始执行



```C
#include "os.h"

/* defined in entry.S */
extern void switch_to(struct context *next);

#define STACK_SIZE 1024
/*
 * In the standard RISC-V calling convention, the stack pointer sp
 * is always 16-byte aligned.
 */
uint8_t __attribute__((aligned(16))) task_stack[STACK_SIZE];
struct context ctx_task;

static void w_mscratch(reg_t x)
{
	asm volatile("csrw mscratch, %0" : : "r" (x));
}

void user_task0(void);
void sched_init()
{
	w_mscratch(0);

	ctx_task.sp = (reg_t) &task_stack[STACK_SIZE];
	ctx_task.ra = (reg_t) user_task0;
}

void schedule()
{
	struct context *next = &ctx_task;
	switch_to(next);
}

/*
 * a very rough implementaion, just to consume the cpu
 */
void task_delay(volatile int count)
{
	count *= 50000;
	while (count--);
}


void user_task0(void)
{
	uart_puts("Task 0: Created!\n");
	while (1) {
		uart_puts("Task 0: Running...\n");
		task_delay(1000);
	}
}
```

**entry.s**

其中定义了保存上下文(`reg_save`)和恢复上下文(`reg_restore`)的宏


```C
# Save all General-Purpose(GP) registers to context.
# struct context *base = &ctx_task;
# base->ra = ra;
# ......
# These GP registers to be saved don't include gp
# and tp, because they are not caller-saved or
# callee-saved. These two registers are often used
# for special purpose. For example, in RVOS, 'tp'
# (aka "thread pointer") is used to store hartid,
# which is a global value and would not be changed
# during context-switch.
.macro reg_save base
	sw ra, 0(\base)
	sw sp, 4(\base)
	sw t0, 16(\base)
	sw t1, 20(\base)
	sw t2, 24(\base)
	sw s0, 28(\base)
	sw s1, 32(\base)
	sw a0, 36(\base)
	sw a1, 40(\base)
	sw a2, 44(\base)
	sw a3, 48(\base)
	sw a4, 52(\base)
	sw a5, 56(\base)
	sw a6, 60(\base)
	sw a7, 64(\base)
	sw s2, 68(\base)
	sw s3, 72(\base)
	sw s4, 76(\base)
	sw s5, 80(\base)
	sw s6, 84(\base)
	sw s7, 88(\base)
	sw s8, 92(\base)
	sw s9, 96(\base)
	sw s10, 100(\base)
	sw s11, 104(\base)
	sw t3, 108(\base)
	sw t4, 112(\base)
	sw t5, 116(\base)
	# we don't save t6 here, due to we have used
	# it as base, we have to save t6 in an extra step
	# outside of reg_save
.endm

# restore all General-Purpose(GP) registers from the context
# except gp & tp.
# struct context *base = &ctx_task;
# ra = base->ra;
# ......
.macro reg_restore base
	lw ra, 0(\base)
	lw sp, 4(\base)
	lw t0, 16(\base)
	lw t1, 20(\base)
	lw t2, 24(\base)
	lw s0, 28(\base)
	lw s1, 32(\base)
	lw a0, 36(\base)
	lw a1, 40(\base)
	lw a2, 44(\base)
	lw a3, 48(\base)
	lw a4, 52(\base)
	lw a5, 56(\base)
	lw a6, 60(\base)
	lw a7, 64(\base)
	lw s2, 68(\base)
	lw s3, 72(\base)
	lw s4, 76(\base)
	lw s5, 80(\base)
	lw s6, 84(\base)
	lw s7, 88(\base)
	lw s8, 92(\base)
	lw s9, 96(\base)
	lw s10, 100(\base)
	lw s11, 104(\base)
	lw t3, 108(\base)
	lw t4, 112(\base)
	lw t5, 116(\base)
	lw t6, 120(\base)
.endm

# Something to note about save/restore:
# - We use mscratch to hold a pointer to context of current task
# - We use t6 as the 'base' for reg_save/reg_restore, because it is the
#   very bottom register (x31) and would not be overwritten during loading.
#   Note: CSRs(mscratch) can not be used as 'base' due to load/restore
#   instruction only accept general purpose registers.

.text

# void switch_to(struct context *next);
# a0: pointer to the context of the next task
.globl switch_to
.balign 4
switch_to:
	csrrw	t6, mscratch, t6	# swap t6 and mscratch
	beqz	t6, 1f			# Note: the first time switch_to() is
	                                # called, mscratch is initialized as zero
					# (in sched_init()), which makes t6 zero,
					# and that's the special case we have to
					# handle with t6
	reg_save t6			# save context of prev task

	# Save the actual t6 register, which we swapped into
	# mscratch
	mv	t5, t6		# t5 points to the context of current task
	csrr	t6, mscratch	# read t6 back from mscratch
	sw	t6, 120(t5)	# save t6 with t5 as base

1:
	# switch mscratch to point to the context of the next task
	csrw	mscratch, a0

	# Restore all GP registers
	# Use t6 to point to the context of the new task
	mv	t6, a0
	reg_restore t6

	# Do actual context switching.
	ret

.end
```


### 04-multitask

本节主要实现协作式多任务调度

只需要对上面单任务调度作出一点改变：
因为需要支持多任务，所以需要对以上数据结构作出改变，使用context数组来存放每个任务的上下文数据，对应每个任务的栈空间也用数组来定义

task_creat：
类似于初始化一个进程，输入一个任务，给其分配栈空间并设置ra寄存器为此任务的地址，这样ret后将进入此地址执行指令

任务创建好之后就开始调度，这里使用的是fifo队列的调度方式

```C
#include "os.h"

/* defined in entry.S */
extern void switch_to(struct context *next);

#define MAX_TASKS 10
#define STACK_SIZE 1024
/*
 * In the standard RISC-V calling convention, the stack pointer sp
 * is always 16-byte aligned.
 */
uint8_t __attribute__((aligned(16))) task_stack[MAX_TASKS][STACK_SIZE];
struct context ctx_tasks[MAX_TASKS];

/*
 * _top is used to mark the max available position of ctx_tasks
 * _current is used to point to the context of current task
 */
static int _top = 0;
static int _current = -1;

static void w_mscratch(reg_t x)
{
	asm volatile("csrw mscratch, %0" : : "r" (x));
}

void sched_init()
{
	w_mscratch(0);
}

/*
 * implment a simple cycle FIFO schedular
 */
void schedule()
{
	if (_top <= 0) {
		panic("Num of task should be greater than zero!");
		return;
	}

	_current = (_current + 1) % _top;
	struct context *next = &(ctx_tasks[_current]);
	switch_to(next);
}

/*
 * DESCRIPTION
 * 	Create a task.
 * 	- start_routin: task routine entry
 * RETURN VALUE
 * 	0: success
 * 	-1: if error occured
 */
int task_create(void (*start_routin)(void))
{
	if (_top < MAX_TASKS) {
		ctx_tasks[_top].sp = (reg_t) &task_stack[_top][STACK_SIZE];
		ctx_tasks[_top].ra = (reg_t) start_routin;
		_top++;
		return 0;
	} else {
		return -1;
	}
}

/*
 * DESCRIPTION
 * 	task_yield()  causes the calling task to relinquish the CPU and a new 
 * 	task gets to run.
 */
void task_yield()
{
	schedule();
}

/*
 * a very rough implementaion, just to consume the cpu
 */
void task_delay(volatile int count)
{
	count *= 50000;
	while (count--);
}
```

### 05-traps

异常控制流程：当发生中断或异常的时候，cpu跳转到中断处理程序进行异常处理，处理完毕后返回到发生中断的指令的位置，
或者返回到发生中断的指令的下一条指令的位置

自定义一个中断处理函数，当发生中断或异常的时候，跳转到此函数进行处理，处理完毕之后返回

只需要将中断处理函数的地址写进mtvec寄存器，当系统发生中断或异常，会跳转到此寄存器中的地址运行指令，
在中断处理程序中可以提前将中断处理完成之后的指令地址写入mepc寄存器，然后调用mret，cpu会架构mepc中的值复制到pc，
从而达到中断返回的效果

mepc：中断发生时，cpu将发生trap的指令地址写入此寄存器，中断返回时(mret)，cpu将此寄存器中的值写回pc
mtvec：存放中断处理程序的基地址，发生中断时，cpu将此地址写入pc
mcause：当发生中断时，cpu将中断或异常的类型信息写入此寄存器
mtval：发生exception时，cpu将相关附加信息写入此寄存器
mstatus：用于跟踪和控制cpu的当前操作状态，比如关闭和打开全局中断

操作系统进行trap初始化只需要将写好的中断处理程序的地址写入mtvec

如何写中断处理程序：对mcause进行switch-case，对不同的trap进行对应的处理

```C

.text

# interrupts and exceptions while in machine mode come here.
.globl trap_vector
# the trap vector base address must always be aligned on a 4-byte boundary
.balign 4
trap_vector:
	# save context(registers).
	csrrw	t6, mscratch, t6	# swap t6 and mscratch
	reg_save t6

	# Save the actual t6 register, which we swapped into
	# mscratch
	mv	t5, t6		# t5 points to the context of current task
	csrr	t6, mscratch	# read t6 back from mscratch
	sw	t6, 120(t5)	# save t6 with t5 as base

	# Restore the context pointer into mscratch
	csrw	mscratch, t5

	# call the C trap handler in trap.c
	csrr	a0, mepc
	csrr	a1, mcause
	call	trap_handler

	# trap_handler will return the return address via a0.
	csrw	mepc, a0

	# restore context(registers).
	csrr	t6, mscratch
	reg_restore t6

	# return to whatever we were doing before trap.
	mret
```

**trap.c**

```C
#include "os.h"

extern void trap_vector(void);

void trap_init()
{
	/*
	 * set the trap-vector base-address for machine-mode
	 */
	w_mtvec((reg_t)trap_vector);
}

reg_t trap_handler(reg_t epc, reg_t cause)
{
	reg_t return_pc = epc;
	reg_t cause_code = cause & 0xfff;
	
	if (cause & 0x80000000) {
		/* Asynchronous trap - interrupt */
		switch (cause_code) {
		case 3:
			uart_puts("software interruption!\n");
			break;
		case 7:
			uart_puts("timer interruption!\n");
			break;
		case 11:
			uart_puts("external interruption!\n");
			break;
		default:
			uart_puts("unknown async exception!\n");
			break;
		}
	} else {
        /* Asynchronous trap - exception */
        switch (cause_code) {
            case 7:
                uart_puts("store/AMO address misaligned\n");
                return_pc += 4;
                break;
        }
		/* Synchronous trap - exception */
//		printf("Sync exceptions!, code = %d\n", cause_code);
//		panic("OOPS! What can I do!");
//		return_pc += 4;
	}

	return return_pc;
}

void trap_test()
{
	/*
	 * Synchronous exception code = 7
	 * Store/AMO access fault
	 */
	*(int *)0x00000000 = 100;

	/*
	 * Synchronous exception code = 5
	 * Load access fault
	 */
	//int a = *(int *)0x00000000;

	uart_puts("Yeah! I'm return back from trap!\n");
}
```

### 06-external-interrupt

cpu有三个关于引脚用于接受三种中断信息，软件中断，定时器中断和外部中断

```C
/* Machine-mode Interrupt Enable */
#define MIE_MEIE (1 << 11) // external
#define MIE_MTIE (1 << 7)  // timer
#define MIE_MSIE (1 << 3)  // software
```

**plic.c**

实现平台级别中断控制器(Platform-Level Interrupt Controller)，可通过配置
每个中断源的优先级、使能、阈值等来统一控制所有的中断信息

plic中有许多寄存器，包括：
+ 64个优先级寄存器（每个中断源一个）
+ 2个Pending寄存器，每一个bit对应于一个中断源，一共有32*2=64个中断源，1-有中断，0-无中断
+ 16个Enable寄存器，每个hart2个，8个hart一共有16个，每个bit对应一个中断源，负责开关每个hart上的中断源
+ 8个Threshold寄存器，每个hart1个，用于设置中断优先级的阈值
+ 8个Claim/Complete寄存器，每个hart1个，读操作成功发生后会清除对应的Pending位，写操作完成后通知PLIC中断处理结束

以下配置了uart的中断信息（hart0）

uart外设的Priority寄存器
uart外设的Enable寄存器
uart外设的MTHRESHOLD寄存器
设置mie和mstatus寄存器，用于打开hart0的外部中断和全局中断

```C
#include "os.h"

void plic_init(void)
{
	int hart = r_tp();
  
	/* 
	 * Set priority for UART0.
	 *
	 * Each PLIC interrupt source can be assigned a priority by writing 
	 * to its 32-bit memory-mapped priority register.
	 * The QEMU-virt (the same as FU540-C000) supports 7 levels of priority. 
	 * A priority value of 0 is reserved to mean "never interrupt" and 
	 * effectively disables the interrupt. 
	 * Priority 1 is the lowest active priority, and priority 7 is the highest. 
	 * Ties between global interrupts of the same priority are broken by 
	 * the Interrupt ID; interrupts with the lowest ID have the highest 
	 * effective priority.
	 */
	*(uint32_t*)PLIC_PRIORITY(UART0_IRQ) = 1;
 
	/*
	 * Enable UART0
	 *
	 * Each global interrupt can be enabled by setting the corresponding 
	 * bit in the enables registers.
	 */
	*(uint32_t*)PLIC_MENABLE(hart, UART0_IRQ)= (1 << (UART0_IRQ % 32));

	/* 
	 * Set priority threshold for UART0.
	 *
	 * PLIC will mask all interrupts of a priority less than or equal to threshold.
	 * Maximum threshold is 7.
	 * For example, a threshold value of zero permits all interrupts with
	 * non-zero priority, whereas a value of 7 masks all interrupts.
	 * Notice, the threshold is global for PLIC, not for each interrupt source.
	 */
	*(uint32_t*)PLIC_MTHRESHOLD(hart) = 0;

	/* enable machine-mode external interrupts. */
	w_mie(r_mie() | MIE_MEIE);

	/* enable machine-mode global interrupts. */
	w_mstatus(r_mstatus() | MSTATUS_MIE);
}

/* 
 * DESCRIPTION:
 *	Query the PLIC what interrupt we should serve.
 *	Perform an interrupt claim by reading the claim register, which
 *	returns the ID of the highest-priority pending interrupt or zero if there 
 *	is no pending interrupt. 
 *	A successful claim also atomically clears the corresponding pending bit
 *	on the interrupt source.
 * RETURN VALUE:
 *	the ID of the highest-priority pending interrupt or zero if there 
 *	is no pending interrupt.
 */
int plic_claim(void)
{
	int hart = r_tp();
	int irq = *(uint32_t*)PLIC_MCLAIM(hart);
	return irq;
}

/* 
 * DESCRIPTION:
  *	Writing the interrupt ID it received from the claim (irq) to the 
 *	complete register would signal the PLIC we've served this IRQ. 
 *	The PLIC does not check whether the completion ID is the same as the 
 *	last claim ID for that target. If the completion ID does not match an 
 *	interrupt source that is currently enabled for the target, the completion
 *	is silently ignored.
 * RETURN VALUE: none
 */
void plic_complete(int irq)
{
	int hart = r_tp();
	*(uint32_t*)PLIC_MCOMPLETE(hart) = irq;
}
```

**trap.c**

当发生外部中断时，通过plic_claim()来获取当前cpu的发生的最高级别的中断源ID，进行相应的处理，
处理成功之后回清除对应的Pending位
调用plic_complete()，来通知plic当前中断已经处理完毕，可以继续处理后续的中断



```C
#include "os.h"

extern void trap_vector(void);
extern void uart_isr(void);

void trap_init()
{
	/*
	 * set the trap-vector base-address for machine-mode
	 */
	w_mtvec((reg_t)trap_vector);
}

void external_interrupt_handler()
{
	int irq = plic_claim();

	if (irq == UART0_IRQ){
      		uart_isr();
	} else if (irq) {
		printf("unexpected interrupt irq = %d\n", irq);
	}
	
	if (irq) {
		plic_complete(irq);
	}
}

reg_t trap_handler(reg_t epc, reg_t cause)
{
	reg_t return_pc = epc;
	reg_t cause_code = cause & 0xfff;
	
	if (cause & 0x80000000) {
		/* Asynchronous trap - interrupt */
		switch (cause_code) {
		case 3:
			uart_puts("software interruption!\n");
			break;
		case 7:
			uart_puts("timer interruption!\n");
			break;
		case 11:
			uart_puts("external interruption!\n");
			external_interrupt_handler();
			break;
		default:
			uart_puts("unknown async exception!\n");
			break;
		}
	} else {
		/* Synchronous trap - exception */
		printf("Sync exceptions!, code = %d\n", cause_code);
		panic("OOPS! What can I do!");
		//return_pc += 4;
	}

	return return_pc;
}

void trap_test()
{
	/*
	 * Synchronous exception code = 7
	 * Store/AMO access fault
	 */
	*(int *)0x00000000 = 100;

	/*
	 * Synchronous exception code = 5
	 * Load access fault
	 */
	//int a = *(int *)0x00000000;

	uart_puts("Yeah! I'm return back from trap!\n");
}
```

### 07-hwtimer

本节实现另一种中断方式，定时器中断，对应的码值是7，主要逻辑只是每次将CLINT_MTIMECMP寄存器加上一个固定值，这样每次固定的时钟都会产生一个中断信号

```C
reg_t trap_handler(reg_t epc, reg_t cause)
{
	reg_t return_pc = epc;
	reg_t cause_code = cause & MCAUSE_MASK_ECODE;
	
	if (cause & MCAUSE_MASK_INTERRUPT) {
		/* Asynchronous trap - interrupt */
		switch (cause_code) {
		case 3:
			uart_puts("software interruption!\n");
			break;
		case 7:
			uart_puts("timer interruption!\n");
			timer_handler();
			break;
		case 11:
			uart_puts("external interruption!\n");
			external_interrupt_handler();
			break;
		default:
			printf("Unknown async exception! Code = %ld\n", cause_code);
			break;
		}
	} else {
		/* Synchronous trap - exception */
		printf("Sync exceptions! Code = %ld\n", cause_code);
		panic("OOPS! What can I do!");
		//return_pc += 4;
	}

	return return_pc;
}
```

```C
#include "os.h"

/* interval ~= 1s */
#define TIMER_INTERVAL CLINT_TIMEBASE_FREQ

static uint32_t _tick = 0;

/* load timer interval(in ticks) for next timer interrupt.*/
void timer_load(int interval)
{
	/* each CPU has a separate source of timer interrupts. */
	int id = r_mhartid();
	
	*(uint64_t*)CLINT_MTIMECMP(id) = *(uint64_t*)CLINT_MTIME + interval;
}

void timer_init()
{
	/*
	 * On reset, mtime is cleared to zero, but the mtimecmp registers 
	 * are not reset. So we have to init the mtimecmp manually.
	 */
	timer_load(TIMER_INTERVAL);

	/* enable machine-mode timer interrupts. */
	w_mie(r_mie() | MIE_MTIE);

	/* enable machine-mode global interrupts. */
	w_mstatus(r_mstatus() | MSTATUS_MIE);
}

void timer_handler() 
{
	_tick++;
	printf("tick: %d\n", _tick);

	timer_load(TIMER_INTERVAL);
}
```

### 08-preemptive

本节利用定时器中断来实现抢占式任务调度，并且利用软件中断来兼容协作式任务调度

```C
.text

# interrupts and exceptions while in machine mode come here.
.globl trap_vector
# the trap vector base address must always be aligned on a 4-byte boundary
.balign 4
trap_vector:
	# save context(registers).
	csrrw	t6, mscratch, t6	# swap t6 and mscratch
	reg_save t6

	# Save the actual t6 register, which we swapped into
	# mscratch
	mv	t5, t6			# t5 points to the context of current task
	csrr	t6, mscratch		# read t6 back from mscratch
	STORE	t6, 30*SIZE_REG(t5)	# save t6 with t5 as base

	# save mepc to context of current task
	csrr	a0, mepc
	STORE	a0, 31*SIZE_REG(t5)

	# Restore the context pointer into mscratch
	csrw	mscratch, t5

	# call the C trap handler in trap.c
	csrr	a0, mepc
	csrr	a1, mcause
	call	trap_handler

	# trap_handler will return the return address via a0.
	csrw	mepc, a0

	# restore context(registers).
	csrr	t6, mscratch
	reg_restore t6

	# return to whatever we were doing before trap.
	mret

# void switch_to(struct context *next);
# a0: pointer to the context of the next task
.globl switch_to
.balign 4
switch_to:
	# switch mscratch to point to the context of the next task
	csrw	mscratch, a0
	# set mepc to the pc of the next task
	LOAD	a1, 31*SIZE_REG(a0)
	csrw	mepc, a1

	# Restore all GP registers
	# Use t6 to point to the context of the new task
	mv	t6, a0
	reg_restore t6

	# Do actual context switching.
	# Notice this will enable global interrupt
	mret

.end

```

```C
#include "os.h"

extern void schedule(void);

/* interval ~= 1s */
#define TIMER_INTERVAL CLINT_TIMEBASE_FREQ

static uint32_t _tick = 0;

/* load timer interval(in ticks) for next timer interrupt.*/
void timer_load(int interval)
{
	/* each CPU has a separate source of timer interrupts. */
	int id = r_mhartid();
	
	*(uint64_t*)CLINT_MTIMECMP(id) = *(uint64_t*)CLINT_MTIME + interval;
}

void timer_init()
{
	/*
	 * On reset, mtime is cleared to zero, but the mtimecmp registers 
	 * are not reset. So we have to init the mtimecmp manually.
	 */
	timer_load(TIMER_INTERVAL);

	/* enable machine-mode timer interrupts. */
	w_mie(r_mie() | MIE_MTIE);
}

void timer_handler() 
{
	_tick++;
	printf("tick: %d\n", _tick);

	timer_load(TIMER_INTERVAL);

	schedule();
}
```

```C
#include "os.h"

/* defined in entry.S */
extern void switch_to(struct context *next);

#define MAX_TASKS 10
#define STACK_SIZE 1024
/*
 * In the standard RISC-V calling convention, the stack pointer sp
 * is always 16-byte aligned.
 */
uint8_t __attribute__((aligned(16))) task_stack[MAX_TASKS][STACK_SIZE];
struct context ctx_tasks[MAX_TASKS];

/*
 * _top is used to mark the max available position of ctx_tasks
 * _current is used to point to the context of current task
 */
static int _top = 0;
static int _current = -1;

void sched_init()
{
	w_mscratch(0);

	/* enable machine-mode software interrupts. */
	w_mie(r_mie() | MIE_MSIE);
}

/*
 * implment a simple cycle FIFO schedular
 */
void schedule()
{
	if (_top <= 0) {
		panic("Num of task should be greater than zero!");
		return;
	}

	_current = (_current + 1) % _top;
	struct context *next = &(ctx_tasks[_current]);
	switch_to(next);
}

/*
 * DESCRIPTION
 * 	Create a task.
 * 	- start_routin: task routine entry
 * RETURN VALUE
 * 	0: success
 * 	-1: if error occured
 */
int task_create(void (*start_routin)(void))
{
	if (_top < MAX_TASKS) {
		ctx_tasks[_top].sp = (reg_t) &task_stack[_top][STACK_SIZE];
		ctx_tasks[_top].pc = (reg_t) start_routin;
		_top++;
		return 0;
	} else {
		return -1;
	}
}

/*
 * DESCRIPTION
 * 	task_yield()  causes the calling task to relinquish the CPU and a new 
 * 	task gets to run.
 */
void task_yield()
{
	/* trigger a machine-level software interrupt */
	int id = r_mhartid();
	*(uint32_t*)CLINT_MSIP(id) = 1;
}

/*
 * a very rough implementaion, just to consume the cpu
 */
void task_delay(volatile int count)
{
	count *= 50000;
	while (count--);
}
```

### 09-lock

