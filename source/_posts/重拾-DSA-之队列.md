---
title: 重拾 DSA 之队列
date: 2017-08-12 16:40:19
tags: 
  - DSA
categories:
  - 技术
description: 本文主要介绍了队列的基本知识。
---
下面是 wikipedia 对于队列的定义：

> **队列**又称为伫列（queue），是先进先出（FIFO, First In First Out）的线性表。在具体应用中通常用链表或者数组来实现。队列只允许在后端（称为rear）进行插入操作，在前端（称为front）进行删除操作。

队列和栈十分相似，不同点在于栈是 Last In First Out，而队列是 First In First Out。

这里使用 LinkedList 存储数据。

- 添加一个元素

```java
private void push(T t) {
	mList.addLast(t);
}
```

- 移除元素

```java
private void pop() {
	if(!mList.isEmpty()) {
		mList.removeFirst();
	}
}
```

- 获取队列元素

```java
private T getHead() {
	return mList.getFirst();
}
```

[Source Code](https://github.com/lucien623/DSA_Review/blob/master/Queue.java)


