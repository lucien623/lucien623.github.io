---
title: 重拾 DSA 排序算法之堆排序
date: 2019-10-21 16:23:47
tags: 
  - DSA
categories:
  - 技术
description: 关于堆排序的简单介绍和 Java 实现。
---

Heapsort（堆排序），顾名思义是基于堆数据结构的一种排序算法，了解堆之前我们需要了解什么是二叉树和完全二叉树，二叉树在之前关于树的文章中有简单介绍（[click here](https://lucien623.github.io/2017/08/19/%E9%87%8D%E6%8B%BE-DSA-%E4%B9%8B%E6%A0%91/)），简单来说就是每个节点最多含有两个子树的树称为二叉树，完全二叉树则是深度为h，除第 h 层外，其它各层 (1～h-1) 的结点数都达到最大个数，第 h 层所有的结点都连续集中在最左边的二叉树，那么堆除了有何完全二叉树同样的性质外，还有一个特殊的性质，即子结点的键值或索引总是小于（或者大于）等于它的父节点。堆一般分为两种，大顶堆和小顶堆。大顶堆指堆中的每个父节点都大于等于其孩子节点，根节点即最大元素，小顶堆则和大顶堆相反，每个父节点小于等于其孩子节点，根节点即最小元素。当我们以数组的形式保存堆时，即有以下性质 array[i] >= array[2 \* i + 1] && array[i] >= array[2 \* i + 1] 或者 array[i] <= array[2 \* i + 1] && array[i] <= array[2 \* i + 1]，这里假设 2 \* i + 1 和 2 \* i + 1 的元素都存在，有了这样的性质我们就可以来构建大顶堆或者小顶堆了，这里我们以构建大顶堆为例，大致思路是先从最后一个非叶子节点开始调整堆直到调整到根节点位置，那么最后一个非叶子节点如何得到呢，其实只要用数组的 length / 2 - 1 就可以计算出下标了，来看看代码。
```java
/**
 * 
 * @param array 待调整数组
 * @param i 需要调整的元素
 * @param length 待调整数组长度
 */
private static void adjustHeap(int array[], int i, int length) {
	for(int j = 2 * i + 1; j < length; j = 2 * j + 1) {
		int temp = array[i];
		if(j + 1 < length && array[j + 1] > array[j])
			j = j + 1;
		if(temp < array[j]) {
			swap(array, i, j);
			i = j;
		} else {
			break;
		}
	}
}

private static void swap(int array[], int i, int j) {
	if(i < 0 || j < 0 || i >= array.length || j >= array.length)
		return;
	int temp = array[i];
	array[i] = array[j];
	array[j] = temp;
}
```
然后我们从最后一个非叶子节点开始调整
```java
int length = array.length;
for(int i = length / 2 - 1; i >= 0; i--) {
	adjustHeap(array, i, length);
}
```
这样大顶堆就构建完成了。ok，堆排序的思路是每次构建完大顶堆的时候，把根节点和待排序数组的最后一个数进行交换，然后再根据根节点重新调整成大顶堆，直到排序完成。
```java
for(int last = length - 1;last > 0; last--) {
	swap(array, 0, last);
	adjustHeap(array, 0, last);
}
```
堆排序的大致介绍就是如此了，其最好最坏的时间复杂度均为 O(nlogn)。