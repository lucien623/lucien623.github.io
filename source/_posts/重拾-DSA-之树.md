---
title: 重拾 DSA 之树
date: 2017-08-19 09:50:40
tags: 
  - DSA
categories:
  - 技术
---

下面是 wikipedia 对于树的定义：

> **树**（英语：tree）是一种抽象数据类型（ADT）或是实作这种抽象数据类型的数据结构，用来模拟具有树状结构性质的数据集合。它是由 n（n>0）个有限节点组成一个具有层次关系的集合。把它叫做“树”是因为它看起来像一棵倒挂的树，也就是说它是根朝上，而叶朝下的。它具有以下的特点：
> * 每个节点有零个或多个子节点；
> * 没有父节点的节点称为根节点；
> * 每一个非根节点有且只有一个父节点；
> * 除了根节点外，每个子节点可以分为多个不相交的子树；

### 术语
**节点的度**：一个节点含有的子树的个数称为该节点的度；
**树的度**：一棵树中最大的节点的度称为树的度；
**叶子结点**：度为零的节点；
**分支节点**：度不为零的节点；
**父节点**：若一个节点含有字节点，则这个节点称为该字节点的父节点；
**子节点**：一个节点含有的子树的根节点称为该节点的字节点；
**兄弟节点**：具有相同父节点的节点称为兄弟节点；
**节点的层次**：从根开始定义，根为第一层，根的子节点为第2层，以此类推；
**树的高度或深度**：树中节点的最大层次；
**堂兄弟节点**：父节点在同一层的节点互为堂兄弟；
**节点的祖先**：从根到该节点所经分支上的所有节点；
**子孙**：以某节点为根的子树中任一节点都称为该节点的子孙；
**森林**：由 m（m >= 0）棵互不相交的树的结合成为森林；
![](http://wx2.sinaimg.cn/large/9302210cly1fyghx5gzzej207t07j74a.jpg)
如上图 B 节点的度为 2，C 的度为 1，整棵树的高度为 2。这棵树的叶子结点有 D、G、F，剩下的是分支节点。A 是 B 的父节点，A 是 C 的父节点，B 和 C 是兄弟节点。A 节点的层次为 1， B 节点的层次为 2，D 节点的层次为 3，树的高度为 3， D、E、F互为堂兄弟。

### 树的种类
树的种类可以分为两种，分别是**无序树**和**有序树**。无序树树中任意节点的子节点之间没有顺序关系，无序树也可以称为自由树；有序树表示树中任意节点的子节点之间有顺序关系。

### 二叉树
二叉树是属于有序树中的一种，每个节点最多含有两个子树的树称为二叉树。

三种遍历方式（递归方式）：

先序遍历： 根节点-左节点-右节点
``` java
private static void preOrder(Node node) {
	if(node != null) {
		System.out.print(node.data + "-->");
		preOrder(node.left);
		preOrder(node.right);
	}
}
```

中序遍历： 左节点-根节点-右节点
``` java
private static void inOrder(Node node) {
	if(node != null) {
		inOrder(node.left);
		System.out.print(node.data + "-->");
		inOrder(node.right);
	}
}
```

后续遍历： 左节点-右节点-根节点
``` java
private static void postOrder(Node node) {
	if(node != null) {
		postOrder(node.left);
		postOrder(node.right);
		System.out.print(node.data + "-->");
	}
}
```

[Source Code](https://github.com/lucien623/DSA_Review/blob/master/BinaryTree.java)

