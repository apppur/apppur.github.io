---
layout: post
title:  lua instruction (Unfinished)
date:   2015-07-02 11:00
categories: coding
---

## 1. Introduction
本文讲解lua 5.3.1虚拟机指令。lua的指令具有固定的大小，缺省使用一个32bit无符号整型数据类型。当前lua 5.3.1使用4个指令类型和47个操作码(编号从0到46)。指令类型枚举为：iABC, iABx, iAsBx, iAx。每个操作码占用最初的6bits。指令可以有以下的域：

	'A' : 8 bits
	'B' : 9 bits
	'C' : 9 bits
	'Ax': 26 bits ('A', 'B', and 'C' together)
	'Bx': 18 bits ('B' and 'C' together)
	'sBx': signed Bx

字段 A、B 和 C 通常引用寄存器编码(我将使用术语“寄存器”，因为它与处理器的寄存器相似)。虽然算术操作中字段A是目标操作数，但这个规则并非也适用于其他指令。寄存器通常是指向当前栈帧中的索引，0号寄存器是栈底位置。与 Lua C API 不同的是负索引(从栈顶开始计数)是不支持的。某些指令需要指定栈顶，则索引被编码为特定的操作数(通常是0)。局部变量等价于当前栈中的某个寄存器，但是也有允许读/写全局变量和upvalue的操作码。对某些指定字段B和C的值可能为寄存器或常量池中已编码的编号。

## 2. 指令集摘要

| opcode | name         | description                    |
| ------ | ------------ | ------------------------------ |
| 0 	 | OP_MOVE 		| 在寄存器间拷贝值				 |
| 1 	 | OP_LOADK 	| 把一个常量载入寄存器 			 |
| 2 	 | OP_LOADKX 	| 								 |
| 3 	 | OP_LOADBOOL 	| 把一个布尔值载入寄存器 		 |
| 4 	 | OP_LOADNIL 	| 把nil载入一系列寄存器 		 |
| 5 	 | OP_GETUPVAL 	| 把一个upvalue读入寄存器 		 |
| 6 	 | OP_GETTABUP  | 								 |
| 7 	 | OP_GETTABLE 	| 把一个表元素读入寄存器 		 |
| 8  	 | OP_SETTABUP 	| 								 |
| 9 	 | OP_SETUPVAL 	| 								 |
| 10 	 | OP_SETTABLE 	| 								 |
| 11 	 | OP_NEWTABLE 	| R(A) = {} (size = B, C)		 |
| 12 	 | OP_SELF 		| 								 |
| 13 	 | OP_ADD 		| +								 |
| 14 	 | OP_SUB 		| -								 |
| 15 	 | OP_MUL 		| *								 |
| 16 	 | OP_MOD 		| %								 |
| 17 	 | OP_POW 		| ^ 							 |
| 18 	 | OP_DIV 		| / 							 |
| 19 	 | OP_IDIV 		| // 							 |
| 20 	 | OP_BAND 		| & 							 |
| 21 	 | OP_BOR 		| | 							 |
| 22 	 | OP_BXOR 		| ~ 							 |
| 23 	 | OP_SHL 		| << 							 |
| 24 	 | OP_SHR 		| >> 							 |
| 25 	 | OP_UNM 		| - 							 |
| 26 	 | OP_BNOT 		| ~ 							 |
| 27 	 | OP_NOT 		| not 							 |
| 28 	 | OP_LEN 		| length of R(B) 				 |
| 29 	 | OP_CONCAT 	| .. 							 |
| 30 	 | OP_JMP 		| 无条件跳转 					 |
| 31 	 | OP_EQ 		| 相等测试 						 |
| 32 	 | OP_LT 		| 小于测试 						 |
| 33 	 | OP_LE 		| 小于或等于测试 				 |
| 34 	 | OP_TEST 		| 布尔测试带条件跳转 			 |
| 35 	 | OP_TESTSET 	| 布尔测试带条件跳转和赋值 		 |
| 36 	 | OP_CALL 		| 调用闭包						 |
| 37 	 | OP_TAILCALL 	| 执行尾调用 					 |
| 38 	 | OP_RETURN 	| 函数返回 						 |
| 39 	 | OP_FORLOOP 	| 迭代 for 循环					 |
| 40 	 | OP_FORPREP 	| 初始化数字for循环 			 |
| 41 	 | OP_TFORCALL 	| 								 |
| 42 	 | OP_TFORLOOP 	| 迭代泛型for循环				 |
| 43 	 | OP_SETLIST 	| 设置表的一系列元素		 	 |
| 44 	 | OP_CLOSURE 	| 创建一个函数原型的闭包		 |
| 45 	 | OP_VARARG 	| 把可变参数量赋给寄存器 		 |
| 46 	 | OP_EXTRAARG 	| 								 |


## 3. 详细说明：
OP_MOVE: A B 	R(A) := R(B) 
将寄存器B中的值拷贝到寄存器A中

lua

	local b
	local a = b

vm

	1	[1]	LOADNIL  	0 0
	2	[2]	MOVE     	1 0
	3	[2]	RETURN   	0 1

在编译过程中，Lua会把每个local变量都分配一个指定的寄存器。在运行期间，Lua使用local变量所对应的寄存器ID来操作local变量，而local变量名仅仅提供DEBUG信息。
如上：b被分配到register 0,a被分配到register 1。MOVE表示将b(register 0)的值赋给a(register 1)。

OP_LOADK: A Bx 	R(A) := Kst(Bx)
将Bx表示的常量表中的常量值装载到寄存器A中。有些指令本身可以直接从常量表中索引操作数，比如数学操作指令，所以可以不依赖LOADK指令。

lua

	local a = 0x1111000
	local b = "hello world!"

vm

	1	[1]	LOADK    	0 -1	; 17895424
	2	[2]	LOADK    	1 -2	; "hello world!"
	3	[2]	RETURN   	0 1

OP_LOADKX: A 	R(A) := Kst(extra arg) LOADKX
是lua5.2新加入的指令。当需要生成LOADK指令时，如果需要索引的常量id超出了Bx所能表示的有效范围，那么就生成一个LOADKX指令，取代LOADK指令，并且接下来立即生成一个EXTRAARG指令，并且用Ax来存放这个id。

OP_LOADBOOL: A B C 	R(A) = (Bool)B; if (C) pc++

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

OP_LOADNIL: A B 	R(A), R(A+1), ... R(A+B) := nil

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

在Lua5.3.1中，a和c不能被合并成一个LOADNIL指令。所以以上写法理论上会生成更多的指令，应该予以避免，而改写成

lua

	local a, c
	local b = 1
	local d = 1

vm

	1	[1]	LOADNIL  	0 1
	2	[3]	LOADK    	2 -1	; 1
	3	[4]	LOADK    	3 -1	; 1
	4	[4]	RETURN   	0 1

编译期间访问变量a时，会按照以下的顺序决定变量a信息：

* a是当前函数的local变量

* a是外层函数的local变量，即a是当前函数的upvalue

* a是全局变量(note)

local变量本身就存在于当前的register中，所有的指令都可以直接使用它的id来访问。而对于upvalue，lua则有专门的指令负责获取和设置。

全局变量在lua5.1中也是使用专门的指令来操作，在lua5.2对这一点做了改变。lua5.2及以后版本没有对全局变量操作的指令，而是把全局表放在最外层函数的名字为“_ENV”的upvalue中。对于全局变量a，编译后为_ENV.a来进行访问。

* OP_GETUPVAL: A B 		R(A) := UpValue[B] 	GETUPVAL将B为索引的upvalue的值装载到A寄存器中。

* OP_SETUPVAL: A B 		UpValue[B] := R(A) 	SETUPVAL将A寄存器的值保存到B为索引的upvalue中。

* OP_GETTABUP: A B C 	R(A) := UpValue[B][RK(C)] 	GETTABUP将B为索引的upvalue当作一个table，并将C做为索引的寄存器或者常量当作key获取的值放入寄存器A。

* OP_SETTABUP: A B C 	UpValue[A][RK(B)] := RK(C) 	SETTABUP将A为索引的upvalue当作一个table，将C寄存器或者常量的值以B寄存器或常量为key，存入table

lua

	local u = 0
	function f()
    	local l
    	u = 1
    	l = u
    	g = 1
    	l = g
	end

vm

	main <main.lua:0,0> (4 instructions at 0x95c590)
	0+ params, 2 slots, 1 upvalue, 1 local, 2 constants, 1 function
		1	[1]	LOADK    	0 -1	; 0
		2	[8]	CLOSURE  	1 0	; 0x95c970
		3	[2]	SETTABUP 	0 -2 1	; _ENV "f"
		4	[8]	RETURN   	0 1

	function <main.lua:2,8> (7 instructions at 0x95c970)
	0 params, 2 slots, 2 upvalues, 1 local, 2 constants, 0 functions
		1	[3]	LOADNIL  	0 0
		2	[4]	LOADK    	1 -1	; 1
		3	[4]	SETUPVAL 	1 0	; u
		4	[5]	GETUPVAL 	0 0	; u
		5	[6]	SETTABUP 	1 -2 -1	; _ENV "g" 1
		6	[7]	GETTABUP 	0 1 -2	; _ENV "g"
		7	[8]	RETURN   	0 1

上述代码片断包括一个主函数和一个内嵌函数，根据变量规则，在内嵌函数中，l是local变量，u是upvalue，g既不是local，也不是upvalue，当作全局变量处理。
在内嵌函数，首先LOADNIL为local变量赋值，然后用LOADK和SETUPVAL组合，完成 u = 1。1是一个常量，存在于常量表中，而lua没有常量与upvalue的直接操作指令，所以需要先把常量1装在到临时寄存器1种，然后将寄存器1的值赋给upvalue 0，也就是u。GETUPVAL将upvalue u赋给local变量l。SETTABUP和GETTABUP就是前面提到的对全局变量的处理了。g=1被转化为_ENV.g=1。_ENV是系统预先设置在主函数中的upvalue，所以对于全局变量g的访问被转化成对upvalue[_ENV][g]的访问。SETTABUP将upvalue 1(_ENV代表的upvalue)作为一个table，将常量表2（常量"g"）作为key的值设置为常量表1（常量1）；GETTABUP则是将upvalue 1作为table，将常量表2为key的值赋给寄存器0（local l）。

OP_NEWTABLE: A B C R(A) := {} (size = B, C)

lua

	local t = {}

vm

	1	[1]	NEWTABLE 	0 0 0
	2	[1]	RETURN   	0 1

NEWTABLE在寄存器A处创建一个table对象。B和C分别用来存储这个table数组部分和hash部分的初始大小。初始大小是在编译期计算出来并生成到这个指令中的，目的是使接下来对table的初始化填充不会造成rehash而影响效率。B和C使用“floating point byte”的方法来表示成(eeeeexxx)的二进制形式，其实际值为(1xxx) * 2^(eeeee-1)。
上面代码生成一个空的table，放入local变量t，B和C参数都为0。

OP_SETLIST: A B C R(A)[(C-1)*FPF+i] := R(A+i), 1 <= i <= B
SETLIST用来配合NEWTABLE，初始化表的数组部分使用的。A为保存待设置表的寄存器，SETLIST要将A下面紧接着的寄存器列表(1--B)中的值逐个设置给表的数组部分。

lua

	local t = {1, 2, 3, 4, 5}

vm

	1	[1]	NEWTABLE 	0 5 0
	2	[1]	LOADK    	1 -1	; 1
	3	[1]	LOADK    	2 -2	; 2
	4	[1]	LOADK    	3 -3	; 3
	5	[1]	LOADK    	4 -4	; 4
	6	[1]	LOADK    	5 -5	; 5
	7	[1]	SETLIST  	0 5 1	; 1
	8	[1]	RETURN   	0 1

第1行先用NEWTABLE构建一个具有5个数组元素的表，放到寄存器0中；然后使用5个LOADK向下面3个寄存器装入常量；最后使用SETLIST设置表的1~5为寄存器1~寄存器5。

如果创建一个包含很多数组项元素的表，将数据放到寄存器时，就是会超出寄存器的范围。lua中的解决办法就是按照固定大小进行分批处理。每批的数量由在lopcodes.h中的LFIELDS_PER_FLUSH的宏控制，默认为50。因此，大量的数组元素就会按照50个一批，先将值设置到下面的寄存器，然后设置给对应的项。C代表的就是这一个调用SETLIST设置的是第几批。

lua
	local t = {
    	1, 2, 3, 4, 5, 6, 7, 8, 9, 0,
    	1, 2, 3, 4, 5, 6, 7, 8, 9, 0,
    	1, 2, 3, 4, 5, 6, 7, 8, 9, 0,
    	1, 2, 3, 4, 5, 6, 7, 8, 9, 0,
    	1, 2, 3, 4, 5, 6, 7, 8, 9, 0,
    	1, 2, 3, 4, 5,
	}

vm
	main <main.lua:0,0> (59 instructions at 0x112b590)
	0+ params, 51 slots, 1 upvalue, 1 local, 10 constants, 0 functions
		1	[1]	NEWTABLE 	0 30 0
		2	[2]	LOADK    	1 -1	; 1
		3	[2]	LOADK    	2 -2	; 2
		4	[2]	LOADK    	3 -3	; 3
		5	[2]	LOADK    	4 -4	; 4
		... ...
		49	[6]	LOADK    	48 -8	; 8
		50	[6]	LOADK    	49 -9	; 9
		51	[6]	LOADK    	50 -10	; 0
		52	[6]	SETLIST  	0 50 1	; 1
		53	[7]	LOADK    	1 -1	; 1
		54	[7]	LOADK    	2 -2	; 2
		55	[7]	LOADK    	3 -3	; 3
		56	[7]	LOADK    	4 -4	; 4
		57	[8]	LOADK    	5 -5	; 5
		58	[8]	SETLIST  	0 5 2	; 2
		59	[8]	RETURN   	0 1
如果数据量超出了C的表示范围，那么C会被设置为0，然后在SETLIST指令后面生成一个EXTRAARG指令，并用Ax来存储批次。与LOADKX的处理方法类似，来为处理超大数据服务的。

OP_GETTABL A B C R(A) := R(B)[RK(C)]
OP_SETTABL A B C R(A)[RK(B)] := RK(C)

GETTABLE使用C表示的key，将寄存器B中的表项值获取到寄存器A中。SETTABLE设置寄存器A的表的B项为C代表的值。

lua
	
	local t = {}
	t.k = 1
	local v = t.k

vm

	main <main.lua:0,0> (4 instructions at 0x24e3590)
	0+ params, 2 slots, 1 upvalue, 2 locals, 2 constants, 0 functions
		1	[1]	NEWTABLE 	0 0 0
		2	[2]	SETTABLE 	0 -1 -2	; "k" 1
		3	[3]	GETTABLE 	1 0 -1	; "k"
		4	[3]	RETURN   	0 1

OP_ADD A B C R(A) := RK(B) + RK(C)
OP_SUB A B C R(A) := RK(B) - RK(C)
OP_MUL A B C R(A) := RK(B) * RK(C)
OP_MOD A B C R(A) := RK(B) % RK(C)
OP_POW A B C R(A) := RK(B) ^ RK(C)
OP_DIV A B C R(A) := RK(B) / RK(C)
OP_IDIV A B C R(A) := RK(B) // RK(C)

lua

	local v = 8
	v = v + 2
	v = v - 2
	v = v * 2
	v = v % 2
	v = v ^ 2
	v = v / 2
	v = v // 2

vm
	main <main.lua:0,0> (9 instructions at 0x773590)
	0+ params, 2 slots, 1 upvalue, 1 local, 2 constants, 0 functions
		1	[1]	LOADK    	0 -1	; 8
		2	[2]	ADD      	0 0 -2	; - 2
		3	[3]	SUB      	0 0 -2	; - 2
		4	[4]	MUL      	0 0 -2	; - 2
		5	[5]	MOD      	0 0 -2
		6	[6]	POW      	0 0 -2	; - 2
		7	[7]	DIV      	0 0 -2	; - 2
		8	[8]	IDIV     	0 0 -2	; - 2
		9	[8]	RETURN   	0 1

OP_BAND A B C R(A) := RK(B) & RK(C)
OP_BOR A B C R(A) := RK(B) | RK(C)
OP_BXOR A B C R(A) := RK(B) ~ RK(C)
OP_BNOT A B R(A) := ~R(B)

lua

	local v = 7
	v = v & 8
	v = v | 8
	v = v ~ 8
	v = ~ v

vm

	main <main.lua:0,0> (6 instructions at 0x1d54590)
	0+ params, 2 slots, 1 upvalue, 1 local, 2 constants, 0 functions
		1	[1]	LOADK    	0 -1	; 7
		2	[2]	BAND     	0 0 -2	; - 8
		3	[3]	BOR      	0 0 -2	; - 8
		4	[4]	BXOR     	0 0 -2	; - 8
		5	[5]	BNOT     	0 0
		6	[5]	RETURN   	0 1

OP_SHL A B C R(A) := RK(B) << RK(C)
OP_SHR A B C R(A) := RK(B) >> RK(C)

lua

	local v = 7
	v = v << 4
	v = v >> 4

vm

	main <main.lua:0,0> (4 instructions at 0x18e4590)
	0+ params, 2 slots, 1 upvalue, 1 local, 2 constants, 0 functions
		1	[1]	LOADK    	0 -1	; 7
		2	[2]	SHL      	0 0 -2	; - 4
		3	[3]	SHR      	0 0 -2	; - 4
		4	[3]	RETURN   	0 1

OP_UNM A B R(A) := -R(B)
OP_NOT A B R(A) := not R(B)

lua
	
	local v = 7
	v = -v
	v = not v

vm

	1	[1]	LOADK    	0 -1	; 7
	2	[2]	UNM      	0 0
	3	[3]	NOT      	0 0
	4	[3]	RETURN   	0 1

OP_LEN A B R(A) := length of R(B)
LEN直接对应'#'操作符，返回B对象的长度，并保存到A中。

lua

	local v = #"hello world"

vm
	
	1	[1]	LOADK    	0 -1	; "hello world"
	2	[1]	LEN      	0 0
	3	[1]	RETURN   	0 1

OP_CONCAT A B C R(A) := R(B).. ... ..R(C)
字符串连接

lua
	
	local greed = "welcome to "
	local name = "applepurple"
	local v = greed .. name

vm 

	1	[1]	LOADK    	0 -1	; "welcome to "
	2	[2]	LOADK    	1 -2	; "applepurple"
	3	[3]	MOVE     	2 0
	4	[3]	MOVE     	3 1
	5	[3]	CONCAT   	2 2 3
	6	[3]	RETURN   	0 1