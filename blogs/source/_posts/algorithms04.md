---
title: 二分查找升序序列
date: 2022-03-02 18:09:37
categories: [数据结构和算法]
tags: [数据结构和算法]
---

## 问题描述
有一个连续的int数组，数组中的数据升序排序，数组中的数据不唯一，有重复数据，要求在数组中查找指定值为target的数据，返回target最小的下标，如果找到返回其最小的下标，如果没有找到，返回-1， 要求用 用二分查找的方式解决上述问题， 要求时间复杂度为Olog(n),空间复杂度为O(1)
 
举例 数组nums = [0,1,1,3,3,4]， 查找target为1的数据最小的下标则返回1。如果查找target为100，则返回-1
## 解决思路
用二分查找解决上述问题，用一个数组中间的数据和target比较，如果中间值比target小，则说明target在中间值右侧。
如果中间值比target大，则说明target在中间值左侧。
如果中间值和target相等，则找到该target的位置，但是由于target不唯一，所以要向左移动，找到target最小的下标。
<!--more-->
## 编码
将上边的逻辑实现如下
``` cpp
int binary_search(int nums[], int begin, int end, int target)
{
    //只要有一个越界就是没找到
    if (begin < 0 || begin > end || end < 0)
    {
        return -1;
    }

    //找到中间值比较
    int mid = (begin + end) / 2;
    // cout << "mid is " << mid << endl;
    //要找到的target在右半部分
    if (target > nums[mid])
    {
        return binary_search(nums, mid + 1, end, target);
    }

    //要找到的target在左半部分
    if (target < nums[mid])
    {
        return binary_search(nums, begin, mid - 1, target);
    }
    // cout << "nums[mid] is " << nums[mid] << endl;
    //如果找到target相等的元素，需要向左移动找到最小索引
    for (int i = mid; i > begin; i--)
    {
        if (target == nums[mid])
        {
            continue;
        }
        //   cout << "i is " << endl;
        return i + 1;
    }
}
```
在main函数中调用如下
``` cpp
int nums[] = {0, 1, 1, 3, 3, 4};
int find = binary_search(nums, 0, sizeof(nums) / sizeof(int) - 1, 3);
cout << "find is " << find << endl;
int find2 = binary_search(nums, 0, sizeof(nums) / sizeof(int) - 1, 4);
cout << "find is " << find2 << endl;
int find3 = binary_search(nums, 0, sizeof(nums) / sizeof(int) - 1, 0);
cout << "find is " << find3 << endl;
```
程序输出如下
``` cpp
find is 3
find is 5
find is 0
```
## 总结
本文实现了用二分查找的方式找到给定的target值
源码链接[https://gitee.com/secondtonone1/algorithms](https://gitee.com/secondtonone1/algorithms)
更多算法实现点击下方链接
[https://llfc.club/category?catid=22zkaaQaS2wMZRygv3qQVpRKpXI](https://llfc.club/category?catid=22zkaaQaS2wMZRygv3qQVpRKpXI)
