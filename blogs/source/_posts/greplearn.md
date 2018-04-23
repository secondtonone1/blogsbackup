---
title: grep命令学习和总结
date: 2018-04-02 22:36:12
categories: 技术开发
tags: [shell]
---

总结grep命令知识
grep主要功能是返回指定文件中包含符合规则的文本行。

## 1 在指定文件file_name中查找包含match_pattern 的文本行
``` bash
grep "match_pattern" file_name
```
 
<!--more-->

## 2 在多文件中查找包含file_name的文本行
``` bash
grep "match_pattern" file_name1 file_name2 file_name3
```

## 3反向查找，输出不匹配file_name的文本行
``` bash
grep -v "match_pattern" file_name
```

## 标记匹配颜色
``` bash
greap "match_pattern" file_name --color=auto
```

## 使用正则表达式-E选项
``` bash
grep -E "[1-9]+"
```

``` bash
egrep "[1-9]+"
```

## 只输出文件中匹配的部分
``` bash
echo this is a test line.|grep -E "[a-z]+\."
```

## 统计文本中或文件中包含匹配字段的行个数
``` bash
grep -c "matchtest" file_name
```

## 输出包含匹配字符串的行行号
``` bash
grep -n "matchtest" file_name
```

## 打印包含匹配字符串位于该行的字节偏移
``` bash
echo i'm not girl | grep -b -o "not"
```
## 列出包含符合匹配字符串的文件名
``` bash
grep -l "matchtest" file1 file2 file3
```
## 在多级目录中递归搜索
``` bash
grep "text" . -r -n
```
## 忽略匹配样式中的字符大小写

``` bash
echo "hello world" | grep -i "HELLO"
```
## 多个匹配字段

``` bash
echo this is text test | grep -e "is" -e "line" -o
```
## 根据文件匹配

``` bash
cat textfile
aaa
bbb
```
``` bash
echo aaa bbb ccc ddd eee | grep -f textfile -o
```

## 显示匹配某个结果之后的3行，使用 -A 选项

``` bash
seq 10 | grep "5" -A 3
```

## 显示匹配某个结果之后的3行，使用 -B 选项

``` bash
seq 10 | grep "5" -B 3
```
## 显示匹配某个结果的前三行和后三行，使用 -C 选项：

``` bash
seq 10 | grep "5" -C 3

```
