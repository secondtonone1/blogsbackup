---
title: 汇编语言学习笔记(六)
date: 2017-08-15 16:08:33
categories: [汇编]
tags: [汇编]
---
十八、`字符串处理`

前文介绍过字符串的处理，字符串是`byte类型` 的数组，现在实现一段代码，将字符串string1数据copy到字符串string2中

代码如下

``` asm
　　　　.data
string1 byte "Hello World!", 0
string2 byte 12 dup(?), 0
　　　　.code
mov ecx, 12
mov ebx,0
.repeat
mov al, string1[ebx]
mov string2[ebx], al
inc ebx
.untilcxz
```

通过`ecx递减`，将字符串string1每个字符一次copy给string2中，其中用到了`ebx基址寄存器`。

 也可以通过esi和edi寄存器

``` asm
.data
string1 byte "Hello World!"
stirng2 byte 12 dup(?), 0
　　　　.code
mov ecx,12
lea esi, string1
lea edi, string2
.repeat
mov al,[esi]
mov [edi],al
inc edi
inc esi
.untilcxz
```
<!--more-->
这些代码可以通过字符串操作的其他指令替代，使代码更简洁

1 `movsb指令`

`movsb是字符串移动指令`，`将esi指向的字符移动到edi指向的空间`，`并且将ecx减1`，`寄存器esi和edi内容增加1或者减少1`

如果仅仅使用movsb指令作用不大，配合循环使用，功能很强大。`cld指令表示方向标志值清0，调用movsb会使esi和edi内容增加1`

`std指令表示设置方向标志值，调用movsb会使esi和edi内容减少1`

还是上面的需求，这次使用movsb完成目标

``` asm
mov ecx,12                ; 字符串大小为12
mov esi, offset string1   ; esi指向string1起始位置
mov edi, offset string2   ; edi指向string2起始位置
cld                       ; cld指令调用后 ，调用movsb会使esi和edi分别加1
.repeat
movsb                    ; 先将esi指向的空间数据拷贝到edi指向的空间
                          ; 然后esi和edi分别加1，并且ecx值减少1
.untilcxz
```
和movsb配合使用的几个前缀：

`rep `  重复操作，等于循环，直到ecx为0退出循环

`repe  `如果相等，则重复操作，如果不等，那么退出循环，或者ecx为0退出循环

`repne`  不相等，则重复操作，相等则退出循环，或者ecx为0退出循环

上面的代码可以使用 rep movsb更改，完成同样的目的
``` asm
mov ecx, 12
lea esi, string1
lea edi, string2
cld
rep movsb   ;循环直到ecx为0结束
```

2  `scasb指令`

`scasb指令用于在edi寄存器指定的空间中搜寻al所存储的字符`。

`scasb指令操作流程为`：

将要查询的字符放入al寄存器中，如果要在字符串string中查找匹配的字符，那么将edi存储string的首地址，调用scasb进行匹配，scasb指令调用后会将edi

存储数值加1，并且ecx值-1，为了完成字符串string中逐个字符匹配，需要在scasb前面加上repne指令，这样在不相等时继续

查找，直到查找到指定字符退出循环。此时edi指向的位置为字符串string中匹配字符位置的下一个位置。

举例实现：

``` asm
.data
name1  byte "Abe Lincoln"
.code
mov al, ' '                ; 在al寄存器中存储空格，为了在name1中查找该空格
mov ecx, lengthof name1    ;ecx存储name1字符串长度，为11
lea edi, name1             ;edi指向name1首地址
repne scasb                ;匹配edi指向空间的字符和空格是否相等，不相等则edi加1
                           ;ecx减1，直到相等或者ecx为0退出循环
```
 
运行后ecx为7，edi执行字符L的位置

效果图如下：
![1.jpg](1.jpg)
 

3  `stosb指令`

`stosb用于把寄存器al存储的数据放入寄存器edi所指向的空间,指令完成后edi增加1`

stosb等价于：
``` asm
mov [edi],al
inc edi
```

4  `lodsb指令`

`lodsb用于把寄存器esi所指向的空间数据存储到al寄存器中，指令完成后esi增加1`

lodsb等价于：

``` asm
mov al, [esi]
inc esi
```

下面实现将字符串姓名倒置功能，并将名字和姓中间添加字符“ ”和“,”

如“Abe Lincoln”变为“Lincoln, Abe”

``` asm
 　　.data
name1 byte "Abe Lincoln"
name2 byte 12 dup (?)
　　　　.code
mov al, ''
mov ecx, lengthof name1
lea edi, name1
repne scasb
push ecx
mov esi, edi
lea edi, name2
rep movsb
mov al, ' '
stosb 
mov al, ' '
stosb
mov ecx, lengthof name2
pop eax
sub ecx, eax
dec ecx
lea esi , name1
rep movsb
```

5  `cmpsb指令`

字符串比较指令，该指令和movsb差不多，比较edi和esi指向空间数据大小，并且edi和esi增加1或者减少1，

可通过`cld或std`设置，`并且ecx减少1`

比较“James” 和 “James”

代码如下：

``` asm
.data
name1 byte "James"
name2 byte "James"
.code
mov ecx, lengthof name1
lea esi, name1
lea edi, name2
cld
repe cmpsb
```

相等情况下ecx为0，edi和esi都指向字符串最后一个字符的下一个位置
![1.jpg](1.jpg)

如果比较的字符为“James”和“Jamie”,那么效果如下：
![2.jpg](2.jpg)

如果比较字符为“Marci”和“Marcy”，那么上述代码就没办法区分是否相等了，因为此时ecx也为0
![3.jpg](3.jpg)

如果ecx大于0，那么两个字符串肯定不相等的，如果ecx为0，那么将esi和edi都-1，得到最后一个元素，

再将esi和edi指向空间的数据比较即可。

下面将上述代码完善一下，比较两个长度相等的字符串name1和name2

``` asm
mov ecx, lengthof name1
lea esi, name1
lea edi, name2
cld 
repe cmpsb
.if ecx > 0
;输出不相等
.else 
  dec esi
  dec edi
  mov al, [esi]
  .if al != [edi]
   ; 输出不相等
  .else
    ;输出相等
  .endif
.endif
```
 

十九、`字符串总结`

1  `movsb 指令将寄存器esi指向的字节类型字符串的内容移动到寄存器edi指向的位置`，`通过cld或std设置esi和edi增加还是减少`

2  `cmpsb 指令对寄存器的esi指向的字符串中的一个字符和edi所指向的一个字符进行比较`，`通过cld或std设置esi和edi增加还是减少`

3  `movsb指令前边的rep前缀会让该指令重复执行`，`循环次数等于ecx的值`，`每一次循环ecx的值减1`，`直到为0停止`。

4  `cmpsb指令前面的repe前缀的作用类似于rep前缀，当ecx为0是，停止执行指令。或者当esi和edi所指向的两个空间的字节内容不同，循环也会停止`。

    `repne当esi和edi所指向的两个空间的字节内容相同时，停止`。

5  `scasb指令将一个字符串进行搜索，查找字符串中是否存在寄存器al中所存储的字符。如果找到该字符，那么edi指向的地址比该字符的地址靠后一个字节`。

    `stosb将al寄存器的内容存储在edi所指向的内存空间的位置中。lodsb指令将把寄存器esi所指向的内存空间位置中的字符复制到al寄存器中`。

 

我的微信公众号，谢谢关注：

![4.jpg](4.jpg)