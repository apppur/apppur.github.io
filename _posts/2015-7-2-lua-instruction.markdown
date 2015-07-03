---
layout: post
title:  lua instruction (Unfinished)
date:   2015-07-02 11:00
categories: coding
---

## 1. Introduction
本文讲解lua 5.3.1虚拟机指令。lua的指令具有固定的大小，缺省使用一个32bit无符号整型数据类型。当前lua 5.3.1使用4个指令类型和47个操作码(编号从0到47)。指令类型枚举为：iABC, iABx, iAsBx, iAx。每个操作码占用最初的6bits。指令可以有以下的域：

	'A' : 8 bits
	'B' : 9 bits
	'C' : 9 bits
	'Ax': 26 bits ('A', 'B', and 'C' together)
	'Bx': 18 bits ('B' and 'C' together)
	'sBx': signed Bx

字段 A、B 和 C 通常引用寄存器编码(我将使用术语“寄存器”，因为它与处理器的寄存器相似)。虽然算术操作中字段A是目标操作数，但这个规则并非也适用于其他指令。寄存器通常是指向当前栈帧中的索引，0号寄存器是栈底位置。与 Lua C API 不同的是负索引(从栈顶开始计数)是不支持的。某些指令需要指定栈顶，则索引被编码为特定的操作数(通常是0)。局部变量等价于当前栈中的某个寄存器，但是也有允许读/写全局变量和upvalue的操作码。对某些指定字段B和C的值可能为寄存器或常量池中已编码的编号。

## 2. 指令集摘要
| opcode | name 		| description 					|
| ------ |:------------:| ------------------------------|
| 0 	 | OP_MOVE 		| 在寄存器间拷贝值				|
| 1 	 | OP_LOADK 	| 把一个常量载入寄存器 			|
| 2 	 | OP_LOADKX 	| 								|
| 3 	 | OP_LOADBOOL 	| 把一个布尔值载入寄存器 		|
| 4 	 | OP_LOADNIL 	| 把nil载入一系列寄存器 		|
| 5 	 | OP_GETUPVAL 	| 把一个upvalue读入寄存器 		|
| 6 	 | OP_GETTABUP  | 								|
| 7 	 | OP_GETTABLE 	| 把一个表元素读入寄存器 		|
| 8  	 | OP_SETTABUP 	| 								|
| 9 	 | OP_SETUPVAL 	| 								|
| 10 	 | OP_SETTABLE 	| 								|
| 11 	 | OP_NEWTABLE 	| R(A) = {} (size = B, C)		|
| 12 	 | OP_SELF 		| 								|
| 13 	 | OP_ADD 		| +								|
| 14 	 | OP_SUB 		| -								|
| 15 	 | OP_MUL 		| *								|
| 16 	 | OP_MOD 		| %								|
| 17 	 | OP_POW 		| ^ 							|
| 18 	 | OP_DIV 		| / 							|
| 19 	 | OP_IDIV 		| // 							|
| 20 	 | OP_BAND 		| & 							|
| 21 	 | OP_BOR 		| | 							|
| 22 	 | OP_BXOR 		| ~ 							|
| 23 	 | OP_SHL 		| << 							|
| 24 	 | OP_SHR 		| >> 							|
| 25 	 | OP_UNM 		| - 							|
| 26 	 | OP_BNOT 		| ~ 							|
| 27 	 | OP_NOT 		| not 							|
| 28 	 | OP_LEN 		| length of R(B) 				|
| 29 	 | OP_CONCAT 	| .. 							|
| 30 	 | OP_JMP 		| 无条件跳转 					|
| 31 	 | OP_EQ 		| 相等测试 						|
| 32 	 | OP_LT 		| 小于测试 						|
| 33 	 | OP_LE 		| 小于或等于测试 				|
| 34 	 | OP_TEST 		| 布尔测试带条件跳转 			|
| 35 	 | OP_TESTSET 	| 布尔测试带条件跳转和赋值 		|
| 36 	 | OP_CALL 		| 调用闭包						|
| 37 	 | OP_TAILCALL 	| 执行尾调用 					|
| 38 	 | OP_RETURN 	| 函数返回 						|
| 39 	 | OP_FORLOOP 	| 迭代 for 循环					|
| 40 	 | OP_FORPREP 	| 初始化数字for循环 			|
| 41 	 | OP_TFORCALL 	| 								|
| 42 	 | OP_TFORLOOP 	| 迭代泛型for循环				|
| 43 	 | OP_SETLIST 	| 设置表的一系列元素		 	|
| 44 	 | OP_CLOSURE 	| 创建一个函数原型的闭包		|
| 45 	 | OP_VARARG 	| 把可变参数量赋给寄存器 		|
| 46 	 | OP_EXTRAARG 	| 								|

## 3. 详细说明：
OP_MOVE: A B | R(A) := R(B) 将寄存器B中的值拷贝到寄存器A中
lua
	local b
	local a = b
vm
	1	[1]	LOADNIL  	0 0
	2	[2]	MOVE     	1 0
	3	[2]	RETURN   	0 1

在编译过程中，Lua会把每个local变量都分配一个指定的寄存器。在运行期间，Lua使用local变量所对应的寄存器ID来操作local变量，而local变量名仅仅提供DEBUG信息。
如上：b被分配到register 0,a被分配到register 1。MOVE表示将b(register 0)的值赋给a(register 1)。

OP_LOADK: A Bx | R(A) := Kst(Bx) 将Bx表示的常量表中的常量值装载到寄存器A中。有些指令本身可以直接从常量表中索引操作数，比如数学操作指令，所以可以不依赖LOADK指令。
lua
	local a = 0x1111000
	local b = "hello world!"
vm
	1	[1]	LOADK    	0 -1	; 17895424
	2	[2]	LOADK    	1 -2	; "hello world!"
	3	[2]	RETURN   	0 1

OP_LOADKX: A | R(A) := Kst(extra arg) LOADKX是lua5.2新加入的指令。当需要生成LOADK指令时，如果需要索引的常量id超出了Bx所能表示的有效范围，那么就生成一个LOADKX指令，取代LOADK指令，并且接下来立即生成一个EXTRAARG指令，并且用Ax来存放这个id。

OP_LOADBOOL: A B C | R(A) = (Bool)B; if (C) pc++
lua
	local a = true
vm
	1	[1]	LOADBOOL 	0 1 0
	2	[1]	RETURN   	0 1
LOADBOOL将B所表示的boolean值装载到寄存器A中。B使用0和1分别代表false和true。C也表示一个boolean值，如果C为1，就跳过下一个指令。
C的作用比较特殊，下面通过lua中对逻辑和关系表达式处理来看一下C的具体作用：
lua
	local a = 1 < 2
vm
	1	[1]	LT       	1 -1 -2	; 1 2
	2	[1]	JMP      	0 1	; to 4
	3	[1]	LOADBOOL 	0 0 1
	4	[1]	LOADBOOL 	0 1 0
	5	[1]	RETURN   	0 1
lua生成了LT和JMP指令，另外再加上两个LOADBOOL对于a赋予不同的boolean值。LT指令本身并不产生一个boolean结果值，而是配合紧跟后面的JMP实现的true和false的不同跳转。如果LT评估为true，就继续执行也就是执行到JMP，然后跳转到4，对a赋予true；否则就跳过下一条指令到达第三行，对a赋予false,并且跳过下一个指令。所以上面的代码实际意思可以转化为：
lua
	local a;
	if 1 < 2 then
		a = true;
	else
		a = false;
	end
逻辑或者关系表达式之所以被设计成这个样子，主要是为if语句和循环语句所做的优化。不用将整个表达式估值成一个boolean值后再决定跳转路径，而是评估过程中就可以直接跳转，节省了很多指令。C的作用就是配合这种使用逻辑或关系表达式进行赋值的操作，他节省了后面必须跟的一个JMP指令。

OP_LOADNIL: A B | R(A), R(A+1), ... R(A+B) := nil
lua
	local a, b, c, d
vm
	1	[1]	LOADNIL  	0 3
	2	[1]	RETURN   	0 1
LOADNIL将使用A到B所表示范围的寄存器赋值成nil。用范围表示寄存器主要为了对上述情况进行优化。对于连续的local变量声明，使用一条LOADNIL指令就可以完成，而不需要分别进行赋值。
对于以下情况：
lua
	local a
	local b = 1
	local c
	local d = 1
vm
	1	[1]	LOADNIL  	0 0
	2	[2]	LOADK    	1 -1	; 1
	3	[3]	LOADNIL  	2 0
	4	[4]	LOADK    	3 -1	; 1
	5	[4]	RETURN   	0 1
在Lua5.2中，a和c不能被合并成一个LOADNIL指令。所以以上写法理论上会生成更多的指令，应该予以避免，而改写成
lua
	local a, c
	local b = 1
	local d = 1
vm
	1	[1]	LOADNIL  	0 1
	2	[3]	LOADK    	2 -1	; 1
	3	[4]	LOADK    	3 -1	; 1
	4	[4]	RETURN   	0 1