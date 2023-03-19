---
title: 重拾 DSA 排序算法之冒泡排序
date: 2018-05-23 18:36:06
tags: 
  - DSA
categories:
  - 技术
description: 关于冒泡排序的简单介绍和 Java 实现。
---
下面是 wikipedia 对于冒泡排序的概括：

> Bubble sort, sometimes referred to as sinking sort, is a simple sorting algorithm that repeatedly steps through the list to be sorted, compares each pair of adjacent items and swaps them if they are in the wrong order. The pass through the list is repeated until no swaps are needed, which indicates that the list is sorted. The algorithm, which is a comparison sort, is named for the way smaller or larger elements "bubble" to the top of the list. Although the algorithm is simple, it is too slow and impractical for most problems even when compared to insertion sort.[2] Bubble sort can be practical if the input is in mostly sorted order with some out-of-order elements nearly in position.

简而言之呢就是相领元素的比较，下面还是来自 wikipedia 的一张图，很直观了：
![](https://upload.wikimedia.org/wikipedia/commons/c/c8/Bubble-sort-example-300px.gif) 

不妨假设我们有这样一个数组`int list[] = {5, 1, 4, 2, 8};`
那么冒泡排序的步骤就如下:

第一趟：
{**5**, **1**, 4, 2, 8} -> {**1**, **5**, 4, 2, 8}, 5 > 1, 交换位置;
{1, **5**, **4**, 2, 8} -> {1, **4**, **5**, 2, 8}, 5 > 4, 交换位置;
{1, 4, **5**, **2**, 8} -> {1, 4, **2**, **5**, 8}, 5 > 2, 交换位置;
{1, 4, 2, **5**, **8**} -> {1, 4, 2, **5**, **8**}, 5 < 8, 位置不变;
所以第一趟排序下来的数组顺序是 {1, 4, 2, 5, 8}，此时数组中最大的数 **8** 已经在末尾。

第二趟：
{**1**, **4**, 2, 5, 8} -> {**1**, **4**, 2, 5, 8}, 1 < 4, 位置不变;
{1, **4**, **2**, 5, 8} -> {1, **2**, **4**, 5, 8}, 4 > 2, 交换位置;
{1, 2, **4**, **5**, 8} -> {1, 2, **4**, **5**, 8}, 4 < 5, 位置不变;
第二趟排序下来数组中第二大的数已经在倒数第二的位置。

第三趟：
{**1**, **2**, 4, 5, 8} -> {**1**, **2**, 4, 5, 8}, 1 < 2, 位置不变;
{1, **2**, **4**, 5, 8} -> {1, **2**, **4**, 5, 8}, 2 < 4, 位置不变;

第四趟：
{**1**, **2**, 4, 5, 8} -> {**1**, **2**, 4, 5, 8}, 1 < 2, 位置不变;

以下是代码实现：
```java
private static int[] bubbleSort(int[] list) {
		int length = list.length;
		for(int i = 0; i < length - 1; i++) {
			for(int j = 0; j < length - i - 1; j++) {
				if(list[j] > list[j + 1]) {
					int temp = list[j];
					list[j] = list[j + 1];
					list[j + 1] = temp;
				}
			}
		}
		
		return list;
	}
```
冒泡排序的效率并不高，时间复杂度为 O(n^2)；
