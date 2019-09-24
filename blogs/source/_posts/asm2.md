---
title: 汇编语言学习笔记(二)
date: 2017-08-12 12:04:04
categories: [汇编]
tags: [汇编]
---
## 六、选择结构

`if-then结构`

C语言版本
``` cpp
if(count == 10)
{
   count --;
   i++;
}
```

MASM汇编
``` asm
.if count==10
dec count
inc i
.endif
```

`cmp指令，该指令用于比较两个参数大小`

`cmp mem, imm `比较内存mem和立即数imm大小

`cmp reg, imm `比较寄存器reg和立即数imm大小

`cmp reg, mem `比较寄存器reg和内存mem大小

`cmp mem, reg `比较内存mem和寄存器reg大小

`cmp imm, reg` 比较立即数和寄存器reg大小

`cmp imm, mem`比较立即数和内存mem大小

`cmp reg, reg`比较两个寄存器数据大小

`.if 内部是通过cmp实现的，.if和cmp都不能比较两个内存中数据大小` 
<!--more-->

`条件跳转指令`

`je 相等时跳转`

`jne 不相等时跳转`

`je 和jne可以用于无符号数比较跳转，也可用于符号数比较跳转`

 

`符号数据跳转指令`

`jg  大于时跳转`

`jge 大于或等于时跳转`

`jl 小于是跳转`

`jle 小于或等于时跳转`

 

`jnle 不小于且不等于时跳转，即大于时跳转`

`jnl 不小于时跳转`

`jnge 不大于且不等于时跳转`

`jng 不大于时跳转`

`这类指令可用上面的jg，jge，jl，jle替代，尽量不要使用，不容易理解。`

.if-.end汇编指令可用上面的cmp指令和条件跳转指令替代

如
``` cpp
.if number == 0
dec number
.endif
```

 用条件跳转指令实现为：
``` asm
if01: cmp number, 0
        jne  endif01
then01:  dec number
endif01: nop
```

 `if-then-else结构`

汇编指令的结构为

``` asm
.if 条件

    ;执行语句

.else

   ;执行语句

.endif
```
 

举例说明，C语言
``` cpp
if (x>=y)
    x--;
else
    y--;
```

汇编语言

``` asm
if01:     mov eax,x
          cmp eax,y
          jl else01
then03:   dec x
          jmp endif01
else01:  dec y
endif01:  nop
```
采用汇编指令
``` asm
mov eax , x
.if eax >= y
   dec x
.else
   dec y
.endif
```
嵌套if结构

C语言

``` cpp
if(x < 50)
  y++;
else
    if(x <= 100)
       y = 0;
   else
       y--;
```

汇编语言

采用汇编指令

``` asm
.if x < 50
   inc y
.elseif x <= 100
   mov y, 0
.else
   dec y
.endif 
```
 

通过跳转指令实现上述步骤

``` asm
if01:   cmp eax, 50
        jge else01
then01: inc y
        jmp endif01
else01: nop
if02:   cmp x, 100
        jg else02
then02: mov y,0
        jmp endif01
else02:  dec y
endif01:  nop
```
case 结构

C语言

``` cpp
switch(w)
{
   case 1: x++;
     break
  case 2:
  case 3: y++;
     break;
  default: z++;

}
```
`汇编语言中没有case switch对应的汇编指令，但是可以通过cmp指令和跳转指令实现`

``` asm
switch01:    cmp  w, 1
             je case1
             cmp w,2
             je case2
             cmp w,3
             je case2
             jmp default01
case1:  inc x
            jmp endswitch01
case2: inc y
           jmp endswitch01
default01 : inc z
endswitch01: nop
```

`无符号跳转指令`

`ja 高于跳转`

`jae 高于或等于跳转`

`jb 低于跳转`
``` cpp
i = 1;
while(i <= 3)
{
     i++;
}
```
`jbe 低于或等于时跳转`

 逻辑运算符综合实现
 ``` asm
.if w==1 || x==2&&y == 3
  inc z
.endif
```
条件跳转指令实现上述逻辑

``` asm
if01:  cmp w,1
       jne or01
or01:   cmp x, 2
        jne endif01
        cmp y, 3
        jne  endif01
then01: inc z
endif01: nop
```
 ## 七、迭代结构

 ### 1 前置循环结构while

C语言
``` cpp
i = 1;
 while(i <= 3)
{
    i++; 
}
```
汇编指令实现
``` asm
mov i, 1
.while (i <= 3)
   inc i
.endw
```

条件跳转指令实现
``` asm
mov i, 1
while01: cmp i, 3
         jg endw01
         inc i
         jmp while01
endw01: nop
```

### 2 后置循环结构 do while

C语言
``` cpp
i = 1;
do
{
   i++;

}while(i <= 3);
```

汇编指令 .repeat .until

``` asm
mov i, 1
.repeat
   inc i
.until i > 3
```

`.repeat-.until和C语言do -while一样，第一次先执行循环体内容，do-while是在每次运行循环体后判断条件不满足跳出循环`

`.repeat-.until 是判断条件不满足则一直循环，直到.until后面条件成立则跳出循环`

通过汇编跳转指令实现
``` asm
mov i, 1
repeat01:  nop
           inc i
           cmp i, 3
           jle repeat01
endrpt01: nop
```
### 3 固定循环结构 for
``` cpp
for(i=1; i <= 3; i++)
{
     //循环体
}

```
汇编语言
``` asm
mov ecx, 3
.repeat 
;  循环体
.untilcxz
```
`.repeat-.untilcxz 对应C语言的for循环固定循环结构，它和.repeat-.until都是后置循环结构`。

`都要先执行一次循环体，然后.untilcxz会将ecx中的数据值减1，接着判断ecx是否为0，ecx为0则跳出循环`

`ecx大于0则继续执行循环。所以执行.repeat-.untilcxz这个循环前保证ecx大于0`

`.repeat-.untilcxz 内部是loop实现的，所以同样可以通过loop实现该循环`
``` asm
mov ecx, 3
for01 : nop
         ; 循环体
         loop for01
endfor01: nop
```

`loop 和.untilcxz一样，都会将ecx中的值减1，然后判断ecx是否为0，为0则终止循环，大于0则继续循环。`

`loop指令最多只能向前跳转128字节处，也就是循环体内容不得超过128字节。同样.repeat-.untilcxz循环体也不得超过128字节。`

`与.if高级汇编指令一样，.while-end高级汇编指令和.repeat-.until高级汇编指令都基于cmp，不可直接比较两个内存变量。`

`使用loop指令和.repeat-untilcxz指令前，需要保证ecx大于0，在循环体中尽量不要更改ecx数值，如果需要更改则将ecx

数据先保存在内存中，执行完相关操作后将ecx数值还原，保证.untilcxz和loop操作ecx的准确性。`

下面用`汇编语言实现斐波那契数列`

if n= 0, then 0

if n = 1, then 1

if n = 2, then 0+ 1 = 1

if n = 3, then 1+ 1 = 2

if n = 4, then 1+2 = 3

n 为任意非负数，计算n对应的数值sum。

用.while循环实现如下

``` asm
.if n==0
mov sum, 0
.endif
.if n==1
mov sum, 1
.endif
mov ecx, n
mov eax, 0
mov sum, 1
.while (ecx <= n)
   mov ebx, sum
   add sum, eax
   mov eax, ebx
.endwhile
```
 
到目前为止，循环和条件判断就介绍到这里，下一期介绍堆栈位移之类的指令。

我的公众号：
![1.jpg](1.jpg)

