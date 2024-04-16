# RISC-V

一个通用的，简约的，稳定的，开源指令集架构

模块化的ISA，以RV32I为基础，运行一个完整的软件栈，且稳定不变；除此之外还有其他可选的指令集，比如：RV32M，RV32F，RV32D，或者以上三个指令集的合集RV32MFD

# RV32I 基础整数指令集

## RV32I 指令格式

一共有6种指令格式：
- R-typy：寄存器-寄存器操作
- I-type：短立即数和访存load操作
- S-type：访存store操作
- B-type：条件跳转
- U-type：长立即数
- J-type：无条件跳转

特点：
- 固定长度为32位
- 提供3个寄存器作为操作
- 每条指令中将要操作的寄存器在同一位置，因此在进行指令解码之前就可以访问寄存器
- 立即数总是符号扩展，且符号位位于最高位

rv32i指令分成以下几种：
- 算逻
- 访存
- 跳转

![[IMG_1970.jpeg]]

![[IMG_1971.jpeg]]


## RV32I 寄存器

32个寄存器

![[IMG_1972.jpeg]]


## RV32I 算逻

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


## RV32I 访存

lw
sw

lb
lbu 
lh 
lhu 

sb
sh

## RV32I 跳转

beq
bne
bgt 
blt 

betu 
bltu 


jal 
jalr 