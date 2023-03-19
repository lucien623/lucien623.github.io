---
title: 重拾 DSA 排序算法之简单选择排序
date: 2018-05-31 17:18:26
tags: 
  - DSA
categories:
  - 技术
description: 关于简答选择排序的简单介绍和 Java 实现。
---
wikipedia:
> The algorithm divides the input list into two parts: the sublist of items already sorted, which is built up from left to right at the front (left) of the list, and the sublist of items remaining to be sorted that occupy the rest of the list. Initially, the sorted sublist is empty and the unsorted sublist is the entire input list. The algorithm proceeds by finding the smallest (or largest, depending on sorting order) element in the unsorted sublist, exchanging (swapping) it with the leftmost unsorted element (putting it in sorted order), and moving the sublist boundaries one element to the right.

概括就是将数列分为两部分，分别是排好序和未排好序部分，每次取未排序部分的最大值或者最小值，放到已排序部分的数列中。

![](https://upload.wikimedia.org/wikipedia/commons/9/94/Selection-Sort-Animation.gif)

以下是代码实现：
```java
private static void simpleSelectionSort(int[] array) {
	for(int i = 0; i < array.length - 1; i++) {
		for(int j = i + 1; j < array.length; j++) {
			if(array[i] > array[j]) {
				int temp = array[i];
				array[i] = array[j];
				array[j] = temp;
			}
		}
	}
}
```
该算法的时间复杂度为 O(n^2)。