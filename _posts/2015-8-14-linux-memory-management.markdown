---
layout: post
title:  linux 内存管理
date:   2015-08-14 14:30
categories: coding
---

Linux操作系统采用虚拟内存管理技术，使得每个进程都有各自互不干涉的进程地址空间。用户所看到和接触到的都该虚拟地址，无法看到真实的物理内存地址。利用这种虚拟地址不但能起到保护操作系统的效果(用户不能直接访问物理内存)，而且更重要的是，用户程序可使用比实际物理内存更大的地址空间。

###1.32位操作系统进程具有独立的大小为4G的线性虚拟地址空间：

* 第一、4G的进程空间被人为的分为两部分--用户空间和内核空间。用户空间从0到3G(0xC0000000)。内核空间占据3G到4G。用户进程通常情况下只能访问用户空间的虚拟地址，不能访问内核空间虚拟地址。只有用户进程进行系统调用（代表用户进程在内核态执行）等时刻可以访问到内核空间。

* 第二、用户空间对应进程，所以每当进程切换，用户空间就会跟着变化；而内核空间是由内核负责映射，它并不会跟着进程改变，是固定的。内核空间地址有自己对应的页表（init_mm.pgd），用户进程各自有不同的页表

* 第三、每个进程的用户空间都是完全独立、互不相干的。


![32 bit memory layout](/images/mem32.png)	

###2.64位操作系统的进程虚拟地址布局如下：

![64 bit memory layout](/images/mem64.png)

##1.进程与内存

程序一般是指一些可运行的代码，它存放在磁盘上。进程是指运行状态中的程序，它占用一定数量的内存。对于任何一个普通进程来讲，它涉及到5种不同的数据段，如下：

* 代码段(text segment):代码段是用来存放可执行文件的操作指令，也就是说它是可执行程序在内存中的镜像。代码段需要防止在运行时被修改，所以只允许读取操作，禁止写入(修改)操作(read only)

* 数据段(data segment):数据段用来存放可执行文件中已初始化的全局变量，即静态分配的变量和全局变量。

* BBS段:BBS段包含了程序中未初始化的全局变量，在内存中BBS段全部置0.

* 堆(heap):堆是用于进程运行中被动态分配的内存段，大小不固定，可动态扩张与缩小。当进程调用malloc等函数分配内存时，新分配的内存被动态添加到堆上(堆扩张)；当利用free等函数释放内存时，被释放的内存从堆中清除(堆缩小)

* 栈(stack):栈是用户存放程序临时创建的局部变量，也就是说我们函数中定义的变量(但不包括static声明的变量，static表明在数据段中的变量)；还有在函数调用时，其参数也会被压入发起调用的进程栈中，在调用结束后函数本身和与之关联的局部变量会pop出栈。其特点是FILO

进程按照下面的布局来组织这些数据段：数据段、BBS和堆通常是被连续存储的，而代码段和栈往往会被独立存放。堆和栈两个区域一个向上"长"，一个向下"长"，相对而生。一般情况下你不必担心他们碰到一起，因为它们之间的间隔很大。

![memory layout](/images/memlayout.png)

	#include<stdio.h>
	#include<malloc.h>
	#include<unistd.h>
	int bss_var;
	int data_var0=1;
	int main(int argc,char **argv)
	{
    	printf("below are addresses of types of process's mem\n");
    	printf("Text location:\n");
    	printf("\tAddress of main(Code Segment):%p\n",main);
    	printf("____________________________\n");
    	int stack_var0=2;
    	printf("Stack Location:\n");
    	printf("\tInitial end of stack:%p\n",&stack_var0);
    	int stack_var1=3;
    	printf("\tnew end of stack:%p\n",&stack_var1);
    	printf("____________________________\n");
    	printf("Data Location:\n");
    	printf("\tAddress of data_var(Data Segment):%p\n",&data_var0);
    	static int data_var1=4;
    	printf("\tNew end of data_var(Data Segment):%p\n",&data_var1);
    	printf("____________________________\n");
    	printf("BSS Location:\n");
    	printf("\tAddress of bss_var:%p\n",&bss_var);
    	printf("____________________________\n");
    	char *b = sbrk((ptrdiff_t)0);
    	printf("Heap Location:\n");
    	printf("\tInitial end of heap:%p\n",b);
    	brk(b+4);
    	b=sbrk((ptrdiff_t)0);
    	printf("\tNew end of heap:%p\n",b);

    	return 0;
	}

在centos 7 64位系统上的运行结果如下：

	below are addresses of types of process's mem
	Text location:
		Address of main(Code Segment):0x400600
	____________________________
	Stack Location:
		Initial end of stack:0x7fffbe16a594
		new end of stack    :0x7fffbe16a590
	____________________________
	Data Location:
		Address of data_var(Data Segment):0x60104c
		New end of data_var(Data Segment):0x601050
	____________________________
	BSS Location:
		Address of bss_var:0x601058
	____________________________
	Heap Location:
		Initial end of heap:0x928000
		New end of heap    :0x928004

从输出结果来看栈和堆相差的距离为：0x7fffbe16a594-0x928000=0x7FFFBE0D7D94，大约为：131070G，是一个非常大的数，也许永远都不会用到那么大。

在centos 5 32位系统上的运行结果如下：

	below are addresses of types of process's mem
	Text location:
    	Address of main(Code Segment):0x80484e4
	____________________________
	Stack Location:
    	Initial end of stack:0xbfce4a9c
    	new end of stack    :0xbfce4a98
	____________________________
	Data Location:
    	Address of data_var(Data Segment):0x8049a98
    	New end of data_var(Data Segment):0x8049a9c
	____________________________
	BSS Location:
        Address of bss_var:0x8049aa8
	____________________________
	Heap Location:
        Initial end of heap:0x9a36000
        New end of heap    :0x9a36004

栈和堆相距为：0xbfce4a9c-0x9a36000=0xB62AEA9C,大小约为：2.8G左右，一般的程序用不到，但是大型的程序也许会用到。比如今天一个朋友说外网的游戏服务器在动态分配2G左右的内存后，再用malloc分配内存，返回内存分配失败。由于程序比较大，Data segment和BSS segment数据量比较大，用去了大约0.8G的内存，所以会引起内存分配失败。
