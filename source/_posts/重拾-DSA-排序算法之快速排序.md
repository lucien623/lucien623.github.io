---
title: 重拾 DSA 排序算法之快速排序
date: 2018-05-29 16:23:15
tags: 
  - DSA
categories:
  - 技术
description: 关于快速排序的简单介绍和 Java 实现。
---
快速排序的主要运用的就是分治的思想。

> The steps are:

1.Pick an element, called a pivot, from the array.

2.Partitioning: reorder the array so that all elements with values less than the pivot come before the pivot, while all elements with values greater than the pivot come after it (equal values can go either way). After this partitioning, the pivot is in its final position. This is called the partition operation.

3.Recursively apply the above steps to the sub-array of elements with smaller values and separately to the sub-array of elements with greater values.

上面是 wiki 中定义的步骤，其实就是先找一个 pivot，一般我们设定数组的第一个或者最后一个元素为 pivot，把数组中小于 pivot 的元素移到左边，大于 pivot 的元素移到右边，然后以同样的方式再对左右两边的子数组操作，最后就会得到一个有序数组。

sample:
![](https://upload.wikimedia.org/wikipedia/commons/thumb/a/af/Quicksort-diagram.svg/200px-Quicksort-diagram.svg.png) 


以下是代码实现：
```java
private static void sort(int[] array, int low, int high) {
	int l = low;
	int h = high;
	if(l >= h)
		return;
	int pivot = array[high];
	while(l < h) {
		while(array[l] > pivot && l < h) {
			int temp = array[h - 1];
			array[h] = array[l];
			array[l] = temp;
			h--;
		}
		while(array[l] <= pivot && l < h) {
			l++;
		}
	}
	array[l] = pivot;
	sort(array, low, l - 1);
	sort(array, l + 1, high);
}
```
快速排序的平均复杂度为 O(nlogn),最差情况下为 O(n^2)。

