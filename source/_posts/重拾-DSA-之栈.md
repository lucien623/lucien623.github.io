---
title: 重拾 DSA 之栈
date: 2017-08-12 16:15:03
tags: 
  - DSA
categories:
  - 技术
description: 本文主要介绍了栈的基本知识。
---
下面是 wikipedia 对于栈的定义：

> **堆栈** （英语：stack），也可直接称栈（港澳台作堆叠），在计算机科学中，是一种特殊的串列形式的数据结构，它的特殊之处在于只能允许在链接串列或阵列的一端（称为堆叠顶端指标，英语：top）进行加入数据（英语：push）和输出数据（英语：pop）的运算。另外栈也可以用一维数组或连结串列的形式来完成。
由于堆叠数据结构只允许在一端进行操作，因而按照后进先出（LIFO, Last In First Out）的原理运作。

简单示意图：
![](https://upload.wikimedia.org/wikipedia/commons/thumb/2/29/Data_stack.svg/391px-Data_stack.svg.png)

这里使用 LinkedList 存储数据。

- 入栈

```java
public void push(T t) {
	mList.addFirst(t);
}
```

- 出栈

```java
public void pop() {
	if(!mList.isEmpty()) {
		mList.removeFirst();
	}
}
```

- 获取栈顶元素

```java
public T getPop() {
	return mList.getFirst();
}
```

[Source Code](https://github.com/lucien623/DSA_Review/blob/master/Stack.java)



