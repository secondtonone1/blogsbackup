---
title: golang实现四种排序(快速，冒泡，插入，选择)
date: 2019-06-29 09:58:13
categories: [golang]
tags: [golang]
---
前面已经介绍golang基本的语法和容器了，这一篇文章用golang实现四种排序算法,快速排序，插入排序，选择排序，冒泡排序。既可以总结前文的基础知识，又可以熟悉下golang如何实现这四种排序。
## 快速排序
### 算法介绍
假设用户输入了如下数组

下标 | 0     |   1    |   2    |   3    |   4    |   5    |
数值 | 6     |   2    |   7    |   3    |   8    |   9    |

<!--more-->
创建变量i=0（指向第一个数据）, j=5(指向最后一个数据), k=6(赋值为第一个数据的值)
我们要把所有比k小的数移动到k的左面，所以我们可以开始寻找比6小的数，从j开始，从右往左找，不断递减变量j的值，我们找到第一个下标3的数据比6小，于是把数据3移到下标0的位置，把下标0的数据6移到下标3，完成第一次比较

下标 | 0     |   1    |   2    |   3    |   4    |   5    |
数值 | 3     |   2    |   7    |   6    |   8    |   9    |

i=0 j=3 k=6
接着，开始第二次比较，这次要变成找比k大的了，而且要从前往后找了。递加变量i，发现下标2的数据是第一个比k大的，于是用下标2的数据7和j指向的下标3的数据的6做交换，数据状态变成下表：

下标 | 0     |   1    |   2    |   3    |   4    |   5    |
数值 | 3     |   2    |   6    |   7    |   8    |   9    |

i=2,j=3,k=6
接下来继续重复上边的步骤，从j开始，从右往左找比k小的，此时i=2,j=3，j再移动一位就和i相等了，此时就完成了k=6的排序，此时k=6,i和j相等都为2，6右边的数都比6大，6左边的数比6小。
接下来分别比较6左边的序列(下标从0到1)和6右边(下标从3到5)的序列，同样采用上述办法，直到所有序列都比较完成。
### 算法实现
``` golang
func quickSort(slice []int, begin int, end int) {
    //slice为空
    if len(slice) == 0 {
		return
	}
    //end越界
	if end >= len(slice) {
		return
	}
    //begin越界
	if begin < 0 {
		return
	}
    //下标碰头无须比较
	if begin >= end {
		return
	}
    //i从左到右，j从右到左
	i := begin
    j := end
    //value就是比较的值
	value := slice[i]

	for {
        //从右往左比较，找到比value小的交换位置
		index := j
		for ; index > i; index-- {
			if value > slice[index] {
				slice[i] = slice[index]
				//slice[index] = value
				break
			}
        }
        //更新j的位置
        j = index
        //从左往右比较
		for index = i; index < j; index++ {
			if value < slice[index] {
				slice[j] = slice[index]
				//slice[index] = value
				break
			}
        }
        //更新i的位置
		i = index
        //i和j碰头则更新value的位置，并且比较value左右序列
		//fmt.Println(i, j)
		if i >= j {
			slice[i] = value
			quickSort(slice, begin, i-1)
			quickSort(slice, i+1, end)
			return
		}
        //否则继续比较，此时i,j已经缩小范围
	}

}
上述算法时间复杂度达到O(nlogn)
```
## 插入排序
### 算法描述
假设用户输入了如下数组

下标 | 0     |   1    |   2    |   3    |   4    |   5    |
数值 | 6     |   2    |   7    |   3    |   8    |   9    |

假设从小到大排序
插入排序先从下标为1的元素2开始，比较前边下标为0的元素6,2比6小，则将6移动到2的位置，2放到6的位置

下标 | 0     |   1    |   2    |   3    |   4    |   5    |
数值 | 2     |   6    |   7    |   3    |   8    |   9    |

记下来比较下标为2的元素7，和前边0~1下标的元素对比，从后往前找，如果找到比7大的元素，则将该元素后边的序列依次后移，将7插入该元素位置
目前7不需要移动。
接下来寻找下标为3 的元素3，从下标3往前找，由于下标1,下标2的元素都比3大，所以依次后移，将3放倒下标1的位置。

下标 | 0     |   1    |   2    |   3    |   4    |   5    |
数值 | 2     |   3    |   6    |   7    |   8    |   9    |
以此类推，进行比较。
### 算法实现
``` golang
func insertSort(slice []int) {
	if len(slice) <= 0 {
		return
	}

	for i := 1; i < len(slice); i++ {
        //下标i的元素temp
		temp := slice[i]
		for j := i - 1; j >= 0; j-- {
            //从i-1的位置往前查找，如果前边的元素比temp大
            //就进行后移
			if temp < slice[j] {
				slice[j+1] = slice[j]
            }
            //否则找到比temp小的就将tmep插入。
			slice[j+1] = temp
			break
		}
	}
}
```
该算法时间复杂度为O(n*n)
## 冒泡排序
### 算法描述
假设用户输入了如下数组

下标 | 0     |   1    |   2    |   3    |   4    |   5    |
数值 | 6     |   2    |   7    |   3    |   8    |   9    |
冒泡排序依次比较相邻的两个元素 ，将大的元素后移即可。
先比较下标为0和下标为1的元素，6比2大，所以6和2交换位置。

下标 | 0     |   1    |   2    |   3    |   4    |   5    |
数值 | 2     |   6    |   7    |   3    |   8    |   9    |
接下来比较下标为1和下标为2的元素，6比7小所以不做交换。然后比较7和3,7比3大，7和三交换位置，以此类推，直到比较到最后一个元素

下标 | 0     |   1    |   2    |   3    |   4    |   5    |
数值 | 2     |   6    |   3    |  7    |   8    |   9    |
经过这一轮相邻元素的比较，将最大的元素9冒泡到最后的位置。
接下来重复上述步骤，从下标0开始到下标4两两比较，将第二大元素放到下标4的位置，因为下标5已经是最大元素，所以不参与比较。
### 算法实现
``` golang
func bubbleSort(slice []int) {
	for i := 0; i < len(slice); i++ {
		for j := 0; j < len(slice)-i-1; j++ {
			if slice[j] > slice[j+1] {
				slice[i], slice[j+1] = slice[j+1], slice[j]
			}
		}
	}
}
```
## 选择排序
### 算法描述
假设用户输入了如下数组

下标 | 0     |   1    |   2    |   3    |   4    |   5    |
数值 | 6     |   2    |   7    |   3    |   8    |   9    |
从下标0开始，比较6和其他位置的元素，找到最小的元素2和6交换位置

下标 | 0     |   1    |   2    |   3    |   4    |   5    |
数值 | 2     |   6    |   7    |   3    |   8    |   9    |
接下来从下标1开始，比较6和后边位置的元素，选择最小的和6交换位置。

下标 | 0     |   1    |   2    |   3    |   4    |   5    |
数值 | 2     |   3    |   7    |   6    |   8    |   9    |
以此类推，从下标2开始，比较7和后边的元素，选择最小的6交换位置

下标 | 0     |   1    |   2    |   3    |   4    |   5    |
数值 | 2     |   3    |   6    |   7    |   8    |   9    |
以此类推，直到下标5的位置元素都比较完。
### 算法实现
``` golang
func selectSort(slice []int) {
	for i := 0; i < len(slice); i++ {
		for j := i + 1; j < len(slice); j++ {
			if slice[i] > slice[j] {
				slice[i], slice[j] = slice[j], slice[i]
			}
		}
	}
}
```
该算法时间复杂度为o(n*n)
## main函数中调用并测试
``` golang
func main() {
	array := [...]int{6, 2, 7, 3, 8, 9}
	slice := array[:]
	quickSort(slice, 0, len(slice)-1)
	fmt.Println(slice)
	slice2 := array[:]
	bubbleSort(slice2)
	fmt.Println(slice2)
	slice3 := array[:]
	selectSort(slice3)
	fmt.Println(slice3)
	slice4 := array[:]
	insertSort(slice4)
	fmt.Println(slice4)

}
```
到此为止，四种基本的比较算法已经完成，感兴趣的可以自己实现以下。
上述所有源码下载地址
[源码下载地址](https://github.com/secondtonone1/golang-)
谢谢关注我的公众号
![wxgzh.jpg](wxgzh.jpg)




