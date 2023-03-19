---
title: 重拾 DSA 排序算法之插入排序
date: 2018-06-09 15:50:47
tags: 
  - DSA
categories:
  - 技术
description: 关于插入排序的介绍和 Java 实现。
---
wikipedia:
> Insertion sort iterates, consuming one input element each repetition, and growing a sorted output list. At each iteration, insertion sort removes one element from the input data, finds the location it belongs within the sorted list, and inserts it there. It repeats until no input elements remain.

意思就是从输入的元素里每次取出一个插入到有序的列表中，直到取完所有的元素为止。
![](https://upload.wikimedia.org/wikipedia/commons/0/0f/Insertion-sort-example-300px.gif)

以下是代码实现：
```java
private static void insertionSort(int[] array) {
	for(int i = 0;  i < array.length - 1; i++) {
		for(int j = i + 1; j > 0; j--) {
			if(array[j - 1] <= array[j])
				break;
			int temp = array[j - 1];
			array[j - 1] = array[j];
			array[j] = temp;
		}
	}
}
```
算法的时间复杂度为 O(n^2)。