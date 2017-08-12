---
title: 汇编语言学习笔记(一)
date: 2017-08-12 11:18:46
categories: 技术开发
tags: [汇编]
---
## 一：变量类型

汇编语言变量基本类型如下：

`sdword ：表示32位整数`

`dword：表示32位无符号整数`

`sword：表示16位整数`

`word：表示16位无符号整数`

`sbyte：表示8位整数`

`byte：用于表示字节，大小为8位`

变量的表示和定义：

C语言中
``` cpp
int num3 = 5；
```
汇编中
``` asm
num3 sdword 5；
```
C语言中
``` cpp
char name1；

char name2=‘A’；
```
汇编中：
``` asm
name1 byte ？

name2 byte ‘A’；
```
C语言中：
``` cpp
char name[] = "iloveu";
```
汇编中：
``` asm
name byte ‘iloveu’，0 ;字符串结汇通常带有二进制的0，占用一个字节。
```
<!--more-->

## 二：立即数的存储

`在汇编语言中将一个常量存储在内存空间需要通过mov操作，立即数为一个常量`。

将5存储在变量num1中

num1 sdword ？;

mov num1 5;

`mov操作后面两个参数为内存或寄存器，mov两个参数可以为以下关系：`

`mov mem , imm 将立即数存储到内存中`

`mov reg, mem 将内存单元的数据存储到寄存器中`

`mov mem, reg 将寄存器中的内容存储到内存单元中`

`mov reg, imm 将立即数存储在寄存器单元中`

`mov reg, reg 将第二个寄存器中的内容存储到第一个寄存器中`

`但是不能mov mem, mem`， 不能将一个内存中的内容存储到另一个内存中

需要通过mov reg， mem1； mov mem2 reg；可达到目的。

 ## 三：寄存器

数据不能直接从一个内存单元中移动到另一个内存单元中，需通过寄存器做中转。

`内存单元数据放在随机存储器(RAM)中`，`RAM将内存单元中的数据拷贝到CPU的寄存器中`，然后将CPU的寄存器中的数据copy到另一个存储单元中。
![1.png](1.png)

`将内存单元的内容复制到寄存器中的动作成为装载`，`将cpu中寄存器的数据copy到内存中成为存储`。

不是所有的寄存器都可以被开发者使用，提供给程序员的寄存器为通用寄存器。

`16位通用寄存器为ax,bx,cx,dx`

`32位通用寄存器为eax,ebx,ecx,edx`

`以eax举例，eax低16位为ax寄存器，ax寄存器低8位为al寄存器，高8位为ah寄存器`。如图所示：

![2.png](2.png)

`eax为累加器`，`主要用于参与算术逻辑`

`ebx为基址寄存器`，`用于处理数组`

`ecx为计数器`，`用于循环操作`

`edx为数据寄存器`，`用于算术指令`。

除此之外还有`eflags表示状态的寄存器，esp堆栈指针指向栈顶部，ebp基址指针指向栈底部`。

## 四：输入输出

字符串的输出

C语言版：
``` cpp
#include<stdio.h>
int main()
{
     printf("%s\n"," Hello World!");  
     return 0;
}
```
用汇编实现为：

``` asm
            .386
            .model flat ,c
            .stack 100h
printf      PROTO arg1:Ptr Byte, printlist:VARARG
            .data
msg1fmt     byte "%s", 0Ah, 0
msg1        byte "Hello World", 0
            .code
main        proc
            INVOKE printf, ADDR msg1fmt, ADDR msg1
            ret
main        endp
            end            
```
 `.386是一个汇编指令`，它指明程序可以被汇编为在Intel386系列或更高级的计算机运行，

`.model flat表明程序使用保护模式`，.model flat ,c表示该程序可以与C或C++程序进行连接，且需要运行在Visual C++环境中。

`.stack 100h 表示堆栈的大小应该达到十六进制100个字节`

`printf 名称后面的 PROTO表示printf为函数原型， arg1表示printf第一个参数为指向Byte类型的Ptr指针。`

第二个参数为VARARG类型可变参数printlist

`.data表示数据段，.code表示代码段。`

`msg1fmt 为byte类型的字符串,0Ah为16进制表示\n,0表示字符串结尾\0`

`msg1为byte类型的字符串，“Hello World”`

`proc汇编指令表示一个过程，过程名称为main`

`INVOKE 为汇编指令，调用函数printf完成输出，`

`格式为 INVOKE 函数名， 参数一，参数二`

这里为msg1fmt字符串的首地址，以及msg1的首地址。

`ADDR为取地址操作。`

`ret表示返回和C语言return 0一样。`

`main endp表示main proc结束`

`end表示整个汇编程序结束。`

整数的输出和字符串类似，下面通过输入输出程序介绍汇编语言的输入和输出

``` asm
　　　　 .386
　　　　 .model flat c
　　　　 .stack 100h
printf  PROTO arg1:Ptr Byte, printlist:VARARG
scanf   PROTO arg2:Ptr Byte, inputlist:VARARG
　　　　 .data
in1fmt  byte "%d", 0
msg0fmt byte 0Ah, "%s", 0
msg1fmt byte 0Ah, "%s%d", 0Ah, 0Ah, 0
msg0    byte "Enter an integer for num1: ", 0
msg1    byte "The integer in num2 is: ",0
num1 　 sdword ?
num2    sdword ?
　　　　.code
main   proc
　　　 INVOKE printf, ADDR msg0fmt, ADDR msg0
       INVOKE scanf, ADDR in1fmt, ADDR num1
       mov eax, num1
       mov num2, eax
       INVOKE printf, ADDR msg1fmt, ADDR msg1, num2
      ret
main  endp
      end
       
```
输入和输出类似，C语言中输入函数为scanf

汇编程序通过

`scanf   PROTO arg2:Ptr Byte, inputlist:VARARG` 
`PROTO表示scanf为函数原型， 第一个参数为指向Byte的指针,第二个参数为可变参数`
`scanf输入整数num1，需要取地址，所以会采用ADDR num1`

将上述汇编转译为C语言
``` cpp
#include<stdio.h>
int main()
{
    int num1, num2;
    printf("\n%s", "Enter an integer for num1: ");
    scanf("%d", &num1);
    num2 = num1;
    printf("\n%s%d\n\n", "The integer in num2 is: ", num2);
    return 0;

}
```
## 五：算数指令

加法指令

`add mem, imm  将立即数imm和内存单元mem中数值相加存储在mem中`

`add reg, mem 将内存mem中的数值和寄存器reg中的数值相加，结果存储在reg中`

`add reg, imm 将立即数imm和寄存器reg中数据相加存储在reg中`

`add mem, reg 将寄存器中的数据和内存单元mem中数据相加存储在mem中`

`add reg, reg 将寄存器中的数据和寄存器相加，最后存储在第一个reg中`

注意：不可add mem, mem 也不可 add imm, imm 两个立即数没法相加结果也没办法存储

`两个内存单元也没办法相加，和mov mem, mem是一样的，都不可以直接操纵两个内存。`

减法指令

`sub mem, imm 将内存单元mem中数据减去立即数imm存储在mem中`

`sub reg, mem 将寄存器reg中数据减去mem中的数据，存储在reg中。`

`sub mem, reg 将内存单元mem中数据减去寄存器reg中数据，结果存储在mem中`

`sub reg ,imm 将寄存器reg中的数据减去立即数imm，将结果存储在reg中`

`sub reg, reg 将第一个reg数据减去第二个reg数据，最后结果存储在第一个reg中`

`注意：不可sub mem, mem 也不可 add imm, imm 两个立即数没法相减结果也没办法存储

两个内存单元也没办法相减，和mov mem, mem是一样的，都不可以直接操纵两个内存。`

 

乘法指令imul

`imul mem 将内存单元mem中的数据和寄存器eax中的数据相乘，结果存储在edx:eax中`

`imul reg 将寄存器reg中的数据和寄存器eax中的数据相乘，结果存储在edx:eax中`

`注意：imul 只有一个参数，这个参数只能为内存单元或者寄存器，不能是imm立即数`

`由于乘法指令imul可能会造成乘积位数大于乘数和被乘数的位数，所以将结果存储在edx:eax中`

`imul为有符号的乘法指令， mul为无符号的乘法指令， imul和mul参数只能为mem或reg`，如果

想计算num = mem* 3；mem为内存单元数据，3为立即数

imul实现为
``` asm
mov eax, 3

imul mem

mov num, eax
```
 
`除法指令idiv`

`和乘法指令类似，idiv 为有符号除法， div为无符号除法`

`进行除法操作前将被除数放入edx:eax中，除法操作后edx存储的为余数，eax存储的为商。`

举例 num = 13/,5;

`汇编语言实现过程先将13放入eax中，调用cdq 可以将eax内容扩展到edx:eax寄存器对中，将5放入内存单元或寄存器中，调用idiv最终edx存储为3，eax存储为2`

汇编语言实现如下：
``` asm
mov eax， 13
mov ebx，5
cdq
idiv ebx
mov num , eax
```

`cbw 指令可将字节转换为字，一个字为两个字节，即将al内容扩展到寄存器ax中`

`cwd 字转换为双字， 即将ax中的内容扩展到eax中`

`cdq 双字转换为4个字， 即将eax中内容扩充为edx:eax中。`

`只有cdq允许有符号位，所以进行idiv和div操作前执行cdq将eax中内容扩充到edx:eax中。`

`注意：idiv和div 只有一个参数，这个参数只能为内存单元或者寄存器，不能是imm立即数`

自增自减指令

`dec reg`    reg中的数据自减

`dec mem `   mem中的数据自减

`inc reg`    reg中数据自增

`inc mem`    mem中数据自增。

`注意：dec 和inc  只有一个参数，这个参数只能为内存单元或者寄存器，不能是imm立即数`

取反指令，也就是求一个数的补码

`neg mem`

`neg imm`

`注意：neg  只有一个参数，这个参数只能为内存单元或者寄存器，不能是imm立即数`

下面两个例子是综合计算

``` asm
; v = -w + m * 3 - y/(z-4) 
;w,v,m,y, z分别为内存单元

mov ebx, w
neg ebx
mov eax,3
imul m
add eax, ebx
mov v ,eax
mov ebx, z
sub ebx, 4
mov eax, y
cdq
idiv ebx
sub v, eax
```

汇编语言基础第一部分记录到此为止，下一篇跟进记录

![1.jpg](1.jpg)
