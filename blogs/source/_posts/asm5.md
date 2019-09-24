---
title: 汇编语言学习笔记(五)
date: 2017-08-15 15:39:31
categories: [汇编]
tags: [汇编]
---
## 十六、`数组`

数组的基本表示方法

`numary sdword 2,5,7`

numary数组中有三个元素，为sdword类型，分别为2,5,7

`empary sdword ?, ?,?`

empary数组为sdword类型元素，未初始化。

如果数组元素很多可通过

`zeroary sdword 100 dup(0)`

zeroary数组中有100个0

`empary sdword 3 dup(?)`

empary 数组中有3个未初始化的sdword类型数据

`mov eax, numary+8`; `表示把数组numary第3个元素放入eax中`

`sdword为四字节，numary+0表示numary首地址，也是第一个元素地址，以此类推，numary+8为第三个元素首地址`。

`mov numary+0, eax;` 将eax内容放入数组第一个元素中。

`除了采用数组首地址+偏移地址方式，可以采用ebx基址寄存器进行数组索引`。

访问numary第三个元素

`mov ebx, 8;`ebx存储8

`mov eax, numary[ebx];访问numary偏移8字节的元素，也就是第三个元素，放入eax中。`
<!--more-->

举个例子C语言：
``` cpp
sum = 0
for(i = 0; i < 3; i++)
{
  sum +=numary[i];
}
```
汇编语言：

``` asm
mov sum, 0
mov ecx ,3
mov ebx, 0
.repeat
mov eax, numary[ebx]
add sum, eax
add ebx,4
.untilcxz
```
除了使用基址寄存器ebx，还可以使用寄存器esi和寄存器edi进行索引,

`esi为源索引寄存器`，`edi为目的索引寄存器`。

第一种方法
``` asm
mov ebx,4

mov eax,numary[ebx]
```

第二种方法
``` asm
mov esi, offset numary+4

mov eax,[esi]
```


`第二种方法先将numary+4的地址放入esi中`，`然后[esi]取出esi指向的地址空间的数据`，`也就是numary+4地址空间里的数据将数据放入eax中`。

两种方法的效果图：

![1.jpg](1.jpg)
![2.jpg](2.jpg)

除了上述两种方法，还有第三种方法
``` asm
lea esi, memory+4

mov eax, [esi]
```

`lea 和offset的区别`：

`offset 是在编译时将地址写入esi`

`lea是动态写入，每次运行时将地址写入esi中`。

去实现如下代码：

``` cpp
j=n-1;
for(i=0; i < n/2; i++)
{
    temp=numary[i];
    numary[i] = numary[j];
    numary[j] = temp;
    j--;
}
```

通过汇编实现：

``` asm
mov ecx,n
sar ecx,1
lea esi,numary+0
mov edi, esi
mov  eax,n
dec eax
sal eax,2
add edi, eax
.repeat
mov eax, [esi]
xchg eax, [edi]
mov [esi], eax
add esi,4
sub edi,4
.untilcxz
```
数组还有两个指令，`lengthof表示数组元素的个数`

`sizeof表示数组总共占用多少字节`。

前面的代码可以通过这两个指令修改

``` asm
mov ecx, lengthof numary
sar ecx,1
lea esi, numary+0
mov edi, esi
mov eax, sizeof numary
sub eax,4
add edi, eax
.repeat
mov eax,[esi]
xchg eax, [edi]
mov [esi], eax
add esi, 4
sub edi,4
.untilcxz
```

## 十七、数组总结

`1 esi 为源索引寄存器，主要用于从esi指向地址空间取出数据`

`2 edi为目的索引寄存器，主要用于向edi指向地址空间写入数据`

`3 esi和edi存储的为地址，[esi]和[edi]为他们指向的地址空间存储的数据`

`4 可以通过mov edi, esi将esi寄存器存储的地址放入edi中，因为两个操作数都是寄存器`

`5 不可以使用mov [edi],[esi];因为两个操作数都为内存，这是汇编指令mov不允许的。`

`6 寄存器ebx为基址寄存器，可通过数组名[ebx]取出数组首地址偏移ebx存储的字节数的元素`。

7 offset操作符的mov指令，如`mov eax, offset sumary+4`
是`将sumary首地址偏移4字节地址写入eax`，此时eax存储的是第二个元素首地址,他是静态的获取地址，而lea是动态的获取地址

8 `lengthof用于计算数组元素数量`，`sizeof用于计算数组总共占用多少字节`。

 

数组的介绍到此为止，下一篇是字符串的介绍

我的公众号：

![3.jpg](3.jpg)

 