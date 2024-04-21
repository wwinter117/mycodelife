# RISC-V

一个通用的，简约的，稳定的，开源指令集架构

模块化的ISA，以RV32I为基础，运行一个完整的软件栈，且稳定不变；除此之外还有其他可选的指令集，比如：RV32M，RV32F，RV32D，或者以上三个指令集的合集RV32MFD

# 基础整数指令集

## 指令格式

rv32i一共有6种指令格式：
- R-type：寄存器-寄存器操作
- I-type：短立即数和访存load操作
- S-type：访存store操作
- B-type：条件跳转
- U-type：长立即数
- J-type：无条件跳转

rv32i特点：
- 固定长度为32位
- 提供3个寄存器作为操作
- 每条指令中将要操作的寄存器在同一位置，因此在进行指令解码之前就可以访问寄存器
- 立即数总是符号扩展，且符号位位于最高位

rv32i指令分成以下几种：
- 算逻
- 访存
- 跳转

![](IMG_1970.jpeg)
![](IMG_1971.jpeg)

## 寄存器

32个寄存器

![](IMG_1972.jpeg)


## 整数计算

算术(add,sub)、逻辑(and,or,xor)、移位(sll,srl,sra)

add
addi
sub

and
andi
or
ori
xor
xori

sll
slli
srl
srli
sra
srai

slt
slti 
sltu
sltiu

lui
auipc


## 访存

lw
sw

lb
lbu
lh
lhu

sb
sh

## 跳转

beq
bne
bgt
blt

betu
bltu

jal
jalr 

## 其他

csrrc
csrrs
csrrw
csrci
csrsi
csrwi

ecall
ebreak
fence

# RISC-V汇编

## 函数调用规范

从调用者和被调用者的角度来看

调用者：
+ 初始化栈指针sp
+ 准备入参，a0等寄存器赋值
+ 保存返回地址（如果是调用非叶函数），保存到ra寄存器

被调用者：
- 分配栈帧，sp指针下移
- 保存s0等寄存器（如果需要）
- 执行函数体
- 释放栈帧，sp指针上移
- ret


当然调用者也可能被其他函数调用，此时调用者需要分配栈帧，根据需要保存返回地址的同时保存s0等寄存器，无需初始化栈指针sp，执行完函数体后释放栈帧

例如下面示例：

\_start调用aa_bb，aa_bb调用square
aa_bb是调用者调用square，同时也是被调用者被\_start调用

```shell
# Calling Convention
# Demo how to write nested routines
#
# void _start()
# {
#     // calling nested routine
#     aa_bb(3, 4);
# }
#
# int aa_bb(int a, int b)
# {
#     return square(a) + square(b);
# }
#
# int square(int num)
# {
#     return num * num;
# }

	.text			# Define beginning of text section
	.global	_start		# Define entry _start

_start:
	la sp, stack_end	# prepare stack for calling functions

	# aa_bb(3, 4);
	li a0, 3
	li a1, 4
	call aa_bb

stop:
	j stop			# Infinite loop to stop execution

# int aa_bb(int a, int b)
# return a^2 + b^2
aa_bb:
	# prologue
	addi sp, sp, -16
	sw s0, 0(sp)
	sw s1, 4(sp)
	sw s2, 8(sp)
	sw ra, 12(sp)

	# cp and store the input params
	mv s0, a0
	mv s1, a1

	# sum will be stored in s2 and is initialized as zero
	li s2, 0

	mv a0, s0
	jal square
	add s2, s2, a0

	mv a0, s1
	jal square
	add s2, s2, a0

	mv a0, s2

	# epilogue
	lw s0, 0(sp)
	lw s1, 4(sp)
	lw s2, 8(sp)
	lw ra, 12(sp)
	addi sp, sp, 16
	ret

# int square(int num)
square:
	# prologue
	addi sp, sp, -8
	sw s0, 0(sp)
	sw s1, 4(sp)

	# `mul a0, a0, a0` should be fine,
	# programing as below just to demo we can contine use the stack
	mv s0, a0
	mul s1, s0, s0
	mv a0, s1

	# epilogue
	lw s0, 0(sp)
	lw s1, 4(sp)
	addi sp, sp, 8

	ret

	# add nop here just for demo in gdb
	nop

	# allocate stack space
stack_start:
	.rept 12
	.word 0
	.endr
stack_end:

	.end			# End of file
```

## 汇编器

主要做两件事：
- 翻译伪指令
- 根据汇编指令将汇编代码翻译成机器指令


![[IMG_1982.jpeg]]

![[IMG_1979.jpeg]]

![[IMG_1980.jpeg]]


## 链接器


调整对象文件的指令中程序和数据的地址，使之与下图规范相符

![[IMG_1981.jpeg]]


## 加载器


# 乘除法指令

RV32M

# 单精度和双精度浮点计算

RV32F RV32D

# 原子指令

RV32A

![[IMG_1983.jpeg]]

